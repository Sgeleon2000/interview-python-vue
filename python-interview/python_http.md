# http — подготовка к собеседованию

## Что это и зачем

`http` — пакет стандартной библиотеки Python, содержащий низкоуровневые средства для работы с протоколом HTTP:

- `http.client` — низкоуровневый **HTTP-клиент** (`HTTPConnection`, `HTTPSConnection`). На нём построены `urllib.request` и `requests`.
- `http.server` — простой **HTTP-сервер** (`HTTPServer`, `BaseHTTPRequestHandler`, `SimpleHTTPRequestHandler`). Для разработки/тестов, не для продакшена.
- `http.HTTPStatus` — перечисление **кодов состояния** HTTP с именами, числами и описаниями.
- `http.cookies` — формирование и разбор HTTP-cookie (заголовки `Set-Cookie`/`Cookie`).
- `http.cookiejar` — хранилище cookies для клиента (используется с `urllib`).

Зачем знать: понимание устройства HTTP «снизу», быстрый локальный сервер (`python -m http.server`), корректная работа с кодами состояния и cookie. На собеседованиях спрашивают и про сам протокол (методы, статусы, идемпотентность), и про модуль.

## Ключевые концепции

- **HTTP — текстовый запрос/ответ.** Запрос: строка (`METHOD path HTTP/1.1`), заголовки, пустая строка, тело. Ответ: статус-строка (`HTTP/1.1 200 OK`), заголовки, тело.
- **Методы:** `GET` (получить, безопасный, идемпотентный), `POST` (создать/действие, не идемпотентный), `PUT` (заменить, идемпотентный), `PATCH` (частичное обновление), `DELETE` (удалить, идемпотентный), `HEAD` (как GET без тела), `OPTIONS` (возможности).
- **Идемпотентность** — повтор запроса даёт тот же эффект (GET/PUT/DELETE — да, POST — нет).
- **Коды состояния по классам:** 1xx информационные, 2xx успех, 3xx перенаправление, 4xx ошибка клиента, 5xx ошибка сервера.
- **`HTTPStatus`** — `IntEnum`: можно сравнивать с числом, но даёт `.phrase` и `.description`.
- **Cookie** — пары имя=значение, передаются сервером в `Set-Cookie` и возвращаются браузером в `Cookie`. Атрибуты: `Expires`, `Max-Age`, `Path`, `Domain`, `Secure`, `HttpOnly`, `SameSite`.

## Основные функции/классы/методы

### `http.HTTPStatus` — коды состояния

```python
from http import HTTPStatus

HTTPStatus.OK                 # <HTTPStatus.OK: 200>
HTTPStatus.OK == 200          # True (это IntEnum)
HTTPStatus.OK.value           # 200
HTTPStatus.OK.phrase          # 'OK'
HTTPStatus.OK.description     # 'Request fulfilled, document follows'

HTTPStatus.NOT_FOUND          # 404, phrase 'Not Found'
HTTPStatus.INTERNAL_SERVER_ERROR  # 500

# Проверка класса
code = HTTPStatus.CREATED     # 201
print(code.is_success)        # True (2xx)
print(HTTPStatus(404).is_client_error)  # True (4xx)
print(HTTPStatus(503).is_server_error)  # True (5xx)
```

Полезные коды для собеседования:
- `200 OK`, `201 Created`, `204 No Content`
- `301 Moved Permanently`, `302 Found`, `304 Not Modified`, `307/308` (с сохранением метода)
- `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `405 Method Not Allowed`, `409 Conflict`, `422 Unprocessable Entity`, `429 Too Many Requests`
- `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout`

### `http.client` — низкоуровневый клиент

```python
import http.client
import json

conn = http.client.HTTPSConnection("httpbin.org", timeout=10)

# GET
conn.request("GET", "/get?x=1", headers={"Accept": "application/json"})
resp = conn.getresponse()          # HTTPResponse
print(resp.status, resp.reason)    # 200 OK
print(resp.getheader("Content-Type"))
body = resp.read()                  # bytes; ОБЯЗАТЕЛЬНО прочитать перед след. запросом
data = json.loads(body)

# POST с телом
payload = json.dumps({"name": "Иван"})
conn.request("POST", "/post", body=payload,
             headers={"Content-Type": "application/json"})
resp = conn.getresponse()
print(resp.status)
resp.read()

conn.close()
```

Особенности `http.client`:
- Нужно **обязательно прочитать ответ** (`resp.read()`) перед следующим запросом на том же соединении.
- Соединение переиспользуется (keep-alive), что эффективнее, чем `urllib`.
- Это «сырой» уровень — нет редиректов, JSON, удобной обработки ошибок.

### `http.server` — простой сервер

Запуск сервера статики из командной строки (раздаёт текущую директорию):

```bash
python -m http.server 8000
# с привязкой к адресу и директории:
python -m http.server 8000 --bind 127.0.0.1 --directory ./public
```

Простой сервер программно:

```python
from http.server import HTTPServer, SimpleHTTPRequestHandler

