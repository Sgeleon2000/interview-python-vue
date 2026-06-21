# base64 / binascii — подготовка к собеседованию

## Что это и зачем

`base64` и `binascii` — модули стандартной библиотеки для **преобразования бинарных данных в текстовое (ASCII) представление и обратно**.

- `base64` — кодирование в Base64/Base32/Base16 (и их URL-safe/специальные варианты). Используется, когда бинарные данные нужно пропустить через канал, рассчитанный на текст (email/MIME, JSON, URL, HTTP-заголовки, data URI, JWT).
- `binascii` — низкоуровневые преобразования между бинарными и ASCII-форматами (hex, base64, CRC, контрольные суммы). На нём частично построен `base64`.

Ключевая идея: **это НЕ шифрование и НЕ хеширование.** Base64 — обратимое кодирование без ключа, не даёт никакой защиты. Любой может декодировать. Цель — безопасная транспортировка бинарных данных в текстовых протоколах.

Base64 увеличивает размер примерно на **33%** (4 символа на каждые 3 байта).

## Ключевые концепции

- **Base64** использует алфавит из 64 символов: `A–Z`, `a–z`, `0–9`, `+`, `/` и `=` как padding (выравнивание до кратности 4). Каждый символ кодирует 6 бит; 3 байта (24 бита) → 4 символа.
- **URL-safe Base64** заменяет проблемные в URL символы: `+` → `-`, `/` → `_`. Padding `=` тоже часто проблемен в URL (его убирают).
- **Base32** — алфавит из 32 символов (`A–Z`, `2–7`), регистронезависимый, длиннее, но устойчив к ошибкам и подходит для человекочитаемых кодов (например, TOTP-секреты, ключи).
- **Base16 (hex)** — обычное шестнадцатеричное представление; 1 байт → 2 символа.
- **Вход — всегда `bytes`, выход тоже `bytes`.** `b64encode("строка")` — ошибка; нужно `"строка".encode()`.
- **`binascii`** даёт `hexlify`/`unhexlify` (hex), `b2a_base64`/`a2b_base64` (база `base64` модуля), CRC-функции.

## Основные функции/классы/методы

### Base64: кодирование и декодирование

```python
import base64

data = "Привет, мир!".encode("utf-8")        # bytes (важно!)

encoded = base64.b64encode(data)             # bytes: b'0J/RgNC4...=='
print(encoded.decode("ascii"))               # строка для передачи

decoded = base64.b64decode(encoded)          # обратно в bytes
print(decoded.decode("utf-8"))               # 'Привет, мир!'
```

### URL-safe Base64

```python
import base64

raw = b"\xfb\xff\xfe data"
std = base64.b64encode(raw)            # может содержать '+' и '/'
url = base64.urlsafe_b64encode(raw)    # '+' -> '-', '/' -> '_'
print(std)   # b'+//+IGRhdGE='
print(url)   # b'-__-IGRhdGE='

# Декодирование
base64.urlsafe_b64decode(url)          # == raw
```

URL-safe вариант нужен для токенов в URL, JWT, query-параметров.

### Base64 без padding (частый кейс для JWT/токенов)

```python
import base64

def b64url_nopad(data: bytes) -> str:
    # Убираем '=' — компактнее и URL-friendly
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode("ascii")

def b64url_decode(s: str) -> bytes:
    # Восстанавливаем padding: длина должна быть кратна 4
    pad = -len(s) % 4
    return base64.urlsafe_b64decode(s + "=" * pad)

token = b64url_nopad(b"payload")
print(token)                  # 'cGF5bG9hZA'
print(b64url_decode(token))   # b'payload'
```

### Base32 и Base16

