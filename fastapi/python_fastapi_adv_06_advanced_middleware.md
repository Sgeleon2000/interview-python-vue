# FastAPI Advanced — Продвинутый middleware — конспект и вопросы

## О чём раздел

Middleware (промежуточный слой / «прослойка») — это код, который выполняется **до** того, как запрос дойдёт до обработчика пути (path operation), и **после** того, как обработчик вернёт ответ. Middleware «оборачивает» обработку каждого запроса.

Типичные задачи middleware:
- добавление заголовков ко всем ответам (например, `X-Process-Time`, CORS, security-заголовки);
- логирование всех запросов/ответов;
- измерение времени обработки;
- сжатие тела ответа (GZip);
- редирект с HTTP на HTTPS;
- проверка заголовка `Host`;
- корректная работа за прокси/балансировщиком (X-Forwarded-* заголовки);
- глобальная обработка ошибок, rate limiting, трассировка (tracing).

FastAPI построен на **Starlette**, а Starlette — на спецификации **ASGI**. Поэтому FastAPI поддерживает два уровня middleware:
1. «Высокоуровневый» декоратор `@app.middleware("http")` — удобная обёртка Starlette.
2. «Чистый» ASGI middleware через `app.add_middleware(...)` — более мощный и универсальный механизм.

В этом разделе разбираем именно продвинутый уровень: готовые middleware из Starlette, порядок их выполнения и написание собственного ASGI middleware.

## Ключевые концепции

- **ASGI** — асинхронный интерфейс между сервером (uvicorn) и приложением. Приложение — это вызываемый объект `app(scope, receive, send)`.
  - `scope` — словарь с метаданными запроса (тип, путь, заголовки, метод, тип соединения: `http`/`websocket`/`lifespan`).
  - `receive` — async-функция для получения событий (тело запроса, дисконнект).
  - `send` — async-функция для отправки событий (старт ответа, тело ответа).
- **ASGI middleware** — это объект, который оборачивает другое ASGI-приложение: он сам реализует интерфейс `(scope, receive, send)`, что-то делает и вызывает «следующее» приложение `await self.app(scope, receive, send)`.
- **`app.add_middleware(MiddlewareClass, **options)`** — добавляет ASGI middleware в стек. Работает с любым типом соединения (`http`, `websocket`, `lifespan`).
- **`@app.middleware("http")`** — упрощённый способ от Starlette. Работает **только** для HTTP-запросов, оперирует объектами `Request`/`Response`, не видит websocket и lifespan-события.
- **Порядок выполнения** — middleware образуют «луковицу» (onion). Тот, что добавлен **последним** через `add_middleware`, выполняется **первым** на входе и **последним** на выходе (выполнение идёт в обратном порядке добавления).
- **Готовые middleware Starlette**: `HTTPSRedirectMiddleware`, `TrustedHostMiddleware`, `GZipMiddleware`, `CORSMiddleware`, `SessionMiddleware`.
- **ProxyHeadersMiddleware** — относится к uvicorn (`uvicorn.middleware.proxy_headers`), корректирует клиентский адрес и схему на основе `X-Forwarded-For` / `X-Forwarded-Proto`. Включается флагом `--proxy-headers`.
- **Middleware vs зависимость (Depends)** — middleware глобален и работает на уровне ASGI, зависимость локальна, типизирована и интегрирована в систему DI/документацию.

## Подробный разбор с примерами кода

### 1. Простой HTTP-middleware через декоратор

Самый простой способ — декоратор. Подходит, когда нужно работать с `Request`/`Response` и логика касается только HTTP.

```python
import time
from fastapi import FastAPI, Request, Response
from starlette.types import ASGIApp  # для типизации, опционально

app = FastAPI()


# call_next — функция, которая передаёт запрос дальше по стеку
# и возвращает Response от обработчика пути
@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    # --- код ДО обработчика (на входе) ---
    start_time = time.perf_counter()

    # вызываем следующий слой/обработчик
    response: Response = await call_next(request)

    # --- код ПОСЛЕ обработчика (на выходе) ---
    process_time = time.perf_counter() - start_time
    # добавляем заголовок ко всем ответам
    response.headers["X-Process-Time"] = f"{process_time:.4f}"
    return response
```

