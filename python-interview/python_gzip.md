# gzip / bz2 / lzma / zlib — подготовка к собеседованию

## Что это и зачем

Это четыре модуля стандартной библиотеки для сжатия данных без потерь:

- **`zlib`** — низкоуровневая обёртка над библиотекой zlib (алгоритм DEFLATE = LZ77 + Huffman). Работает с «сырыми» данными в памяти. Это «движок» под капотом gzip и ZIP.
- **`gzip`** — формат `.gz`: тот же DEFLATE, но с заголовком/футером gzip (имя файла, mtime, CRC32, размер). Удобен для сжатия файлов и потоков, совместим с утилитой `gzip`.
- **`bz2`** — алгоритм bzip2 (BWT — Burrows–Wheeler Transform + Huffman). Жмёт сильнее gzip, но медленнее.
- **`lzma`** — формат `.xz` / raw LZMA (алгоритм LZMA/LZMA2). Лучший коэффициент сжатия из стандартных, но самый медленный и прожорливый по памяти.

Зачем: уменьшение размера данных для хранения и передачи (HTTP gzip, логи, бэкапы, сериализованные данные, сетевые протоколы).

Все четыре — **lossless** (без потерь): распаковка восстанавливает байт-в-байт.

## Ключевые концепции

- **DEFLATE** = LZ77 (поиск повторов) + Huffman-кодирование. Основа zlib, gzip, ZIP. Быстрый, хорошая совместимость.
- **gzip vs zlib vs raw deflate** — это один алгоритм DEFLATE, но разные обёртки/заголовки:
  - `zlib`-формат: 2-байтный заголовок + Adler-32 чексумма.
  - `gzip`-формат: gzip-заголовок (магия `1f 8b`, mtime, имя) + CRC32 + ISIZE.
  - raw deflate: без заголовка и чексуммы.
- **Уровень сжатия (compresslevel)** — компромисс «время CPU ↔ размер». Обычно 0/1 (быстро, слабо) … 9 (медленно, сильно). 0 — без сжатия.
- **Streaming (потоковое сжатие)** — обработка данных по кускам через объекты compressor/decompressor, без загрузки всего в память.
- **Чексуммы:** gzip/zlib хранят контрольную сумму (CRC32/Adler-32) → распаковка детектит повреждение.
- **Память vs скорость:** lzma жмёт лучше всех, но требует больше RAM и CPU; gzip — золотая середина; lz4/zstd (не в stdlib) быстрее, но это сторонние пакеты.

## Основные функции/классы/методы

### zlib — сжатие в памяти

```python
import zlib

raw = b"a" * 1000 + b"some data" * 50

# One-shot сжатие/распаковка (zlib-формат: заголовок + Adler-32)
packed = zlib.compress(raw, level=6)     # level 0..9, -1 = по умолчанию (6)
back = zlib.decompress(packed)
assert back == raw

# Контрольные суммы (для проверки целостности, НЕ для безопасности)
print(zlib.crc32(raw))      # быстрый CRC32
print(zlib.adler32(raw))    # ещё быстрее, но слабее

# Потоковое (инкрементальное) сжатие
co = zlib.compressobj(level=9, wbits=zlib.MAX_WBITS)
chunks = [co.compress(b"part1"), co.compress(b"part2"), co.flush()]
compressed = b"".join(chunks)

# wbits управляет форматом:
#   15  (MAX_WBITS)      -> zlib-формат
#   -15                  -> raw deflate (без заголовка)
#   31 (15 + 16)         -> gzip-формат
do = zlib.decompressobj(wbits=zlib.MAX_WBITS)
original = do.decompress(compressed) + do.flush()
```

### gzip — файлы и потоки

```python
import gzip
import shutil

# Запись в .gz файл
with gzip.open("data.txt.gz", "wt", encoding="utf-8", compresslevel=9) as f:
    f.write("строка логов\n")

# Чтение .gz файла
with gzip.open("data.txt.gz", "rt", encoding="utf-8") as f:
    text = f.read()

# Бинарный режим
with gzip.open("blob.gz", "wb") as f:
    f.write(b"\x00\x01\x02")

# One-shot для байтов (удобно для сети/кэша)
packed = gzip.compress(b"payload", compresslevel=6)
back = gzip.decompress(packed)

# Сжать существующий файл потоково (не грузя в память)
with open("big.log", "rb") as src, gzip.open("big.log.gz", "wb") as dst:
    shutil.copyfileobj(src, dst, length=1024 * 1024)
```

### bz2 — сильнее, медленнее