server = HTTPServer(("127.0.0.1", 8000), SimpleHTTPRequestHandler)
print("Сервер на http://127.0.0.1:8000")
server.serve_forever()   # блокирует поток
```

### Свой обработчик запросов

```python
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from http import HTTPStatus
import json

class MyHandler(BaseHTTPRequestHandler):
    def _send_json(self, obj, status=HTTPStatus.OK):
        body = json.dumps(obj, ensure_ascii=False).encode("utf-8")
        self.send_response(status)
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def do_GET(self):
        if self.path == "/health":
            self._send_json({"status": "ok"})
        else:
            self._send_json({"error": "not found"}, HTTPStatus.NOT_FOUND)

    def do_POST(self):
        length = int(self.headers.get("Content-Length", 0))
        raw = self.rfile.read(length)                # читаем тело по Content-Length
        try:
            payload = json.loads(raw)
        except json.JSONDecodeError:
            return self._send_json({"error": "bad json"}, HTTPStatus.BAD_REQUEST)
        self._send_json({"received": payload}, HTTPStatus.CREATED)

    def log_message(self, fmt, *args):
        pass   # отключить логирование в stderr

# ThreadingHTTPServer обрабатывает запросы в отдельных потоках
server = ThreadingHTTPServer(("127.0.0.1", 8000), MyHandler)
server.serve_forever()
```

Ключевые элементы обработчика:
- `self.path`, `self.command` (метод), `self.headers` (заголовки запроса).
- `self.rfile` — поток чтения тела, `self.wfile` — поток записи ответа.
- `send_response()` → `send_header()`×N → `end_headers()` → запись в `wfile`.
- Методы `do_GET`, `do_POST`, `do_PUT`, `do_DELETE` и т.д.

### `http.cookies` — формирование Set-Cookie

```python
from http.cookies import SimpleCookie

# Сервер формирует cookie
c = SimpleCookie()
c["session"] = "abc123"
c["session"]["path"] = "/"
c["session"]["httponly"] = True
c["session"]["max-age"] = 3600
c["session"]["samesite"] = "Lax"

print(c["session"].OutputString())
# 'session=abc123; HttpOnly; Max-Age=3600; Path=/; SameSite=Lax'

# В обработчике: self.send_header("Set-Cookie", c["session"].OutputString())

# Разбор заголовка Cookie от клиента
incoming = SimpleCookie()
incoming.load("session=abc123; theme=dark")
print(incoming["session"].value)   # 'abc123'
print(incoming["theme"].value)     # 'dark'
```

### `http.cookiejar` — хранилище cookies на клиенте

```python
from http.cookiejar import CookieJar
from urllib.request import build_opener, HTTPCookieProcessor

