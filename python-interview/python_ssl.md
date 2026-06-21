# ssl — подготовка к собеседованию

## Что это и зачем

Модуль `ssl` даёт доступ к **TLS/SSL** (Transport Layer Security) поверх сокетов — шифрование, аутентификацию и целостность данных при сетевом обмене. Это то, что превращает HTTP в **HTTPS**.

TLS решает три задачи:
1. **Конфиденциальность** — данные шифруются, их нельзя прочитать в пути.
2. **Целостность** — нельзя незаметно изменить данные.
3. **Аутентификация** — клиент убеждается, что общается с настоящим сервером (по сертификату), а при mTLS сервер проверяет клиента.

SSL — устаревшее название протокола (SSLv2/SSLv3 небезопасны и отключены); современный протокол — **TLS** (актуальны TLS 1.2 и TLS 1.3), но имя модуля и многих классов осталось «ssl».

## Ключевые концепции

### TLS handshake (рукопожатие)
Перед передачей данных стороны договариваются о версии протокола и наборе шифров (cipher suite), сервер предъявляет **сертификат**, проверяется его подлинность, согласуются сессионные ключи. После этого весь трафик шифруется.

### Сертификаты и центры сертификации (CA)
Сертификат сервера подписан **доверенным удостоверяющим центром** (CA). Клиент имеет список доверенных корневых CA и по цепочке проверяет, что сертификат сервера подлинный и выдан для нужного домена. Самоподписанные сертификаты не проходят такую проверку без явного добавления в доверенные.

### Проверка имени хоста (hostname verification)
Помимо валидности сертификата, проверяется, что он выдан **именно для запрашиваемого домена** (CN/SAN). Без этой проверки злоумышленник с валидным сертификатом на другой домен мог бы выдать себя за сервер.

### SSLContext
Центральный объект: хранит настройки TLS — версии протокола, набор шифров, сертификаты, режим проверки. Из контекста «оборачивают» обычные сокеты в защищённые.

### mTLS (взаимная аутентификация)
Не только клиент проверяет сервер, но и сервер требует/проверяет сертификат клиента. Применяется в межсервисном взаимодействии, банках.

## Основные функции/классы/методы

### Создание контекста (рекомендуемый способ)
```python
import ssl

# Контекст для КЛИЕНТА с безопасными настройками по умолчанию:
# проверка сертификата + проверка имени хоста включены
client_ctx = ssl.create_default_context()

# Контекст для СЕРВЕРА
server_ctx = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
```

`create_default_context()` — предпочтительный способ: он включает проверку сертификатов (`CERT_REQUIRED`), проверку hostname, отключает небезопасные протоколы и шифры.

### TLS-клиент (обёртка сокета)
```python
import socket
import ssl

hostname = "example.com"
context = ssl.create_default_context()   # безопасные настройки

with socket.create_connection((hostname, 443)) as sock:
    # Оборачиваем TCP-сокет в TLS; server_hostname нужен для SNI и проверки имени
    with context.wrap_socket(sock, server_hostname=hostname) as ssock:
        print("Версия:", ssock.version())        # например TLSv1.3
        ssock.sendall(b"GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
        data = ssock.recv(2048)
        print(data[:100])
        cert = ssock.getpeercert()               # сертификат сервера
        print("Сертификат:", cert.get("subject"))
```

### TLS-сервер
```python
import socket
import ssl

# Контекст сервера загружает свой сертификат и приватный ключ
context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
context.load_cert_chain(certfile="server.crt", keyfile="server.key")

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server:
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(("0.0.0.0", 8443))
    server.listen(5)
    print("HTTPS-сервер на :8443")
    while True:
        conn, addr = server.accept()
        # Оборачиваем соединение клиента в TLS (handshake)
        with context.wrap_socket(conn, server_side=True) as ssock:
            data = ssock.recv(1024)
            ssock.sendall(b"HTTP/1.0 200 OK\r\n\r\nHello over TLS")
```