```python
import bz2

# One-shot
packed = bz2.compress(b"data" * 1000, compresslevel=9)  # 1..9
back = bz2.decompress(packed)

# Файлы
with bz2.open("data.bz2", "wt", encoding="utf-8") as f:
    f.write("текст")

# Потоковое
c = bz2.BZ2Compressor(compresslevel=9)
out = c.compress(b"chunk1") + c.compress(b"chunk2") + c.flush()
d = bz2.BZ2Decompressor()
src = d.decompress(out)
```

### lzma — максимальное сжатие

```python
import lzma

# One-shot, формат .xz по умолчанию
packed = lzma.compress(b"data" * 10000, preset=9)  # preset 0..9, | lzma.PRESET_EXTREME
back = lzma.decompress(packed)

# Файлы
with lzma.open("data.xz", "wt", encoding="utf-8", preset=6) as f:
    f.write("текст")

# Форматы: FORMAT_XZ (по умолчанию), FORMAT_ALONE (.lzma legacy), FORMAT_RAW
packed_raw = lzma.compress(
    b"data",
    format=lzma.FORMAT_RAW,
    filters=[{"id": lzma.FILTER_LZMA2, "preset": 6}],
)

# Потоковое
c = lzma.LZMACompressor(preset=9 | lzma.PRESET_EXTREME)
out = c.compress(b"a" * 100000) + c.flush()
```

### Единый интерфейс через одинаковый API

```python
# Все четыре модуля дают похожий high-level API:
# <mod>.compress(data) / <mod>.decompress(data)
# <mod>.open(path, mode) -> файловый объект
# Это позволяет писать generic-код, выбирая модуль по расширению.

import gzip, bz2, lzma

OPENERS = {".gz": gzip.open, ".bz2": bz2.open, ".xz": lzma.open}

def smart_open(path, mode="rt", **kw):
    import os
    opener = OPENERS.get(os.path.splitext(path)[1], open)
    return opener(path, mode, **kw)
```

## Частые вопросы на собеседовании

**Q1. В чём разница между `gzip`, `zlib` и raw deflate?**
Это один и тот же алгоритм DEFLATE, отличаются только обёрткой: `zlib`-формат добавляет 2-байтный заголовок и Adler-32; `gzip` — gzip-заголовок (магия, mtime, имя файла) и CRC32 + размер; raw deflate — без заголовка и чексуммы. В `zlib` формат выбирается параметром `wbits`.

**Q2. Какой алгоритм жмёт сильнее, какой быстрее?**
По коэффициенту сжатия (сильнее → слабее): обычно `lzma` (xz) > `bz2` > `gzip`/`zlib`. По скорости (быстрее → медленнее): `gzip`/`zlib` > `bz2` > `lzma`. lzma даёт лучший размер ценой CPU и памяти; gzip — лучший баланс и максимальная совместимость.

**Q3. Что означает уровень сжатия и какой выбрать?**
Уровень (1..9 для gzip/bz2, preset 0..9 для lzma) регулирует компромисс CPU/размер. Для gzip разница между 6 и 9 по размеру часто мала, а по времени — велика; 6 — разумный дефолт. Для горячего пути (сеть) берут 1–3, для архивов — 9.

**Q4. Когда сжатие бесполезно или вредно?**
Для уже сжатых/высокоэнтропийных данных (JPEG, MP4, PNG, зашифрованные данные, случайные байты). Повторное сжатие почти не уменьшает размер, а иногда увеличивает (добавляется заголовок) и зря тратит CPU.

**Q5. Как сжимать поток, не загружая всё в память?**
Использовать инкрементальные объекты: `zlib.compressobj()`/`decompressobj()`, `bz2.BZ2Compressor`/`BZ2Decompressor`, `lzma.LZMACompressor`/`LZMADecompressor`, либо `gzip.open` + `shutil.copyfileobj` с заданным `length`. Это позволяет обрабатывать файлы больше RAM.

**Q6. Зачем `flush()` у компрессоров?**
Компрессор буферизует данные ради лучшего сжатия. `flush()` досжимает остаток и завершает поток (для gzip/lzma пишет футер с чексуммой/размером). Без финального `flush()` выходные данные будут неполными и не распакуются.

**Q7. CRC32/Adler-32 — это для безопасности?**
Нет. Это контрольные суммы для обнаружения случайных повреждений (битый диск, обрыв передачи), а не криптографические хеши. Их легко подделать. Для защиты целостности от злоумышленника — HMAC/SHA-256 (модуль `hmac`/`hashlib`).

**Q8. Что такое decompression bomb и как защититься?**
Маленький сжатый блок, разворачивающийся в гигабайты (DoS). Защита: распаковывать потоково с лимитом — читать `decompress(data, max_length)` или ограничивать общий объём прочитанных байт, прерывая при превышении порога. Не доверять заявленному размеру.