```python
import base64

data = b"secret"

# Base32 — регистронезависимый, алфавит A-Z2-7
b32 = base64.b32encode(data)         # b'ONSWG4TFOQ======'
base64.b32decode(b32)                # b'secret'
base64.b32decode("onswg4tfoq======", casefold=True)  # игнор регистра

# Base16 — hex в верхнем регистре
b16 = base64.b16encode(data)         # b'736563726574'
base64.b16decode(b16)                # b'secret'
base64.b16decode("736563726574", casefold=True)  # допустить нижний регистр
```

### `binascii` — hexlify / unhexlify

```python
import binascii

data = b"\x00\xff\x10"
hexed = binascii.hexlify(data)            # b'00ff10'
binascii.unhexlify(hexed)                 # b'\x00\xff\x10'

# Разделитель (Python 3.8+)
binascii.hexlify(data, "-")               # b'00-ff-10'

# Эквивалент через bytes-методы (часто проще):
data.hex()                                 # '00ff10'
data.hex("-")                              # '00-ff-10'
bytes.fromhex("00ff10")                    # b'\x00\xff\x10'
```

### `binascii` — base64 и CRC

```python
import binascii

# Низкоуровневый base64 (b2a_base64 добавляет '\n' в конце!)
binascii.b2a_base64(b"hi")                 # b'aGk=\n'
binascii.b2a_base64(b"hi", newline=False)  # b'aGk='
binascii.a2b_base64(b"aGk=")               # b'hi'

# Контрольные суммы
binascii.crc32(b"data")                    # int CRC-32
binascii.crc_hqx(b"data", 0)               # CRC-16
```

### Практика: data URI для картинки

```python
import base64

with open("logo.png", "rb") as f:
    img = f.read()
b64 = base64.b64encode(img).decode("ascii")
data_uri = f"data:image/png;base64,{b64}"
# <img src="data:image/png;base64,iVBORw0KGgo...">
```

### Практика: Basic Auth заголовок

```python
import base64

user, password = "admin", "secret"
token = base64.b64encode(f"{user}:{password}".encode()).decode("ascii")
header = f"Basic {token}"     # 'Basic YWRtaW46c2VjcmV0'
# ВНИМАНИЕ: это лишь кодирование, не защита! Использовать только поверх HTTPS.
```

## Частые вопросы на собеседовании

**Q: Base64 — это шифрование?**
A: Нет. Это обратимое кодирование без ключа. Любой может декодировать. Не даёт никакой конфиденциальности. Для защиты нужно шифрование (например, `cryptography`).

**Q: Зачем вообще нужен Base64, если он не защищает?**
A: Чтобы безопасно передать бинарные данные через текстовые каналы: email (MIME), JSON, URL, HTTP-заголовки, XML, data URI. Текстовые протоколы могут портить байты вроде `\x00`, перевода строки, не-ASCII — Base64 переводит всё в безопасный набор ASCII.

**Q: На сколько Base64 увеличивает размер?**
A: Примерно на 33% (на каждые 3 байта — 4 ASCII-символа), плюс padding. Поэтому для больших объёмов это накладно.

**Q: В чём разница между обычным и URL-safe Base64?**
A: URL-safe заменяет `+` на `-` и `/` на `_`, потому что `+` и `/` имеют спецзначение в URL. Часто также убирают padding `=`.

**Q: Почему `b64encode("text")` падает с ошибкой?**
A: Функции работают с `bytes`, а не `str`. Нужно `"text".encode()`. И возвращают тоже `bytes` — для строки делайте `.decode("ascii")`.

**Q: Когда выбрать Base32 вместо Base64?**
A: Когда важна регистронезависимость и устойчивость к ошибкам ввода человеком — TOTP/2FA-секреты, лицензионные ключи, коды для голосовой/ручной передачи. Base32 длиннее, но без неоднозначных символов.

**Q: Зачем нужен padding `=` и можно ли без него?**
A: `=` выравнивает длину до кратной 4 символам, чтобы декодер понял границы. Можно передавать без него, если декодер сам восстановит padding (`len % 4`). В JWT и многих токенах padding убирают.

