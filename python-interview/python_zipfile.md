# zipfile / tarfile — подготовка к собеседованию

## Что это и зачем

Модули `zipfile` и `tarfile` входят в стандартную библиотеку Python и предназначены для работы с архивами:

- **`zipfile`** — работа с ZIP-архивами (`.zip`). ZIP хранит каждый файл сжатым отдельно (random access к любому файлу без распаковки всего архива).
- **`tarfile`** — работа с TAR-архивами (`.tar`), в том числе сжатыми (`.tar.gz`, `.tar.bz2`, `.tar.xz`). TAR — это «tape archive»: файлы складываются в один поток, а сжатие применяется ко всему потоку целиком.

Зачем нужны:
- Упаковка нескольких файлов в один контейнер (дистрибутивы, бэкапы, deploy-артефакты).
- Экономия места и трафика за счёт сжатия.
- В Python wheel (`.whl`), egg, `.docx`, `.jar`, `.apk` — это, по сути, ZIP-архивы.

Ключевое отличие:
- **ZIP** — есть центральный каталог (central directory) в конце файла → можно читать отдельные файлы, не распаковывая весь архив. Сжатие пофайловое.
- **TAR** — последовательный поток, не имеет индекса → чтобы достать один файл в конце, нужно прочитать весь поток. Зато сохраняет UNIX-метаданные (права, владельцы, симлинки, устройства).

## Ключевые концепции

- **Compression methods (ZIP):** `ZIP_STORED` (без сжатия), `ZIP_DEFLATED` (zlib/deflate, самый частый), `ZIP_BZIP2`, `ZIP_LZMA`.
- **Compression modes (TAR):** `w` (без сжатия), `w:gz` (gzip), `w:bz2` (bzip2), `w:xz` (lzma). Для потоковой записи — `w|gz` и т.п.
- **Central directory** в ZIP лежит в конце файла, поэтому ZIP можно «дописывать» и читать выборочно.
- **Path traversal (Zip Slip / Tar Slip)** — главная уязвимость: имя внутри архива вида `../../etc/passwd` или абсолютный путь `/etc/cron.d/x` при наивной распаковке записывает файл вне целевой директории.
- **Zip bomb** — маленький архив, распаковывающийся в гигабайты (например, вложенные/повторяющиеся данные с экстремальным коэффициентом сжатия) → DoS.
- **Метаданные:** `ZipInfo` (для ZIP) и `TarInfo` (для TAR) описывают элемент архива (имя, размер, дата, права).

## Основные функции/классы/методы

### zipfile: создание архива

```python
import zipfile
from pathlib import Path

# Создание архива со сжатием DEFLATE
with zipfile.ZipFile(
    "archive.zip",
    mode="w",                       # 'w' перезапись, 'a' дозапись, 'x' создать, ошибка если есть
    compression=zipfile.ZIP_DEFLATED,
    compresslevel=9,                # 0..9 для DEFLATE (Python 3.7+)
) as zf:
    # Добавить файл с диска (arcname — имя внутри архива)
    zf.write("report.txt", arcname="docs/report.txt")

    # Записать данные напрямую из памяти, без временного файла
    zf.writestr("generated/data.json", '{"ok": true}')

    # Рекурсивно добавить директорию
    root = Path("src")
    for path in root.rglob("*"):
        if path.is_file():
            zf.write(path, arcname=path.relative_to(root.parent))
```

### zipfile: чтение архива

```python
import zipfile

with zipfile.ZipFile("archive.zip", mode="r") as zf:
    # Список имён внутри архива
    print(zf.namelist())

    # Метаданные по каждому элементу
    for info in zf.infolist():
        print(info.filename, info.file_size, info.compress_size, info.date_time)

    # Прочитать содержимое одного файла (bytes) без распаковки на диск
    data = zf.read("docs/report.txt")

    # Открыть как файловый объект (потоковое чтение, не грузит всё в память)
    with zf.open("generated/data.json") as fp:
        for line in fp:
            ...

    # Проверка целостности (CRC). Возвращает имя первого битого файла или None
    bad = zf.testzip()

    # Защита паролем (только чтение; ZIP-шифрование слабое!)
    zf.setpassword(b"secret")
    zf.read("secret.txt")
```

### zipfile: извлечение

```python
with zipfile.ZipFile("archive.zip") as zf:
    # ВНИМАНИЕ: extractall/extract в Python 3.6.2+ санитизируют имена,
    # но историческое поведение и сторонние реализации могут быть уязвимы.
    zf.extractall(path="output_dir")
    zf.extract("docs/report.txt", path="output_dir")
```

### tarfile: создание и чтение