**Q9. Можно ли распаковать gzip-данные через zlib?**
Да: `zlib.decompressobj(wbits=31)` или `wbits=zlib.MAX_WBITS | 16` понимает gzip-заголовок. Также `wbits=47` (32 + 15) включает авто-детект zlib/gzip.

**Q10. Чем `lzma` отличается по форматам?**
`FORMAT_XZ` — современный `.xz` с целостностью и метаданными (дефолт). `FORMAT_ALONE` — старый `.lzma`. `FORMAT_RAW` — голый поток без заголовка (нужно вручную задавать `filters` и при распаковке — те же). RAW экономит несколько байт, но требует знания параметров заранее.

**Q11. Почему bzip2 редко используют сейчас?**
Он медленнее gzip и при этом часто проигрывает lzma/zstd по сжатию. Ниша узкая. Современная альтернатива (вне stdlib) — `zstd` (быстрый, отличное сжатие) и `lz4` (сверхбыстрый).

## Подводные камни (gotchas)

- **Забытый `flush()`** → обрезанный поток, который не распакуется. Всегда финализируйте компрессор.
- **Decompression/zip bomb** → ограничивайте размер при распаковке (`max_length`), иначе OOM/DoS.
- **Чексуммы ≠ безопасность.** CRC32/Adler-32 не защищают от целенаправленной подмены.
- **Текстовый vs бинарный режим.** `gzip.open(..., "rt")` требует `encoding`; в `"rb"` вернёт байты. Путаница режимов → `TypeError`/кракозябры.
- **Двойное сжатие** уже сжатых данных бесполезно и тратит CPU.
- **gzip хранит mtime и имя файла** в заголовке → одинаковые данные дают разные байты в разное время; для воспроизводимых сборок задавайте `mtime=0` (`gzip.GzipFile(..., mtime=0)`).
- **Память lzma.** preset 9 / `PRESET_EXTREME` могут требовать сотни МБ RAM на словарь — опасно при многопоточности/множестве параллельных задач.
- **`wbits` легко перепутать** — несовпадение формата при сжатии и распаковке даёт `zlib.error: incorrect header check`.
- **`decompressobj.unused_data`** содержит «хвост» после конца потока (например, при склеенных архивах) — полезно при multi-stream.

## Лучшие практики

- Выбирайте алгоритм под задачу: gzip — дефолт и совместимость; lzma — когда важен размер и не жалко CPU; bz2 — редко; для скорости вне stdlib — zstd/lz4.
- Для больших данных — потоковая обработка (`copyfileobj`, инкрементальные объекты), не `compress(file.read())`.
- При распаковке недоверенных данных всегда ставьте лимит на распакованный объём.
- Не сжимайте уже сжатое; проверяйте энтропию/тип данных.
- Для воспроизводимых артефактов фиксируйте `mtime=0` и не пишите имя файла в gzip-заголовок.
- Для целостности от злоумышленника используйте `hmac`, а не CRC.
- Подбирайте уровень бенчмарком на ваших данных — «9» не всегда стоит затраченного времени.

## Шпаргалка

```python
# === One-shot (в памяти) ===
import zlib, gzip, bz2, lzma
zlib.compress(b"x", 6);      zlib.decompress(p)
gzip.compress(b"x", 6);      gzip.decompress(p)
bz2.compress(b"x", 9);       bz2.decompress(p)
lzma.compress(b"x", preset=6); lzma.decompress(p)

# === Файлы (text/binary) ===
gzip.open("f.gz", "wt", encoding="utf-8", compresslevel=9)
bz2.open("f.bz2", "rb")
lzma.open("f.xz", "wt", preset=9 | lzma.PRESET_EXTREME)

# === Потоковое сжатие ===
co = zlib.compressobj(9); data = co.compress(b"...") + co.flush()
bc = bz2.BZ2Compressor(9)
lc = lzma.LZMACompressor(preset=6)

# === Потоковая распаковка с лимитом (анти-бомба) ===
d = zlib.decompressobj()
out = d.decompress(chunk, 10 * 1024 * 1024)   # max_length

# === Распаковать gzip через zlib ===
zlib.decompressobj(wbits=31).decompress(gz_bytes)    # gzip
zlib.decompressobj(wbits=-15)                        # raw deflate
zlib.decompressobj(wbits=47)                          # авто gzip/zlib

# === Сжать большой файл без RAM ===
import shutil
with open("big", "rb") as s, gzip.open("big.gz", "wb") as d:
    shutil.copyfileobj(s, d, 1 << 20)

# === Воспроизводимый gzip (без mtime/имени) ===
with gzip.GzipFile("r.gz", "wb", mtime=0) as f:
    f.write(b"data")

# Сжатие (сильнее→слабее):  lzma > bz2 > gzip/zlib
# Скорость (быстрее→медленнее): gzip/zlib > bz2 > lzma
```
