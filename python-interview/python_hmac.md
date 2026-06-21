# hmac — подготовка к собеседованию

## Что это и зачем

`hmac` — модуль стандартной библиотеки для вычисления HMAC (Hash-based Message Authentication Code, RFC 2104). HMAC — это код аутентификации сообщения на основе криптографической хеш-функции и **секретного ключа**.

HMAC одновременно гарантирует:
- **целостность** — сообщение не изменили в пути;
- **аутентичность** — сообщение создал тот, кто знает секретный ключ.

Зачем нужно:
- проверка подлинности webhook'ов (GitHub, Stripe, Slack подписывают payload через HMAC-SHA256);
- подпись токенов/сессий/cookie (часть механизма JWT — HS256 это HMAC-SHA256);
- защита API-запросов от подделки;
- проверка, что данные пришли от доверенной стороны с общим секретом.

Ключевое отличие от просто хеша: обычный SHA-256 любой может пересчитать. HMAC может пересчитать только владелец секретного ключа → злоумышленник без ключа не подделает корректную подпись.

## Ключевые концепции

- **HMAC = H(key ⊕ opad ‖ H(key ⊕ ipad ‖ message))** — двойное хеширование с ключом и двумя паддингами. Эта конструкция защищает от length-extension атаки, которой подвержен «голый» `H(secret + msg)`.
- **Симметричный секрет** — обе стороны (отправитель и проверяющий) знают один общий ключ. Это не подпись с открытым ключом (для этого — RSA/ECDSA).
- **MAC vs цифровая подпись:** MAC симметричен (общий секрет, нельзя доказать третьей стороне, кто автор); подпись асимметрична (приватный/публичный ключ, неотрекаемость).
- **`compare_digest`** — сравнение в постоянном времени, защита от timing-атак при проверке подписей.
- **Length-extension атака** — для конструкции `hash(secret ‖ message)` на MD5/SHA-1/SHA-256 злоумышленник, зная хеш и длину секрета, может дописать данные и пересчитать валидный хеш. HMAC этому не подвержен.
- **Выбор хеша:** HMAC-SHA256 — стандарт по умолчанию. HMAC даже поверх MD5/SHA-1 остаётся относительно стойким (атаки на коллизии хеша не ломают HMAC напрямую), но новые системы должны использовать SHA-256+.

## Основные функции/классы/методы

### Базовое вычисление HMAC

```python
import hmac
import hashlib

key = b"super-secret-key"
message = b"important payload"

# Способ 1: one-shot через hmac.new
mac = hmac.new(key, message, hashlib.sha256)
print(mac.hexdigest())        # hex-строка подписи
print(mac.digest())           # сырые байты

# Способ 2: digestmod можно задать строкой
mac = hmac.new(key, message, "sha256")

# Способ 3: инкрементально (для потоков/больших данных)
mac = hmac.new(key, digestmod="sha256")
mac.update(b"part1")
mac.update(b"part2")
signature = mac.hexdigest()
```

### hmac.digest — быстрый one-shot (3.7+)

```python
import hmac

# Оптимизированная функция, не создаёт объект — быстрее для коротких сообщений
sig = hmac.digest(b"key", b"message", "sha256")   # -> bytes
```

### Безопасное сравнение (защита от timing-атак)

```python
import hmac

def verify(key: bytes, message: bytes, received_sig: bytes) -> bool:
    expected = hmac.digest(key, message, "sha256")
    # constant-time сравнение: время не зависит от того, где первое расхождение
    return hmac.compare_digest(expected, received_sig)
```

### Пример: проверка webhook (как у GitHub/Stripe)

```python
import hmac
import hashlib

def verify_github_webhook(secret: bytes, body: bytes, header: str) -> bool:
    """
    header — значение X-Hub-Signature-256 вида 'sha256=<hex>'.
    """
    if not header.startswith("sha256="):
        return False
    received = header.split("=", 1)[1]
    expected = hmac.new(secret, body, hashlib.sha256).hexdigest()
    # сравниваем в постоянном времени
    return hmac.compare_digest(expected, received)


def sign_request(secret: bytes, body: bytes) -> str:
    """Сторона-отправитель формирует подпись."""
    return "sha256=" + hmac.new(secret, body, hashlib.sha256).hexdigest()
```