Важные нюансы декоратора `@app.middleware("http")`:
- нельзя «прочитать» тело запроса наивно — `await request.body()` буферизирует поток, и если потом обработчик тоже захочет тело, нужно аккуратно работать (Starlette кеширует тело при первом чтении в `request._body`, но полагаться на это в стримах опасно);
- `call_next` под капотом запускает остаток приложения в отдельной задаче и стримит ответ; исключения из обработчика всплывают сюда;
- не видит websocket-соединения и lifespan.

### 2. Подключение готовых middleware через add_middleware

```python
from fastapi import FastAPI
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Перенаправляет все http-запросы на https (и ws -> wss)
app.add_middleware(HTTPSRedirectMiddleware)

# Пропускает запросы только с разрешённым заголовком Host
# Защита от HTTP Host Header атак
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["example.com", "*.example.com", "localhost"],
)

# Сжимает ответы gzip, если клиент прислал Accept-Encoding: gzip
# minimum_size — не сжимать ответы меньше N байт (нет смысла)
app.add_middleware(GZipMiddleware, minimum_size=1000, compresslevel=5)

# CORS — разрешает кросс-доменные запросы из браузера
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://frontend.example.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

`fastapi.middleware.*` — это просто реэкспорт `starlette.middleware.*`. Можно импортировать и из Starlette напрямую.

### 3. ProxyHeadersMiddleware (работа за прокси)

Когда приложение стоит за nginx / traefik / load balancer, клиентский IP и схема (http/https) приходят в заголовках `X-Forwarded-For`, `X-Forwarded-Proto`. Без обработки этих заголовков `request.client.host` покажет IP прокси, а `request.url.scheme` — `http`.

Рекомендуемый способ — флаг uvicorn:

```bash
# доверяем заголовкам только от перечисленных прокси
uvicorn app:app --proxy-headers --forwarded-allow-ips="10.0.0.1,127.0.0.1"
```

Под капотом uvicorn оборачивает приложение в `uvicorn.middleware.proxy_headers.ProxyHeadersMiddleware`. Можно подключить и вручную как ASGI middleware:

```python
from uvicorn.middleware.proxy_headers import ProxyHeadersMiddleware

# trusted_hosts — каким адресам прокси доверять (важно для безопасности!)
app.add_middleware(ProxyHeadersMiddleware, trusted_hosts="127.0.0.1")
```

ВАЖНО: никогда не доверяйте `X-Forwarded-*` от всех (`*`) в продакшене без проверки — иначе клиент сможет подделать свой IP/схему.

### 4. Порядок выполнения middleware («луковица»)

```python
app = FastAPI()

app.add_middleware(MiddlewareA)  # добавлен первым
app.add_middleware(MiddlewareB)  # добавлен вторым
app.add_middleware(MiddlewareC)  # добавлен последним
```

Стек выполнения для входящего запроса:

```
Запрос →  C  →  B  →  A  →  обработчик пути
Ответ  ←  C  ←  B  ←  A  ←  обработчик пути
```

То есть: **последний добавленный оборачивает всех остальных**, его код «до» выполняется раньше всех, а код «после» — позже всех. Это как вложенные функции: `C(B(A(app)))`.

Практический вывод: если `HTTPSRedirectMiddleware` должен срабатывать раньше CORS — добавляйте его позже (он окажется снаружи). Логирующий middleware, который должен видеть финальный ответ, тоже добавляют последним.

### 5. Чистый ASGI middleware (scope / receive / send)

Самый мощный и универсальный вариант. Работает для http, websocket и lifespan, не привязан к Starlette `Request`/`Response`.

Структура: класс, принимающий следующее приложение `app` и реализующий `__call__(scope, receive, send)`.

```python
from starlette.types import ASGIApp, Scope, Receive, Send, Message
import time