### Загрузка сертификатов и доверенных CA
```python
context = ssl.create_default_context()
# Доверять собственному CA (например, корпоративному)
context.load_verify_locations(cafile="my-ca.pem")

# Серверу — указать его сертификат и ключ
context.load_cert_chain(certfile="server.crt", keyfile="server.key")
```

### Настройка проверки и версий
```python
context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
context.check_hostname = True                       # проверять имя хоста
context.verify_mode = ssl.CERT_REQUIRED             # требовать валидный сертификат
context.minimum_version = ssl.TLSVersion.TLSv1_2    # запретить старые протоколы
```

Режимы `verify_mode`:
- `CERT_NONE` — не проверять (НЕБЕЗОПАСНО, только для тестов).
- `CERT_OPTIONAL` — проверять, если предоставлен.
- `CERT_REQUIRED` — требовать и проверять (по умолчанию у клиента).

### mTLS — сервер требует клиентский сертификат
```python
server_ctx = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
server_ctx.load_cert_chain("server.crt", "server.key")
server_ctx.verify_mode = ssl.CERT_REQUIRED          # требовать сертификат клиента
server_ctx.load_verify_locations("client-ca.pem")   # доверять CA клиентов
```

### Отключение проверки (антипаттерн, только осознанно)
```python
context = ssl.create_default_context()
context.check_hostname = False
context.verify_mode = ssl.CERT_NONE   # ОПАСНО: уязвимость к MITM!
```

### Интеграция с HTTPS-клиентами
```python
import urllib.request
import ssl

context = ssl.create_default_context()
context.load_verify_locations("corporate-ca.pem")   # доверять корпоративному CA
with urllib.request.urlopen("https://internal.example.com", context=context) as r:
    print(r.status)
```

## Частые вопросы на собеседовании

**Q1. В чём разница между SSL и TLS?**
SSL — устаревшее название и протоколы (SSLv2/SSLv3, небезопасны, отключены). TLS — современный преемник (актуальны TLS 1.2 и 1.3). Имя модуля `ssl` историческое, но реально используется TLS.

**Q2. Что такое `SSLContext` и зачем он нужен?**
Объект с настройками TLS: версии протокола, шифры, сертификаты, режим проверки. Из него оборачивают сокеты (`wrap_socket`). Позволяет переиспользовать единые безопасные настройки для множества соединений.

**Q3. Как клиент проверяет сервер?**
По цепочке доверия: сертификат сервера должен быть подписан доверенным CA (есть в хранилище), не просрочен, не отозван, и выдан для запрашиваемого домена (проверка hostname по SAN/CN). При несоответствии — `ssl.SSLCertVerificationError`.

**Q4. Зачем нужен `create_default_context()` вместо ручной настройки?**
Он включает безопасные настройки «из коробки»: проверку сертификата, проверку hostname, отключение слабых протоколов/шифров. Ручная конфигурация легко приводит к уязвимостям.

**Q5. Что делает `wrap_socket()`?**
Оборачивает обычный TCP-сокет в TLS-сокет: выполняет handshake, после чего весь обмен (`send`/`recv`) шифруется прозрачно. Параметр `server_hostname` нужен клиенту для SNI и проверки имени, `server_side=True` — для серверной стороны.

**Q6. Что такое hostname verification и почему важна?**
Проверка, что сертификат выдан именно для запрашиваемого домена. Без неё злоумышленник с валидным сертификатом на другой домен мог бы выдать себя за сервер (MITM). Управляется `context.check_hostname`.

**Q7. Чем опасно `CERT_NONE` / `check_hostname=False`?**
Отключает проверку подлинности сервера — открывает дорогу атаке «человек посередине» (MITM): соединение шифруется, но вы не знаете, с кем. Допустимо лишь в изолированных тестах.

**Q8. Что такое mTLS и где применяется?**
Mutual TLS — взаимная аутентификация: сервер тоже требует и проверяет сертификат клиента (`verify_mode=CERT_REQUIRED` + доверенный CA клиентов). Используется в межсервисном взаимодействии, zero-trust сетях, банках.