### Подпись и проверка токена (упрощённый JWT-подобный механизм)

```python
import hmac
import hashlib
import base64
import json

SECRET = b"app-signing-secret"

def sign_token(payload: dict) -> str:
    body = base64.urlsafe_b64encode(json.dumps(payload).encode()).rstrip(b"=")
    sig = hmac.new(SECRET, body, hashlib.sha256).digest()
    sig_b64 = base64.urlsafe_b64encode(sig).rstrip(b"=")
    return f"{body.decode()}.{sig_b64.decode()}"

def verify_token(token: str) -> dict | None:
    try:
        body_str, sig_str = token.split(".")
    except ValueError:
        return None
    body = body_str.encode()
    expected = hmac.new(SECRET, body, hashlib.sha256).digest()
    received = base64.urlsafe_b64decode(sig_str + "==")
    if not hmac.compare_digest(expected, received):   # анти-timing
        return None
    pad = "=" * (-len(body_str) % 4)
    return json.loads(base64.urlsafe_b64decode(body_str + pad))
```

### Свойства объекта HMAC

```python
import hmac
mac = hmac.new(b"key", b"msg", "sha256")
mac.name          # 'hmac-sha256'
mac.digest_size   # 32
mac.block_size    # 64
mac.copy()        # копия состояния (для расчёта нескольких вариантов)
```

## Частые вопросы на собеседовании

**Q1. Чем HMAC отличается от обычного хеша?**
Обычный хеш (SHA-256) детерминирован и пересчитывается кем угодно — он даёт только целостность от случайных повреждений. HMAC использует секретный ключ, поэтому корректную подпись может создать только владелец ключа → даёт целостность И аутентичность (защиту от подделки).

**Q2. Почему нельзя использовать `hash(secret + message)` вместо HMAC?**
Из-за length-extension атаки: для хешей на конструкции Меркла–Дамгора (MD5, SHA-1, SHA-256) злоумышленник, зная `hash(secret+msg)` и длину секрета, может вычислить `hash(secret+msg+padding+extra)` без знания секрета и подделать сообщение. HMAC своей двойной конструкцией этому не подвержен.

**Q3. Чем HMAC отличается от цифровой подписи (RSA/ECDSA)?**
HMAC симметричен: один общий секрет, проверяющий должен знать тот же ключ, что и подписант → нельзя доказать третьей стороне авторство (нет неотрекаемости). Цифровая подпись асимметрична: подписывают приватным ключом, проверяют публичным, есть неотрекаемость. HMAC быстрее и проще, подпись — когда нужна верификация без раскрытия секрета.

**Q4. Зачем `hmac.compare_digest` вместо `==`?**
`==` для байтов сравнивает побайтово и выходит на первом несовпадении → время сравнения утечкой выдаёт, сколько байт совпало, что позволяет угадывать подпись побайтно (timing-атака). `compare_digest` работает за постоянное время, не зависящее от позиции расхождения.

**Q5. Можно ли использовать HMAC с MD5 или SHA-1?**
Технически да, и HMAC-MD5/HMAC-SHA1 остаются на удивление стойкими, потому что коллизионные атаки на хеш не переносятся напрямую на HMAC. Но для новых систем стандарт — HMAC-SHA256 и выше; legacy постепенно выводят.

**Q6. Какой длины должен быть секретный ключ для HMAC?**
Рекомендуется ключ длиной не меньше размера выхода хеша (для SHA-256 — 32 байта случайных данных). Слишком короткий ключ снижает стойкость; ключ длиннее block_size хешируется внутри HMAC. Генерируйте через `secrets.token_bytes(32)`.

**Q7. Где на практике применяется HMAC?**
Подпись webhook'ов (GitHub `X-Hub-Signature-256`, Stripe, Slack), JWT с алгоритмом HS256, подпись cookie/сессий, AWS Signature V4 для подписи API-запросов, проверка целостности сообщений в защищённых протоколах.

