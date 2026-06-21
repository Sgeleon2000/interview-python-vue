# email — подготовка к собеседованию

## Что это и зачем

`email` — стандартный пакет Python для **создания, разбора (парсинга), редактирования и сериализации** электронных писем в формате [RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322) (старый RFC 2822) и MIME ([RFC 2045–2049](https://datatracker.ietf.org/doc/html/rfc2045)).

Важно понимать: пакет `email` **не отправляет** письма. Он только строит их структуру и превращает в байты/строку. Отправкой занимается `smtplib`, получением — `imaplib`/`poplib`. `email` — это «модель данных» письма.

Зачем нужен на практике:
- Сборка писем с HTML, текстом и вложениями.
- Парсинг входящих писем (из IMAP/POP) — извлечение темы, отправителя, тела, вложений.
- Корректная работа с кодировками заголовков (русский текст, эмодзи) и тел.
- Генерация уведомлений, отчётов, рассылок.

С Python 3.6 рекомендуется **новый API** на базе `email.message.EmailMessage` и `email.policy.default`. Старый API (`email.mime.*`, `MIMEMultipart`, `MIMEText`) до сих пор широко встречается в коде и на собеседованиях, поэтому знать нужно оба.

## Ключевые концепции

- **Письмо — это дерево объектов `Message`/`EmailMessage`.** Каждый узел имеет заголовки и либо тело (payload-строка/байты), либо список под-частей (multipart).
- **MIME** описывает, как упаковать разнородный контент. Ключевые типы:
  - `multipart/mixed` — основное тело + вложения.
  - `multipart/alternative` — несколько версий одного содержимого (text/plain и text/html), клиент выбирает лучшую.
  - `multipart/related` — HTML + встроенные ресурсы (картинки через `cid:`).
- **Заголовки** (`Subject`, `From`, `To`, `Content-Type`, `Content-Transfer-Encoding` и т.д.). Не-ASCII в заголовках кодируется по [RFC 2047](https://datatracker.ietf.org/doc/html/rfc2047) (encoded-word, `=?utf-8?b?...?=`).
- **Policy** — объект, управляющий поведением парсинга/сериализации. `email.policy.default` (новый API, отдаёт `EmailMessage`), `email.policy.compat32` (по умолчанию, старый API, отдаёт `Message`).
- **Content-Transfer-Encoding** — как байты тела представлены в ASCII-потоке: `7bit`, `8bit`, `quoted-printable`, `base64`.

## Основные функции/классы/методы

### Создание письма (новый API — рекомендуется)

```python
from email.message import EmailMessage

msg = EmailMessage()
msg["Subject"] = "Привет"           # не-ASCII кодируется автоматически (RFC 2047)
msg["From"] = "Иван <ivan@example.com>"
msg["To"] = "petr@example.com, sveta@example.com"
msg.set_content("Здравствуйте!\nЭто простое текстовое письмо.")  # text/plain, utf-8

print(msg.as_string())  # сериализация в строку (для отладки)
print(bytes(msg))       # байты, готовые к передаче в smtplib
```

`set_content()` сам выставляет `Content-Type`, кодировку и `Content-Transfer-Encoding`.

### Письмо с HTML и текстовой альтернативой

```python
from email.message import EmailMessage

msg = EmailMessage()
msg["Subject"] = "Отчёт"
msg["From"] = "report@example.com"
msg["To"] = "boss@example.com"

# Сначала задаём текстовую версию...
msg.set_content("Текстовая версия для клиентов без HTML.")
# ...затем добавляем HTML-альтернативу. Структура станет multipart/alternative
msg.add_alternative(
    "<html><body><h1>Отчёт</h1><p>HTML-версия письма.</p></body></html>",
    subtype="html",
)
```

### Вложения

```python
from email.message import EmailMessage
import mimetypes

msg = EmailMessage()
msg["Subject"] = "Документы"
msg["From"] = "a@example.com"
msg["To"] = "b@example.com"
msg.set_content("Во вложении договор и фото.")

# PDF-вложение
with open("contract.pdf", "rb") as f:
    data = f.read()
ctype, _ = mimetypes.guess_type("contract.pdf")  # 'application/pdf'
maintype, subtype = ctype.split("/", 1)
msg.add_attachment(data, maintype=maintype, subtype=subtype,
                   filename="Договор.pdf")  # имя с кириллицей закодируется по RFC 2231

# Картинка
with open("photo.jpg", "rb") as f:
    msg.add_attachment(f.read(), maintype="image", subtype="jpeg",
                       filename="photo.jpg")
```

`add_attachment` автоматически превращает письмо в `multipart/mixed`, выбирает `Content-Transfer-Encoding: base64` для бинарных данных и ставит `Content-Disposition: attachment`.

### Встроенная картинка в HTML (cid)

```python
from email.message import EmailMessage
from email.utils import make_msgid

msg = EmailMessage()
msg["Subject"] = "Письмо с картинкой"
msg["From"] = "a@example.com"
msg["To"] = "b@example.com"

cid = make_msgid()                       # вид: <unique@host>
cid_clean = cid[1:-1]                     # убираем угловые скобки для src

msg.set_content("Версия без картинок.")
msg.add_alternative(
    f'<html><body><img src="cid:{cid_clean}"></body></html>',
    subtype="html",
)
# Прикрепляем картинку к HTML-части (related)
with open("logo.png", "rb") as f:
    msg.get_payload()[1].add_related(
        f.read(), maintype="image", subtype="png", cid=cid
    )
```

### Парсинг письма

```python
from email import message_from_bytes, message_from_string
from email.parser import BytesParser
from email.policy import default

raw = b"...сырые байты письма из IMAP..."

# Современный способ — получаем EmailMessage благодаря policy=default
msg = message_from_bytes(raw, policy=default)
# Эквивалентно:
# msg = BytesParser(policy=default).parsebytes(raw)

print(msg["Subject"])     # уже декодированная строка (новый API)
print(msg["From"])
print(msg.get_content_type())

# Обход дерева
for part in msg.walk():
    print(part.get_content_type(), part.get_content_disposition())
```

### Извлечение тела и вложений

```python
from email import message_from_bytes
from email.policy import default

msg = message_from_bytes(raw, policy=default)

# Главное тело (новый API сам выбирает text/plain или text/html)
body = msg.get_body(preferencelist=("plain", "html"))
if body is not None:
    print(body.get_content())   # уже декодированная строка с учётом кодировки

# Вложения
for att in msg.iter_attachments():
    fname = att.get_filename()
    payload = att.get_content()   # bytes для бинарных вложений
    print(fname, len(payload))
    # with open(fname, "wb") as f: f.write(payload)
```

### Декодирование заголовков (старый API)

В старом API заголовки приходят «как есть» (encoded-words), их надо декодировать вручную:

```python
from email.header import decode_header, make_header

raw_subject = "=?utf-8?b?0J/RgNC40LLQtdGC?="
parts = decode_header(raw_subject)          # [(b'\xd0...', 'utf-8')]
subject = str(make_header(parts))           # 'Привет'
```

### Старый API (надо знать, встречается в legacy)

```python
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.application import MIMEApplication

msg = MIMEMultipart("mixed")
msg["Subject"] = "Старый API"
msg["From"] = "a@example.com"
msg["To"] = "b@example.com"

alt = MIMEMultipart("alternative")
alt.attach(MIMEText("Текст", "plain", "utf-8"))
alt.attach(MIMEText("<b>HTML</b>", "html", "utf-8"))
msg.attach(alt)

part = MIMEApplication(b"...pdf bytes...", _subtype="pdf")
part.add_header("Content-Disposition", "attachment", filename="doc.pdf")
msg.attach(part)
```

### Полезные утилиты `email.utils`

```python
from email.utils import (
    formataddr, parseaddr, getaddresses,
    formatdate, parsedate_to_datetime, make_msgid,
)

formataddr(("Иван Петров", "ivan@example.com"))   # 'Иван Петров <ivan@example.com>'
parseaddr("Иван <ivan@example.com>")              # ('Иван', 'ivan@example.com')
getaddresses(["a@x.com, b@y.com"])                # [('', 'a@x.com'), ('', 'b@y.com')]
formatdate(localtime=True)                        # дата в формате RFC 2822
parsedate_to_datetime("Mon, 20 Jun 2026 12:00:00 +0300")  # datetime
make_msgid(domain="example.com")                  # уникальный Message-ID
```

### Связка с отправкой (smtplib)

```python
import smtplib
# msg — наш EmailMessage
with smtplib.SMTP("smtp.example.com", 587) as s:
    s.starttls()
    s.login("user", "password")
    s.send_message(msg)   # сам берёт From/To из заголовков
```

## Частые вопросы на собеседовании

**Q: Отправляет ли модуль `email` письма?**
A: Нет. `email` только строит и парсит структуру письма. Отправка — `smtplib` (SMTP), получение — `imaplib`/`poplib`. Часто путают.

**Q: В чём разница между `multipart/mixed`, `multipart/alternative` и `multipart/related`?**
A: `mixed` — тело + независимые вложения. `alternative` — несколько версий ОДНОГО содержимого (plain и html), клиент показывает одну. `related` — HTML вместе со встроенными ресурсами (картинки по `cid:`). Типичное «богатое» письмо с вложением: `mixed[ alternative[plain, related[html, image]], attachment ]`.

**Q: Чем новый API (`EmailMessage` + `policy.default`) лучше старого (`MIMEText`/`MIMEMultipart`)?**
A: Новый API автоматически управляет кодировками, заголовками и структурой: `set_content`, `add_alternative`, `add_attachment`, `get_content`/`get_body`, `iter_attachments`. Заголовки декодируются автоматически при парсинге. Старый API требует ручной сборки дерева и ручного `decode_header`.

**Q: Как закодировать русскую тему письма?**
A: В новом API — просто присвоить строку: `msg["Subject"] = "Привет"`, кодирование по RFC 2047 произойдёт само. Вручную — `email.header.Header("Привет", "utf-8")`.

**Q: Что такое `Content-Transfer-Encoding` и какие значения важны?**
A: Способ представления байтов тела в потоке. `7bit`/`8bit` — почти как есть; `quoted-printable` — для текста с редкими не-ASCII символами (читаемо); `base64` — для бинарных данных и текста с большим объёмом не-ASCII.

**Q: Как извлечь все вложения из входящего письма?**
A: Распарсить с `policy=default` и пройти `msg.iter_attachments()`, для каждого взять `get_filename()` и `get_content()`. В старом API — `msg.walk()` + проверка `get_content_disposition() == 'attachment'` + `get_payload(decode=True)`.

**Q: Зачем нужен параметр `policy`?**
A: Policy определяет правила парсинга/генерации: длину строк, перенос заголовков, какой класс возвращать. `default` → `EmailMessage` (новый API), `compat32` (по умолчанию) → `Message` (старый, обратная совместимость). Для нового кода всегда передавайте `policy=default`.

**Q: Как декодировать заголовок `=?utf-8?b?...?=` в старом API?**
A: `decode_header()` возвращает список пар `(bytes, charset)`, собрать строку через `str(make_header(parts))`.

**Q: В чём опасность парсинга недоверенных писем?**
A: Заголовки могут содержать инъекции (особенно при ручной сборке `To`/`Subject` из пользовательского ввода → header injection через `\r\n`). Имена вложений — потенциальный path traversal (`../../etc/passwd`), их нужно санитизировать перед сохранением.

**Q: Как вставить картинку прямо в тело HTML?**
A: Через `cid:` — сгенерировать `make_msgid()`, в HTML указать `src="cid:<id без скобок>"`, прикрепить картинку через `add_related(..., cid=cid)`.

**Q: Чем `get_payload()` отличается от `get_content()`?**
A: `get_payload(decode=True)` (старый API) возвращает сырые декодированные байты CTE. `get_content()` (новый API) умнее: для текста возвращает уже `str` с учётом charset, для бинарных — `bytes`, для multipart кидает ошибку.

## Подводные камни (gotchas)

- **Header injection.** Никогда не подставляйте пользовательский ввод напрямую в заголовки старым способом конкатенации. `\r\n` в значении заголовка может добавить лишние заголовки. Новый API экранирует, но всё равно валидируйте адреса.
- **Порядок частей в `alternative` важен.** По стандарту последняя часть — «лучшая». Поэтому сначала `set_content` (plain), потом `add_alternative(..., subtype="html")` — клиент покажет HTML.
- **Старый и новый API не смешивать.** Если парсите с `policy=default`, не используйте методы/привычки старого `Message`.
- **`as_string()` ломает бинарные данные.** Для отправки используйте `bytes(msg)` или `s.send_message(msg)`, а не `as_string()`.
- **`Message`-заголовки нельзя переприсвоить простым `=`.** `msg["To"] = ...` ДОБАВЛЯЕТ заголовок, а не заменяет. Для замены: `del msg["To"]; msg["To"] = ...` или `msg.replace_header("To", ...)`.
- **Имена вложений с кириллицей.** Новый API кодирует их по RFC 2231; старые клиенты могут показывать криво. Иногда приходится упрощать имена.
- **`get_filename()` может вернуть `None`** — у inline-частей без `filename`. Всегда проверяйте.
- **Charset не всегда указан корректно** во входящих письмах; `get_content()` может бросить `UnicodeDecodeError` или вернуть мусор. Иногда нужен fallback с `get_payload(decode=True)` и ручным декодированием.

## Лучшие практики

- Используйте **новый API**: `EmailMessage` + `from email.policy import default` для парсинга.
- Для тела письма — `set_content()` (текст) и `add_alternative(..., subtype="html")` (HTML).
- Для вложений — `add_attachment(data, maintype=..., subtype=..., filename=...)`, тип определяйте через `mimetypes.guess_type`.
- Не собирайте MIME-границы и заголовки вручную, если можно избежать.
- Валидируйте и санитизируйте имена вложений перед сохранением на диск (`os.path.basename`, белый список символов).
- Адреса собирайте через `email.utils.formataddr`, парсите через `parseaddr`/`getaddresses`.
- При отправке используйте `smtp.send_message(msg)` вместо ручного `sendmail` со строкой.
- Логируйте `Message-ID` для трассировки писем.

## Шпаргалка

```python
from email.message import EmailMessage
from email import message_from_bytes
from email.policy import default
from email.utils import formataddr, parseaddr, make_msgid
import mimetypes

# --- Создание ---
msg = EmailMessage()
msg["Subject"] = "Тема"
msg["From"] = formataddr(("Имя", "from@x.com"))
msg["To"] = "to@x.com"
msg.set_content("plain текст")                       # text/plain
msg.add_alternative("<b>html</b>", subtype="html")    # -> multipart/alternative

# Вложение
data = b"..."
ctype, _ = mimetypes.guess_type("file.pdf")
maintype, subtype = ctype.split("/", 1)
msg.add_attachment(data, maintype=maintype, subtype=subtype, filename="file.pdf")

raw = bytes(msg)                                      # для отправки

# --- Парсинг ---
m = message_from_bytes(raw, policy=default)
m["Subject"]                                          # декодированная строка
body = m.get_body(preferencelist=("plain", "html"))
text = body.get_content() if body else ""
for att in m.iter_attachments():
    name, content = att.get_filename(), att.get_content()

# --- Замена заголовка ---
del msg["To"]; msg["To"] = "new@x.com"

# --- Утилиты ---
parseaddr("Имя <a@x.com>")        # ('Имя', 'a@x.com')
make_msgid(domain="x.com")        # уникальный Message-ID
```