**Q: `data.hex()` или `binascii.hexlify(data)` — что использовать?**
A: Для простого hex — `bytes.hex()` / `bytes.fromhex()`, это нагляднее и не требует импорта. `binascii.hexlify` нужен реже; даёт `bytes` и работал ещё до появления `.hex()`.

**Q: Почему `binascii.b2a_base64` добавляет перевод строки?**
A: Исторически — для построчного MIME-формата. Если не нужен `\n`, передавайте `newline=False` или используйте `base64.b64encode`.

**Q: Безопасен ли Basic Auth с Base64?**
A: Сам по себе нет — это всего лишь кодирование, тривиально декодируется. Безопасность даёт только HTTPS поверх. Без TLS креды передаются практически открыто.

**Q: Что делать с не-ASCII в исходных данных?**
A: Base64 кодирует именно байты. Сначала переведите строку в байты в нужной кодировке (обычно UTF-8): `text.encode("utf-8")`, потом кодируйте.

## Подводные камни (gotchas)

- **`str` vs `bytes`.** Вход/выход функций — `bytes`. Частая ошибка — передать строку.
- **Не защита.** Использование Base64 «для безопасности» — классическая ошибка на ревью.
- **Padding ломает URL.** `=` в query может требовать дополнительного percent-encoding; для URL чаще убирают padding.
- **`urlsafe_b64decode` требует корректный alphabet.** Нельзя декодировать стандартным `b64decode` строку, закодированную urlsafe (символы `-`/`_` приведут к ошибке/мусору).
- **`binascii.b2a_base64` добавляет `\n`** — неожиданно при сравнении строк.
- **Невалидный ввод.** `b64decode` по умолчанию мягко относится к лишним символам; `validate=True` заставит бросать `binascii.Error` на мусор: `base64.b64decode(s, validate=True)`.
- **Восстановление padding** при декодировании беспэддинговых токенов: длина должна быть кратна 4, добавьте `=`.
- **Размер растёт на 33%** — не кодируйте большие файлы без нужды (например, в JSON).
- **Base32 регистр.** При декодировании нижнего регистра нужен `casefold=True`.

## Лучшие практики

- Всегда кодируйте сначала в `bytes` (`.encode("utf-8")`), декодируйте обратно явно.
- Для URL/JWT/токенов — `urlsafe_b64encode` и обычно без padding (с восстановлением при декодировании).
- Для строгой проверки входных данных — `b64decode(s, validate=True)`.
- Никогда не выдавайте Base64 за защиту; для секретов — настоящее шифрование/хеширование.
- Для простого hex предпочитайте `bytes.hex()` / `bytes.fromhex()`.
- Помните про оверхед 33% — не кодируйте крупные бинарники в текст без необходимости (лучше отдельный бинарный канал/вложение).
- Base32 — для человекочитаемых, регистронезависимых кодов.

## Шпаргалка

```python
import base64, binascii

data = "текст".encode("utf-8")     # сначала -> bytes

# Base64
base64.b64encode(data)             # -> bytes
base64.b64decode(s)                # -> bytes
base64.b64decode(s, validate=True) # строгая проверка

# URL-safe
base64.urlsafe_b64encode(data)     # '+'->'-', '/'->'_'
base64.urlsafe_b64decode(s)

# Без padding (JWT-стиль)
enc = base64.urlsafe_b64encode(data).rstrip(b"=")
dec = base64.urlsafe_b64decode(s + "=" * (-len(s) % 4))

# Base32 / Base16
base64.b32encode(data); base64.b32decode(s, casefold=True)
base64.b16encode(data); base64.b16decode(s, casefold=True)

# Hex (предпочтительно через bytes-методы)
data.hex()                         # 'd182...'
data.hex("-")                      # с разделителем
bytes.fromhex("d182")
binascii.hexlify(data); binascii.unhexlify(s)

# CRC
binascii.crc32(data)               # CRC-32 (int)

# Памятка: Base64 != шифрование; рост размера ~33%; вход/выход = bytes
```