```python
import tarfile

# Создание сжатого tar.gz
with tarfile.open("archive.tar.gz", "w:gz", compresslevel=9) as tf:
    tf.add("src", arcname="src")          # рекурсивно добавляет директорию
    tf.add("config.yml", arcname="config.yml")

# Чтение
with tarfile.open("archive.tar.gz", "r:gz") as tf:
    for member in tf.getmembers():
        print(member.name, member.size, member.mode, member.isfile())

    # Прочитать файл в память
    fp = tf.extractfile("src/main.py")    # None для директорий/симлинков
    if fp is not None:
        content = fp.read()
```

### tarfile: безопасное извлечение (Python 3.12+)

```python
import tarfile

with tarfile.open("archive.tar.gz", "r:gz") as tf:
    # filter='data' — самый безопасный: блокирует абсолютные пути, '..',
    # ссылки наружу, устройства, опасные права (Python 3.12+, PEP 706).
    tf.extractall(path="output_dir", filter="data")
```

### ZipInfo / TarInfo — тонкая настройка

```python
import zipfile, time

info = zipfile.ZipInfo(filename="hello.txt", date_time=time.localtime()[:6])
info.compress_type = zipfile.ZIP_DEFLATED
info.external_attr = 0o644 << 16          # права UNIX в ZIP

with zipfile.ZipFile("a.zip", "w") as zf:
    zf.writestr(info, "Привет")
```

### Проверка, что файл — корректный архив

```python
import zipfile, tarfile

zipfile.is_zipfile("archive.zip")     # True/False (проверяет сигнатуру)
tarfile.is_tarfile("archive.tar.gz")  # True/False
```

## Частые вопросы на собеседовании

**Q1. В чём принципиальная разница между ZIP и TAR?**
ZIP сжимает каждый файл по отдельности и хранит центральный каталог в конце → можно читать произвольный файл, не распаковывая весь архив. TAR — последовательный поток без индекса; сжатие (gzip/bz2/xz) применяется ко всему потоку целиком, поэтому случайный доступ невозможен, но TAR лучше хранит UNIX-метаданные (права, симлинки, владельцев).

**Q2. Зачем нужен `with` при работе с архивами?**
Контекстный менеджер гарантирует закрытие архива и дозапись центрального каталога (для ZIP в режиме записи) даже при исключении. Без `close()` ZIP-файл может оказаться повреждённым (каталог не записан).

**Q3. Как добавить данные в архив без записи временного файла на диск?**
`ZipFile.writestr(name, data)` для ZIP. Для TAR — собрать `TarInfo`, выставить `size` и передать поток через `tf.addfile(info, io.BytesIO(data))`.

**Q4. Что такое Zip Slip / path traversal и как защититься?**
Если имя внутри архива содержит `../` или абсолютный путь, наивная распаковка пишет файл вне целевой папки (вплоть до перезаписи `~/.ssh/authorized_keys`). Защита: проверять, что нормализованный целевой путь лежит внутри директории назначения, либо использовать `tarfile.extractall(filter="data")` (3.12+).

**Q5. Безопасно ли `zip.extractall()` в современном Python?**
Для `zipfile` имена санитизируются (убираются ведущие `/` и `..`) ещё с 3.6.2, но не блокируются симлинки наружу. Для `tarfile` исторически НЕ было защиты — до 3.12 `extractall` уязвим; с 3.14 безопасный фильтр `data` стал значением по умолчанию. Всё равно лучше явно валидировать пути.

**Q6. Что такое Zip bomb и как защититься?**
Маленький архив, распаковывающийся в огромный объём данных (DoS). Защита: до извлечения проверять `info.file_size` и суммарный распакованный объём, ограничивать число файлов, читать потоково с лимитом байт, не доверять заявленному `file_size`.

**Q7. Какие методы сжатия поддерживает `zipfile` и какой выбрать?**
`ZIP_STORED` (без сжатия — для уже сжатых данных вроде jpg/mp4), `ZIP_DEFLATED` (баланс, по умолчанию для большинства задач), `ZIP_BZIP2` и `ZIP_LZMA` (сильнее жмут, но медленнее). LZMA даёт лучший коэффициент, DEFLATE — лучший баланс скорость/совместимость.

**Q8. Как прочитать один файл из большого ZIP, не распаковывая весь архив?**
`ZipFile.open(name)` или `ZipFile.read(name)`. ZIP имеет центральный каталог, поэтому доступ к одному файлу — O(1) по индексу, без чтения остального архива.

**Q9. Можно ли дописывать в существующий ZIP? А удалять из него?**
Дописывать — да, открыв в режиме `"a"`. Удаление файла из ZIP стандартная библиотека напрямую не поддерживает (нужно пересоздать архив без удаляемого элемента; в новых версиях появилось `ZipFile.remove()` в 3.13+).