**Q8. Что вернёт `compare_digest`, если длины разные?**
Вернёт `False`. Важно: для строк используйте только ASCII; лучше сравнивать `bytes`. Функция всё равно старается не утекать информацию через тайминг, но разные длины она различает.

**Q9. HMAC шифрует данные?**
Нет. HMAC не шифрует и не скрывает содержимое сообщения — он только подписывает. Для конфиденциальности нужно отдельное шифрование (AES). Часто комбинируют: «encrypt-then-MAC» или используют AEAD-режимы (AES-GCM, ChaCha20-Poly1305).

**Q10. Можно ли считать HMAC потоково для большого файла?**
Да: `mac = hmac.new(key, digestmod="sha256")` и многократный `mac.update(chunk)`. Это удобно для подписи больших данных без загрузки в память.

## Подводные камни (gotchas)

- **Сравнение через `==`** — главная ошибка: открывает timing-атаку. Всегда `hmac.compare_digest`.
- **Самодельный MAC `hash(secret+msg)`** — уязвим к length-extension. Используйте HMAC или keyed-BLAKE2.
- **Слабый/предсказуемый ключ** — HMAC настолько надёжен, насколько секретен ключ. Не хардкодьте ключи в коде, генерируйте через `secrets`.
- **Путаница MAC и подписи** — HMAC не даёт неотрекаемости; если нужна верификация третьей стороной, нужна асимметричная подпись.
- **Подпись не того, что нужно** — подписывать надо «сырое» тело запроса до парсинга; повторная сериализация (JSON) может изменить байты и сломать проверку.
- **Сравнение строк разного регистра/кодировки** — `compare_digest` для hex-строк чувствителен к регистру; нормализуйте или сравнивайте байты `digest()`.
- **Передача ключа как `str`** — `hmac.new` требует `bytes` для ключа; `str` вызовет ошибку.
- **HMAC ≠ шифрование** — данные остаются видимыми; для секретности нужен отдельный слой.
- **Повтор сообщений (replay)** — HMAC не защищает от повторной отправки валидного сообщения; добавляйте timestamp/nonce и проверяйте их.

## Лучшие практики

- Всегда используйте `hmac.compare_digest` для проверки подписей.
- По умолчанию — HMAC-SHA256; не изобретайте самодельные MAC.
- Ключ — случайный, не меньше 32 байт, из `secrets.token_bytes`; храните в секрет-хранилище/переменных окружения, не в коде.
- Подписывайте точные байты payload (raw body), а не пересериализованные данные.
- Защищайтесь от replay: включайте timestamp/nonce в подписываемое сообщение и проверяйте свежесть.
- Для конфиденциальности добавляйте шифрование (encrypt-then-MAC или AEAD).
- Ротация ключей: поддерживайте идентификатор ключа (key id), чтобы менять секрет без простоя.
- Для потоков/больших данных считайте HMAC инкрементально через `update`.

## Шпаргалка

```python
import hmac, hashlib, secrets

# --- Генерация ключа ---
key = secrets.token_bytes(32)

# --- Вычисление HMAC ---
hmac.new(key, b"msg", hashlib.sha256).hexdigest()   # объект -> hex
hmac.new(key, b"msg", "sha256").digest()            # сырые байты
hmac.digest(key, b"msg", "sha256")                  # быстрый one-shot (3.7+)

# --- Инкрементально (потоково) ---
m = hmac.new(key, digestmod="sha256")
m.update(b"chunk1"); m.update(b"chunk2")
sig = m.digest()

# --- Безопасная проверка (анти-timing) ---
ok = hmac.compare_digest(expected, received)

# --- Проверка webhook ---
def verify(secret, body, recv_hex):
    expected = hmac.new(secret, body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, recv_hex)

# Свойства: m.name 'hmac-sha256', m.digest_size 32, m.block_size 64

# ПРАВИЛА:
#  - проверка только через compare_digest (НЕ ==)
#  - не делать hash(secret+msg) — length-extension; брать HMAC
#  - ключ из secrets, >=32 байт, не в коде
#  - HMAC != шифрование; от replay — timestamp/nonce
```