**Q9. Что такое SNI?**
Server Name Indication — расширение TLS, в котором клиент в начале handshake сообщает имя запрашиваемого хоста. Позволяет одному IP обслуживать много доменов с разными сертификатами. В Python задаётся через `server_hostname` в `wrap_socket`.

**Q10. Как доверять самоподписанному или корпоративному сертификату?**
Добавить CA/сертификат в доверенные через `context.load_verify_locations(cafile=...)`. Это безопаснее, чем полностью отключать проверку.

**Q11. Чем `load_cert_chain` отличается от `load_verify_locations`?**
`load_cert_chain` загружает **собственный** сертификат и приватный ключ (для сервера или клиента в mTLS), которые предъявляются другой стороне. `load_verify_locations` добавляет **доверенные CA**, по которым проверяется сертификат собеседника.

## Подводные камни (gotchas)

- **Отключение проверки** (`CERT_NONE`, `check_hostname=False`) ради «чтобы заработало» — критическая уязвимость к MITM.
- **Забытый `server_hostname`** у клиента → ошибка проверки имени или невозможность SNI.
- **Самоподписанные сертификаты** не проходят проверку без добавления в доверенные — не отключайте проверку, а добавьте CA.
- **Просроченные/отозванные сертификаты** дают `SSLCertVerificationError`.
- **Старые версии TLS** (1.0/1.1) небезопасны — задавайте `minimum_version = TLSVersion.TLSv1_2`.
- **Несовпадение цепочки** — сервер должен отдавать полную цепочку (свой сертификат + промежуточные CA), иначе клиент не построит путь доверия.
- **Перепутанные роли контекста** — для сервера нужен `Purpose.CLIENT_AUTH`, для клиента `Purpose.SERVER_AUTH` (по умолчанию).
- **Блокирующий handshake** на неблокирующих сокетах требует обработки `SSLWantReadError`/`SSLWantWriteError`.

## Лучшие практики

- Всегда начинайте с **`ssl.create_default_context()`** — безопасные настройки по умолчанию.
- **Никогда не отключайте** проверку сертификата и hostname в продакшене.
- Задавайте **`minimum_version = ssl.TLSVersion.TLSv1_2`** (а лучше требуйте 1.3, где возможно).
- Доверяйте кастомным CA через **`load_verify_locations`**, а не отключением проверки.
- Храните приватные ключи **в безопасности**, с правильными правами доступа; не коммитьте в репозиторий.
- Для mTLS — настраивайте `CERT_REQUIRED` и доверенный CA клиентов на сервере.
- Используйте **высокоуровневые** библиотеки (`requests`, `httpx`, `urllib`) — они корректно применяют ssl-контексты.
- Регулярно **обновляйте** сертификаты и следите за сроком действия.

## Шпаргалка

```python
import ssl, socket

# --- Клиент (безопасно) ---
ctx = ssl.create_default_context()                 # проверка + hostname включены
with socket.create_connection((host, 443)) as s:
    with ctx.wrap_socket(s, server_hostname=host) as ss:
        ss.sendall(b"...")
        ss.recv(2048)
        ss.version()          # версия TLS
        ss.getpeercert()      # сертификат сервера

# --- Сервер ---
ctx = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
ctx.load_cert_chain("server.crt", "server.key")
ssock = ctx.wrap_socket(conn, server_side=True)

# --- Доверие кастомному CA ---
ctx.load_verify_locations(cafile="my-ca.pem")

# --- Настройки проверки ---
ctx.check_hostname = True
ctx.verify_mode = ssl.CERT_REQUIRED                # NONE/OPTIONAL/REQUIRED
ctx.minimum_version = ssl.TLSVersion.TLSv1_2

# --- mTLS на сервере ---
ctx.verify_mode = ssl.CERT_REQUIRED
ctx.load_verify_locations("client-ca.pem")
```