jar = CookieJar()
opener = build_opener(HTTPCookieProcessor(jar))
opener.open("https://httpbin.org/cookies/set?token=xyz").read()
print([(c.name, c.value) for c in jar])    # [('token', 'xyz')]
```

## Частые вопросы на собеседовании

**Q: Чем `http.client` отличается от `urllib` и `requests`?**
A: `http.client` — самый низкий уровень (одно соединение, ручное чтение, без редиректов/JSON). `urllib` построен поверх него и добавляет opener'ы, обработчики, parse. `requests` — высокоуровневая сторонняя библиотека (сессии, JSON, пул, удобство).

**Q: Можно ли использовать `http.server` в продакшене?**
A: Нет. Он однопоточный по умолчанию (или наивно-многопоточный через `ThreadingHTTPServer`), без безопасности, балансировки, асинхронности. Только для разработки, тестов и быстрой раздачи файлов.

**Q: В чём разница между `200`, `201`, `204`?**
A: `200 OK` — успех с телом. `201 Created` — создан ресурс (часто с заголовком `Location`). `204 No Content` — успех без тела (например, после DELETE).

**Q: Чем отличаются `301`, `302`, `307`, `308`?**
A: `301`/`308` — постоянное перенаправление; `302`/`307` — временное. Ключевая разница: `301`/`302` исторически могут менять метод на GET при редиректе, а `307`/`308` **сохраняют исходный метод и тело**.

**Q: Что значит идемпотентность? Какие методы идемпотентны?**
A: Повторный одинаковый запрос даёт тот же результат на сервере. Идемпотентны: GET, HEAD, PUT, DELETE, OPTIONS. Не идемпотентен: POST (каждый вызов может создавать новую сущность). PATCH в общем случае не гарантирован идемпотентным.

**Q: Когда вернуть `401`, а когда `403`?**
A: `401 Unauthorized` — не аутентифицирован (нет/неверные креды), клиенту стоит авторизоваться. `403 Forbidden` — аутентифицирован, но нет прав на ресурс.

**Q: Когда `400` vs `422`?**
A: `400 Bad Request` — синтаксически некорректный запрос (битый JSON). `422 Unprocessable Entity` — синтаксис верный, но данные не проходят валидацию (семантика). `422` — не из базового HTTP, из WebDAV, но широко используется в REST API.

**Q: Что такое атрибуты cookie `HttpOnly`, `Secure`, `SameSite`?**
A: `HttpOnly` — недоступна из JS (защита от XSS-кражи). `Secure` — отправляется только по HTTPS. `SameSite` (`Strict`/`Lax`/`None`) — контролирует отправку cookie в кросс-сайтовых запросах (защита от CSRF).

**Q: Как прочитать тело POST-запроса в `BaseHTTPRequestHandler`?**
A: Взять длину из `self.headers["Content-Length"]`, затем `self.rfile.read(length)`. Без длины читать опасно — можно зависнуть.

**Q: Почему в `http.client` нужно читать ответ перед следующим запросом?**
A: Соединение переиспользуется (keep-alive); пока тело предыдущего ответа не вычитано, соединение «занято», и новый `request()` приведёт к ошибке `ResponseNotReady`.

**Q: Что делает `python -m http.server`?**
A: Запускает простой сервер, раздающий файлы текущей (или указанной `--directory`) директории по HTTP. Удобно для быстрой передачи файлов или статики.

## Подводные камни (gotchas)

- **`http.server` не для продакшена** — небезопасен и не масштабируется.
- **Обязательное чтение ответа в `http.client`** перед следующим запросом, иначе `ResponseNotReady`.
- **`resp.read()` отдаёт `bytes`**, как и в `urllib`. Декодируйте сами.
- **Порядок в обработчике сервера строгий:** `send_response` → все `send_header` → `end_headers` → запись в `wfile`. Заголовки после `end_headers` отправить нельзя.
- **Нужно слать `Content-Length` или `Transfer-Encoding`**, иначе keep-alive клиент может зависнуть в ожидании конца тела.
- **`HTTPStatus` — `IntEnum`:** `HTTPStatus.OK == 200` истинно, но в логи/JSON попадёт `<HTTPStatus.OK: 200>`, если не взять `.value`.
- **`SimpleCookie` экранирует значения** с спецсимволами в кавычки; не кладите в cookie произвольные бинарные данные.
- **`BaseHTTPRequestHandler` блокирующий и однопоточный** (с `HTTPServer`). Один медленный клиент блокирует всех. Нужен `ThreadingHTTPServer`.
- **Path traversal:** `SimpleHTTPRequestHandler` в целом защищён, но в собственных обработчиках при отдаче файлов по `self.path` легко открыть `../../etc/passwd` — санитизируйте путь.

## Лучшие практики

- Используйте `HTTPStatus` вместо «магических чисел» — код читаемее (`HTTPStatus.NOT_FOUND`).
- Для реальных HTTP-клиентов берите `requests`/`httpx`; `http.client` — для специальных низкоуровневых нужд.
- `http.server` — только для локальной разработки и тестов; в продакшене — WSGI/ASGI (gunicorn/uvicorn) за реальным веб-сервером.
- В своих обработчиках всегда выставляйте корректные `Content-Type` и `Content-Length`.
- Для cookie аутентификации ставьте `HttpOnly`, `Secure`, разумный `SameSite`.
- Закрывайте соединения `http.client` (`conn.close()`), читайте тела ответов.
- Выбирайте семантически корректные статус-коды (201 при создании, 204 при удалении, 401 vs 403, 400 vs 422).

## Шпаргалка

```python
from http import HTTPStatus
import http.client
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer, SimpleHTTPRequestHandler, HTTPServer
from http.cookies import SimpleCookie

# --- Коды состояния ---
HTTPStatus.OK            # 200, .phrase='OK'
HTTPStatus.CREATED       # 201
HTTPStatus.NO_CONTENT    # 204
HTTPStatus.NOT_FOUND     # 404
HTTPStatus.UNAUTHORIZED  # 401 (нет аутентификации)
HTTPStatus.FORBIDDEN     # 403 (нет прав)
HTTPStatus(503).is_server_error   # True

# --- Клиент ---
conn = http.client.HTTPSConnection("host", timeout=10)
conn.request("GET", "/path")
r = conn.getresponse()           # r.status, r.reason, r.getheader(...)
body = r.read()                  # bytes — обязательно прочитать
conn.close()

# --- Сервер из CLI ---
# python -m http.server 8000 --bind 127.0.0.1 --directory ./public

# --- Свой сервер ---
class H(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(HTTPStatus.OK)
        self.send_header("Content-Type", "text/plain; charset=utf-8")
        self.end_headers()
        self.wfile.write("привет".encode("utf-8"))
ThreadingHTTPServer(("127.0.0.1", 8000), H).serve_forever()

# --- Cookies ---
c = SimpleCookie(); c["sid"] = "abc"; c["sid"]["httponly"] = True
c["sid"].OutputString()          # для заголовка Set-Cookie

# Методы: GET(безопасн./идемпот.) POST(нет) PUT/DELETE(идемпот.) HEAD OPTIONS PATCH
# 301/308 — постоянный редирект, 302/307 — временный (307/308 сохраняют метод)
```