**Q10. Насколько надёжно ZIP-шифрование паролем?**
Классическое ZipCrypto слабое и легко ломается (known-plaintext). `zipfile` умеет только читать такие архивы, но не создавать зашифрованные. Для реальной защиты — шифровать содержимое отдельно (например, AES через `cryptography`) или использовать архиватор с AES.

**Q11. Что делает `testzip()` и `getmember().chksum`?**
`testzip()` пересчитывает CRC32 каждого файла и возвращает имя первого повреждённого (или `None`). Полезно для проверки целостности до распаковки.

## Подводные камни (gotchas)

- **Path traversal — главная угроза.** Никогда не доверяйте именам файлов из архива. Всегда нормализуйте и проверяйте, что результат внутри целевой директории.
- **Симлинки в TAR.** TAR может содержать символические/жёсткие ссылки, указывающие наружу (`link -> /etc/passwd`). `filter="data"` их блокирует; самописная распаковка — нет.
- **Незакрытый ZIP в режиме записи повреждён** — центральный каталог пишется только при `close()`.
- **`extractfile()` возвращает `None`** для директорий и спец-файлов — всегда проверяйте на `None`.
- **Память.** `zf.read()` грузит весь файл в RAM. Для больших файлов используйте `zf.open()` и читайте чанками.
- **Кодировка имён.** Старые ZIP используют CP437; не-ASCII имена могут «поехать». Современные пишут UTF-8 (флаг в заголовке), но совместимость с архиваторами разная.
- **`compresslevel` для STORED игнорируется**; для bzip2 диапазон 1..9, для LZMA уровень не настраивается через этот параметр.
- **Zip bomb через вложенность.** Проверка `file_size` спасает не всегда — заявленный размер может лгать; ограничивайте фактически прочитанные байты.
- **Дата 1980.** ZIP не хранит даты раньше 1980 года — старые mtime обрежутся.

## Лучшие практики

- Всегда используйте `with` для гарантированного закрытия и записи каталога.
- При распаковке недоверенных архивов — **валидируйте каждый путь** (см. шпаргалку) и/или используйте `tarfile.extractall(filter="data")`.
- Ставьте лимиты: максимальный распакованный размер, число файлов, глубина, отказ при симлинках наружу.
- Для уже сжатых данных (медиа) выбирайте `ZIP_STORED` — DEFLATE только тратит CPU.
- Не полагайтесь на ZIP-шифрование паролем для безопасности — шифруйте данные отдельно.
- Для бэкапов с UNIX-правами/симлинками — `tarfile`; для кросс-платформенной раздачи файлов — `zipfile`.
- Проверяйте `is_zipfile`/`is_tarfile` перед обработкой пользовательских загрузок.
- Логируйте и обрабатывайте `BadZipFile`, `LargeZipFile`, `tarfile.TarError`.

## Шпаргалка

```python
import zipfile, tarfile, os

# --- Создание ZIP ---
with zipfile.ZipFile("a.zip", "w", zipfile.ZIP_DEFLATED, compresslevel=9) as z:
    z.write("file.txt", "name_in_zip.txt")
    z.writestr("mem.txt", "данные из памяти")

# --- Чтение ZIP ---
with zipfile.ZipFile("a.zip") as z:
    z.namelist()                 # имена
    z.infolist()                 # ZipInfo-объекты
    data = z.read("name.txt")    # bytes в память
    with z.open("name.txt") as f:  # потоково
        chunk = f.read(8192)

# --- Создание tar.gz ---
with tarfile.open("a.tar.gz", "w:gz") as t:
    t.add("dir", arcname="dir")

# --- Чтение tar ---
with tarfile.open("a.tar.gz", "r:gz") as t:
    members = t.getmembers()
    fp = t.extractfile("dir/f.py")   # None для не-файлов

# --- Безопасная распаковка ZIP (защита от Zip Slip) ---
def safe_extract_zip(zip_path, dest):
    dest = os.path.realpath(dest)
    with zipfile.ZipFile(zip_path) as z:
        for member in z.namelist():
            target = os.path.realpath(os.path.join(dest, member))
            if not (target == dest or target.startswith(dest + os.sep)):
                raise ValueError(f"Path traversal: {member}")
        z.extractall(dest)

# --- Безопасная распаковка TAR (Python 3.12+) ---
with tarfile.open("a.tar.gz", "r:gz") as t:
    t.extractall("out", filter="data")   # блокирует .., абс.пути, симлинки

# --- Проверка целостности / типа ---
zipfile.is_zipfile("a.zip")
tarfile.is_tarfile("a.tar.gz")
with zipfile.ZipFile("a.zip") as z:
    broken = z.testzip()         # None если всё ок

# Методы сжатия ZIP:
# ZIP_STORED | ZIP_DEFLATED | ZIP_BZIP2 | ZIP_LZMA
# Режимы TAR: 'w' 'w:gz' 'w:bz2' 'w:xz' (потоковые: 'w|gz')
```