class ProcessTimeASGIMiddleware:
    """Чистый ASGI middleware: добавляет заголовок X-Process-Time
    только для HTTP-ответов, не трогая websocket и lifespan."""

    def __init__(self, app: ASGIApp) -> None:
        # сохраняем «следующее» приложение в стеке
        self.app = app

    async def __call__(
        self, scope: Scope, receive: Receive, send: Send
    ) -> None:
        # пропускаем всё, что не http (websocket, lifespan)
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        start_time = time.perf_counter()

        # оборачиваем оригинальный send, чтобы перехватить старт ответа
        async def send_wrapper(message: Message) -> None:
            if message["type"] == "http.response.start":
                process_time = time.perf_counter() - start_time
                # заголовки в ASGI — список пар bytes
                headers = message.setdefault("headers", [])
                headers.append(
                    (b"x-process-time", f"{process_time:.4f}".encode())
                )
            await send(message)

        # вызываем следующее приложение с подменённым send
        await self.app(scope, receive, send_wrapper)


app.add_middleware(ProcessTimeASGIMiddleware)
```

Ключевые моменты чистого ASGI middleware:
- проверяйте `scope["type"]` и пропускайте чужие типы соединений без изменений;
- чтобы изменить ответ — оборачивайте `send`; чтобы изменить запрос — оборачивайте `receive`;
- заголовки — это список кортежей `(bytes, bytes)` в нижнем регистре;
- событие ответа `http.response.start` несёт `status` и `headers`; тело идёт в `http.response.body`.

### 6. Перехват тела запроса/ответа через receive/send

Иногда нужно прочитать/модифицировать тело. Пример перехвата тела ответа:

```python
class CaptureResponseBodyMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        body_chunks: list[bytes] = []

        async def send_wrapper(message):
            if message["type"] == "http.response.body":
                body_chunks.append(message.get("body", b""))
            await send(message)

        await self.app(scope, receive, send_wrapper)
        full_body = b"".join(body_chunks)
        # здесь можно залогировать full_body (осторожно с большими телами!)
```

### 7. Когда middleware, а когда зависимость (Depends)

| Критерий | Middleware | Зависимость (`Depends`) |
|---|---|---|
| Область действия | Глобально, на каждый запрос | Точечно: на эндпоинт/роутер/всё приложение |
| Доступ к параметрам пути/тела | Нет (только сырой scope) | Да, типизированный доступ |
| Возврат HTTP-ошибки | Через ручную отправку Response | `raise HTTPException` |
| Документация OpenAPI | Не отражается | Отражается (security schemes и т.д.) |
| Доступ к результату обработчика | Да (видит Response) | Нет (выполняется до обработчика) |
| Работа с websocket/lifespan | ASGI middleware — да | Зависимости — да (в websocket тоже) |
| Тестируемость в изоляции | Сложнее | Проще, переопределяется через `dependency_overrides` |

Правило: **аутентификацию, авторизацию, валидацию конкретного запроса — делайте через зависимости** (они типизированы, попадают в документацию, дают `HTTPException`). **Кросс-резальные задачи на каждый запрос** (логирование, метрики, заголовки, сжатие, CORS) — через middleware.

## Полный рабочий пример

```python
import time
import logging
from contextlib import asynccontextmanager
from typing import Annotated

from fastapi import FastAPI, Request, Depends, HTTPException
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.cors import CORSMiddleware
from starlette.types import ASGIApp, Scope, Receive, Send, Message

logger = logging.getLogger("app")
logging.basicConfig(level=logging.INFO)


# --- Современный lifespan вместо on_event ---
@asynccontextmanager
async def lifespan(app: FastAPI):
    logger.info("Старт приложения")
    yield
    logger.info("Остановка приложения")


app = FastAPI(lifespan=lifespan, title="Middleware demo")


# --- 1. Собственный чистый ASGI middleware: request id + время ---
class RequestContextMiddleware:
    def __init__(self, app: ASGIApp) -> None:
        self.app = app

    async def __call__(
        self, scope: Scope, receive: Receive, send: Send
    ) -> None:
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        request_id = str(time.time_ns())
        start = time.perf_counter()

        async def send_wrapper(message: Message) -> None:
            if message["type"] == "http.response.start":
                took = time.perf_counter() - start
                headers = message.setdefault("headers", [])
                headers.append((b"x-request-id", request_id.encode()))
                headers.append(
                    (b"x-process-time", f"{took:.4f}".encode())
                )
            await send(message)

        logger.info("Запрос %s %s id=%s",
                    scope["method"], scope["path"], request_id)
        await self.app(scope, receive, send_wrapper)


# --- 2. HTTP-middleware через декоратор для логирования статуса ---
@app.middleware("http")
async def log_response_status(request: Request, call_next):
    response = await call_next(request)
    logger.info("Ответ %s -> %s", request.url.path, response.status_code)
    return response


# Порядок: добавленный позже — снаружи (выполняется раньше на входе).
# Сначала Gzip и TrustedHost (внутренние), затем наш контекст (внешний).
app.add_middleware(GZipMiddleware, minimum_size=500)
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["localhost", "127.0.0.1", "testserver"],
)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)
app.add_middleware(RequestContextMiddleware)  # самый внешний


# --- Зависимость для авторизации (а не middleware!) ---
async def verify_token(x_token: Annotated[str | None, Depends(lambda: None)] = None):
    # упрощённо: в реале берём из заголовка через Header(...)
    ...


@app.get("/")
async def root():
    return {"message": "Привет! Это демо middleware."}


@app.get("/big")
async def big_payload():
    # большой ответ, чтобы сработал GZip
    return {"data": ["элемент"] * 1000}
```

## Частые вопросы на собеседовании (Q/A)

**1. В чём разница между `@app.middleware("http")` и `app.add_middleware(...)`?**
Декоратор `@app.middleware("http")` — обёртка Starlette `BaseHTTPMiddleware`, работает только для HTTP, оперирует `Request`/`Response`, удобен для простых случаев. `add_middleware` добавляет чистый ASGI middleware, работающий для http, websocket и lifespan, на уровне `scope/receive/send`. Декоратор под капотом тоже добавляет middleware в тот же стек.

**2. В каком порядке выполняются middleware?**
Как «луковица»: middleware, добавленный последним, оборачивает остальных — его код «до» выполняется первым, «после» — последним. Выполнение в обратном порядке добавления. Эквивалент `C(B(A(app)))`.

**3. Что такое ASGI и зачем три аргумента scope/receive/send?**
ASGI — асинхронный интерфейс между сервером и приложением. `scope` — метаданные соединения, `receive` — получение событий (тело, дисконнект), `send` — отправка событий (старт и тело ответа). Middleware оборачивает приложение, реализуя тот же интерфейс.

**4. Как написать собственный ASGI middleware?**
Класс с `__init__(self, app)` (сохраняем следующее приложение) и `async def __call__(self, scope, receive, send)`. Проверяем `scope["type"]`, при необходимости оборачиваем `send`/`receive`, затем вызываем `await self.app(scope, receive, send)`.

**5. Как добавить заголовок ко всем ответам в чистом ASGI middleware?**
Обернуть `send`: при `message["type"] == "http.response.start"` дописать пару `(b"имя", b"значение")` в `message["headers"]`, затем вызвать оригинальный `send`.

**6. Зачем нужен TrustedHostMiddleware?**
Защита от атак на заголовок Host (Host Header Injection): пропускает запросы только с разрешёнными значениями `Host`, иначе возвращает 400. Важно при генерации абсолютных URL и редиректов.

**7. Что делает GZipMiddleware и когда он не сжимает?**
Сжимает тело ответа gzip, если клиент прислал `Accept-Encoding: gzip`. Не сжимает ответы меньше `minimum_size` и не сжимает уже сжатые/стримовые типы по необходимости. Снижает трафик ценой CPU.

**8. Как корректно получить реальный IP клиента за прокси?**
Включить обработку `X-Forwarded-*`: запустить uvicorn с `--proxy-headers --forwarded-allow-ips=...` (ProxyHeadersMiddleware). Доверять заголовкам только от известных прокси, иначе клиент подделает IP/схему.

**9. Когда выбрать middleware, а когда зависимость?**
Зависимость — для аутентификации/авторизации/валидации конкретного эндпоинта (типизация, OpenAPI, `HTTPException`, переопределение в тестах). Middleware — для глобальных кросс-резальных задач (логирование, метрики, заголовки, сжатие, CORS, request-id).

**10. Почему чтение тела запроса в middleware опасно?**
Тело запроса — одноразовый поток. Если middleware вычитает его наивно, обработчик может не получить тело. Нужно либо кешировать и подменять `receive`, либо пользоваться тем, что Starlette кеширует `request._body` при первом `await request.body()`.

**11. Влияет ли middleware на производительность?**
Да: каждый middleware добавляет накладные расходы на каждый запрос. `BaseHTTPMiddleware` (декоратор) исторически дороже из-за работы со стримами/задачами; чистый ASGI middleware легче. Не плодите лишние слои.

**12. Можно ли в middleware обработать исключение из обработчика?**
В чистом ASGI middleware можно обернуть `await self.app(...)` в try/except и отправить свой ответ через `send`. В декораторе исключение всплывает в `call_next`. Но для типовой обработки ошибок лучше использовать `app.add_exception_handler`.

## Подводные камни (gotchas)

- **Порядок добавления интуитивно обратный.** Последний `add_middleware` — самый внешний. Частая ошибка — ожидать прямой порядок.
- **`@app.middleware("http")` не видит websocket и lifespan.** Для них нужен ASGI middleware.
- **Чтение тела запроса ломает обработчик**, если не подменить `receive` обратно. Не вычитывайте тело без необходимости.
- **GZip + стриминг.** Сжатие стримовых/SSE-ответов может ломать клиента или буферизовать поток. Проверяйте.
- **Заголовки в ASGI — bytes в нижнем регистре.** Передача `str` или верхнего регистра приведёт к ошибкам.
- **CORS-middleware должен корректно располагаться в стеке** относительно редиректов; ошибки CORS часто из-за того, что редирект (HTTPS/TrustedHost) срабатывает раньше и теряет CORS-заголовки.
- **ProxyHeaders с `*`** — серьёзная дыра безопасности: клиент подделает IP. Указывайте конкретные адреса прокси.
- **Исключения в middleware** могут не дойти до глобальных exception handlers, если их перехватить раньше — будьте аккуратны с try/except.
- **BaseHTTPMiddleware и фоновые задачи / контекст.** Из-за запуска в отдельной задаче возможны нюансы с contextvars; чистый ASGI middleware предсказуемее.

## Лучшие практики

- Для глобальных задач (логи, метрики, заголовки, request-id, сжатие, CORS) — middleware; для авторизации/валидации эндпоинта — зависимости.
- Для нетривиальной логики и работы со всеми типами соединений пишите **чистый ASGI middleware**, а не декоратор.
- Минимизируйте число middleware — каждый стоит времени на каждый запрос.
- Всегда проверяйте `scope["type"]` и пропускайте «чужие» типы без изменений.
- Не читайте тело запроса в middleware без крайней необходимости; если читаете — корректно подменяйте `receive`.
- За прокси включайте `--proxy-headers` с явным списком доверенных адресов.
- Документируйте порядок добавления middleware комментариями — он не очевиден.
- Для обработки ошибок используйте `add_exception_handler`, а не try/except в каждом middleware.

## Шпаргалка

```python
# Готовые middleware
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(GZipMiddleware, minimum_size=1000)
app.add_middleware(HTTPSRedirectMiddleware)
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["example.com"])
app.add_middleware(CORSMiddleware, allow_origins=["*"])

# Декоратор HTTP-middleware
@app.middleware("http")
async def m(request, call_next):
    resp = await call_next(request)        # вызвать следующий слой
    resp.headers["X-Custom"] = "1"
    return resp

# Чистый ASGI middleware (шаблон)
class MyMiddleware:
    def __init__(self, app): self.app = app
    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send); return
        async def send_wrapper(message):
            if message["type"] == "http.response.start":
                message.setdefault("headers", []).append((b"x-h", b"v"))
            await send(message)
        await self.app(scope, receive, send_wrapper)

app.add_middleware(MyMiddleware)

# Порядок: последний add_middleware -> самый внешний (выполняется первым)
# C(B(A(app)))  при add_middleware(A); add_middleware(B); add_middleware(C)

# За прокси:
# uvicorn app:app --proxy-headers --forwarded-allow-ips="10.0.0.1"
```
