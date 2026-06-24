# FastAPI — Middleware и CORS — конспект и вопросы

## О чём раздел

Middleware — это слой, который оборачивает обработку каждого HTTP-запроса: код выполняется **до** того, как запрос дойдёт до обработчика пути (path operation), и **после** того, как обработчик вернул ответ, но **до** отправки ответа клиенту. С помощью middleware удобно реализовать сквозные задачи: логирование, замер времени обработки, добавление заголовков, обработку ошибок на верхнем уровне, аутентификацию по токену на уровне всего приложения.

CORS (Cross-Origin Resource Sharing) — механизм браузера, позволяющий веб-странице с одного источника (origin) делать запросы к API на другом источнике. По умолчанию браузер блокирует такие запросы из-за Same-Origin Policy. `CORSMiddleware` — это готовое middleware из Starlette, которое добавляет нужные заголовки и обрабатывает preflight-запросы.

FastAPI построен поверх Starlette, поэтому всё middleware работает на уровне ASGI.

## Ключевые концепции

- **ASGI middleware** — слой между ASGI-сервером (uvicorn) и приложением. Принимает `scope`, `receive`, `send`.
- **`@app.middleware("http")`** — декоратор для простого HTTP-middleware в виде async-функции.
- **`call_next`** — функция, которую middleware вызывает, чтобы передать запрос дальше по цепочке и получить `Response`.
- **Порядок выполнения** — middleware, добавленные позже, выполняются «снаружи» (первыми на входе, последними на выходе). Работает как стек/«луковица».
- **`BaseHTTPMiddleware`** — базовый класс для middleware в виде класса (из Starlette).
- **Same-Origin Policy** — origin = схема + хост + порт. Если хоть что-то отличается — это другой origin.
- **Preflight (предварительный) запрос** — браузер отправляет `OPTIONS` перед «сложным» запросом, чтобы спросить разрешение у сервера.
- **`CORSMiddleware`** — настраивается через `allow_origins`, `allow_methods`, `allow_headers`, `allow_credentials`.

## Подробный разбор с примерами кода

### 1. Простое HTTP-middleware через декоратор

```python
import time
from typing import Annotated
from fastapi import FastAPI, Request, Response
from starlette.middleware.base import RequestResponseEndpoint

app = FastAPI()


@app.middleware("http")
async def add_process_time_header(
    request: Request,
    call_next: RequestResponseEndpoint,  # функция вызова следующего слоя
) -> Response:
    # --- КОД ДО обработки запроса ---
    start_time = time.perf_counter()

    # Передаём управление дальше: следующее middleware или сам обработчик пути.
    # Возвращается готовый Response.
    response = await call_next(request)

    # --- КОД ПОСЛЕ получения ответа, но ДО отправки клиенту ---
    process_time = time.perf_counter() - start_time
    # Добавляем кастомный заголовок с временем обработки в секундах
    response.headers["X-Process-Time"] = f"{process_time:.4f}"
    return response
```

Ключевые моменты:
- Функция обязана быть `async`.
- Аргументы строго: `request: Request` и `call_next`.
- Обязательно `return` результата `call_next`, иначе клиент не получит ответ.
- Нельзя «прочитать тело запроса» наивно — это потребляет поток. Для модификации тела нужны аккуратные приёмы (см. gotchas).

### 2. Несколько middleware и порядок выполнения

```python
@app.middleware("http")
async def middleware_one(request: Request, call_next):
    print("ВХОД one")
    response = await call_next(request)
    print("ВЫХОД one")
    return response


@app.middleware("http")
async def middleware_two(request: Request, call_next):
    print("ВХОД two")
    response = await call_next(request)
    print("ВЫХОД two")
    return response
```

Если зарегистрированы `one`, затем `two`, то для одного запроса вывод будет:

```
ВХОД two       # последний добавленный — внешний слой, выполняется первым на входе
ВХОД one
... обработчик пути ...
ВЫХОД one
ВЫХОД two       # внешний слой завершается последним
```

Это модель «луковицы»: запрос проходит слои снаружи внутрь, ответ — изнутри наружу. Тот, кто добавлен последним, оборачивает всех.

### 3. BaseHTTPMiddleware (middleware в виде класса)

Удобно, когда нужна конфигурация, состояние или переиспользование.

```python
from starlette.middleware.base import BaseHTTPMiddleware, RequestResponseEndpoint
from starlette.types import ASGIApp
from fastapi import Request, Response


class ProcessTimeMiddleware(BaseHTTPMiddleware):
    def __init__(self, app: ASGIApp, header_name: str = "X-Process-Time") -> None:
        super().__init__(app)
        self.header_name = header_name

    async def dispatch(
        self, request: Request, call_next: RequestResponseEndpoint
    ) -> Response:
        start = time.perf_counter()
        response = await call_next(request)
        response.headers[self.header_name] = f"{time.perf_counter() - start:.4f}"
        return response


# Регистрация (через add_middleware, а не декоратор):
app.add_middleware(ProcessTimeMiddleware, header_name="X-Process-Time")
```

`add_middleware` принимает класс и kwargs для его конструктора. Важно: `add_middleware` тоже добавляет «снаружи» — последний добавленный самый внешний.

### 4. Чистое ASGI middleware (низкоуровневое)

Самый производительный вариант, без накладных расходов `BaseHTTPMiddleware`:

```python
from starlette.types import ASGIApp, Receive, Scope, Send


class PureASGIMiddleware:
    def __init__(self, app: ASGIApp) -> None:
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] != "http":
            # Пропускаем websocket / lifespan без изменений
            await self.app(scope, receive, send)
            return

        async def send_wrapper(message) -> None:
            if message["type"] == "http.response.start":
                headers = message.setdefault("headers", [])
                headers.append((b"x-custom", b"value"))
            await send(message)

        await self.app(scope, receive, send_wrapper)


app.add_middleware(PureASGIMiddleware)
```

### 5. Отличие middleware от зависимостей (Depends)

| Признак | Middleware | Зависимости (`Depends`) |
|---|---|---|
| Область действия | Все запросы приложения | Конкретные эндпоинты/роутеры |
| Доступ к параметрам пути | Нет (только `Request`) | Да (валидация, типы) |
| Возврат значения в обработчик | Нет | Да (внедряется как аргумент) |
| Модификация ответа | Да (заголовки и т.п.) | Косвенно (yield + код после) |
| Видимость в OpenAPI/Swagger | Нет | Да (security, параметры) |
| Гранулярность | Грубая | Точная |
| Обработка исключений из обработчика | Видит как ответ | Не всегда |

Правило: сквозные задачи на всё приложение — middleware; логика, привязанная к конкретным маршрутам и нуждающаяся в валидации/документации, — зависимости.

### 6. CORS: что это и зачем

Браузер из соображений безопасности применяет **Same-Origin Policy**: JS со страницы `https://app.example.com` не может читать ответ от `https://api.example.com`, если сервер явно не разрешил. Origin = `схема://хост:порт`. `http` и `https`, разные порты, разные поддомены — это разные origin.

**Preflight**: для «непростых» запросов (методы PUT/DELETE/PATCH, кастомные заголовки, `Content-Type: application/json`) браузер сначала шлёт `OPTIONS` с заголовками `Origin`, `Access-Control-Request-Method`, `Access-Control-Request-Headers`. Сервер должен ответить нужными `Access-Control-Allow-*`. Только потом браузер отправит основной запрос.

### 7. Настройка CORSMiddleware

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

origins = [
    "https://app.example.com",
    "http://localhost:5173",  # фронтенд на Vite в разработке
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,            # какие origin разрешены
    allow_credentials=True,           # разрешить куки/Authorization
    allow_methods=["*"],              # разрешённые HTTP-методы
    allow_headers=["*"],              # разрешённые заголовки запроса
    expose_headers=["X-Process-Time"],# какие заголовки ответа видны JS
    max_age=600,                      # сколько кэшировать preflight (сек)
)
```

Важная деталь безопасности: **`allow_origins=["*"]` несовместимо с `allow_credentials=True`**. Спецификация CORS запрещает отдавать `Access-Control-Allow-Origin: *` вместе с куками. Если нужны credentials — перечисляйте origin явно (или используйте `allow_origin_regex`).

## Полный рабочий пример

```python
import time
from typing import Annotated

from fastapi import Depends, FastAPI, Request, Response
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from starlette.middleware.base import RequestResponseEndpoint

app = FastAPI(title="Middleware и CORS — демо")

# --- CORS (добавляем ПЕРВЫМ, чтобы он был самым внешним и обрабатывал preflight) ---
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173", "https://app.example.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type"],
    expose_headers=["X-Process-Time"],
    max_age=600,
)


# --- Замер времени обработки ---
@app.middleware("http")
async def add_process_time_header(
    request: Request, call_next: RequestResponseEndpoint
) -> Response:
    start = time.perf_counter()
    response = await call_next(request)
    elapsed = time.perf_counter() - start
    response.headers["X-Process-Time"] = f"{elapsed:.4f}"
    return response


# --- Простое логирование ---
@app.middleware("http")
async def log_requests(
    request: Request, call_next: RequestResponseEndpoint
) -> Response:
    print(f"-> {request.method} {request.url.path}")
    response = await call_next(request)
    print(f"<- {response.status_code} {request.url.path}")
    return response


class Item(BaseModel):
    name: str
    price: float


# Пример зависимости для сравнения с middleware
async def get_request_id(request: Request) -> str:
    return request.headers.get("X-Request-ID", "n/a")


@app.get("/items/{item_id}")
async def read_item(
    item_id: int,
    request_id: Annotated[str, Depends(get_request_id)],
) -> dict:
    return {"item_id": item_id, "request_id": request_id}


@app.post("/items")
async def create_item(item: Item) -> Item:
    return item
```

Порядок слоёв при запросе `GET /items/1`:
`CORS -> log_requests -> add_process_time_header -> Depends(get_request_id) -> read_item`
и обратно при формировании ответа.

## Частые вопросы на собеседовании (Q/A)

**1. Что такое middleware в FastAPI?**
Функция/класс, оборачивающий обработку каждого запроса: выполняет код до и после обработчика пути. Реализуется на уровне ASGI (через Starlette).

**2. Какова сигнатура HTTP-middleware через декоратор?**
`async def m(request: Request, call_next) -> Response`. Нужно вызвать `await call_next(request)` и вернуть полученный `Response`.

**3. Что делает `call_next`?**
Передаёт запрос дальше по цепочке (следующее middleware или обработчик пути) и возвращает готовый `Response`. Без её вызова запрос не будет обработан.

**4. В каком порядке выполняются несколько middleware?**
Модель «луковицы»: последний добавленный — самый внешний, выполняется первым на входе и последним на выходе.

**5. Чем middleware отличается от зависимости?**
Middleware действует на все запросы, не имеет доступа к валидированным параметрам пути и не документируется в OpenAPI. Зависимости точечные, типизированные, внедряются в обработчик и видны в Swagger.

**6. Когда выбрать `BaseHTTPMiddleware`, а когда чистое ASGI?**
`BaseHTTPMiddleware` проще и удобнее для большинства задач. Чистое ASGI — когда важна производительность или нужен доступ к стримингу/телу на низком уровне; у `BaseHTTPMiddleware` есть накладные расходы и нюансы со стримингом.

**7. Что такое Same-Origin Policy и origin?**
Политика браузера, запрещающая чтение ответа с другого origin. Origin = схема + хост + порт; различие в любом из трёх — другой origin.

**8. Что такое preflight-запрос?**
Предварительный `OPTIONS`-запрос браузера перед «сложным» кросс-доменным запросом, чтобы узнать у сервера, разрешена ли операция (метод, заголовки, credentials).

**9. Какие основные параметры у `CORSMiddleware`?**
`allow_origins`, `allow_methods`, `allow_headers`, `allow_credentials`, плюс `expose_headers`, `max_age`, `allow_origin_regex`.

**10. Почему нельзя `allow_origins=["*"]` вместе с `allow_credentials=True`?**
Спецификация CORS запрещает отдавать `Access-Control-Allow-Origin: *` при включённых credentials; браузер отклонит ответ. Нужно перечислять origin явно.

**11. Почему мой кастомный заголовок ответа не виден в JS на фронте?**
Браузер скрывает нестандартные заголовки ответа при CORS. Их нужно перечислить в `expose_headers`.

**12. Влияет ли порядок добавления CORS относительно других middleware?**
Да. CORS лучше добавлять так, чтобы он был внешним слоем (обычно — добавлять первым через `add_middleware`), иначе исключения или другие middleware могут «съесть» preflight и заголовки не добавятся.

**13. Можно ли в middleware прервать запрос и сразу вернуть ответ?**
Да, не вызывая `call_next`, можно вернуть свой `Response` (например, 401). Но для авторизации с учётом маршрутов чаще берут зависимости.

**14. Как middleware взаимодействует с исключениями?**
Необработанные исключения из обработчика проходят через `ExceptionMiddleware`. `BaseHTTPMiddleware`-код после `call_next` может не выполниться при определённых ошибках — для гарантированной очистки лучше использовать зависимости с `yield` или обработку внутри.

## Подводные камни (gotchas)

- **Забыли `return response`** — клиент получит пустой/зависший ответ.
- **`allow_origins=["*"]` + `allow_credentials=True`** — не работает по спецификации; перечисляйте origin.
- **Кастомные заголовки не видны на фронте** — добавьте их в `expose_headers`.
- **Чтение тела запроса в middleware** — наивный `await request.body()` потребляет поток, и обработчик получит пустое тело; нужен аккуратный re-injection или использование зависимостей.
- **Порядок CORS** — если CORS не внешний слой, preflight/заголовки могут теряться, особенно при ошибках в других middleware.
- **`BaseHTTPMiddleware` и стриминговые ответы** — может буферизовать или ломать `StreamingResponse`; в сложных случаях берите чистое ASGI middleware.
- **Тяжёлые синхронные операции в async-middleware** блокируют event loop — выносите в `run_in_threadpool`/executor.
- **Origin с/без слеша** — `https://app.example.com` и `https://app.example.com/` различаются; в `allow_origins` слеш в конце не ставят.
- **Локалхост и порт** — `http://localhost:5173` и `http://127.0.0.1:5173` это разные origin.

## Лучшие практики

- Держите middleware тонкими и быстрыми — они выполняются на каждом запросе.
- CORS добавляйте явным списком origin; `*` — только для публичных API без credentials.
- Для авторизации/валидации, привязанной к маршрутам, используйте зависимости, а не middleware.
- Используйте `expose_headers` для всех кастомных заголовков, нужных фронтенду.
- Конфигурацию origin берите из переменных окружения/настроек (`pydantic-settings`), не хардкодьте.
- Для производительности предпочитайте чистое ASGI middleware при высокой нагрузке.
- Логируйте корреляционный ID запроса в middleware для трассировки.

## Шпаргалка

```python
# Декораторное HTTP-middleware
@app.middleware("http")
async def m(request: Request, call_next) -> Response:
    response = await call_next(request)
    response.headers["X-Process-Time"] = "..."
    return response

# Класс
app.add_middleware(MyMiddleware, option="value")   # последний добавленный — внешний

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],  # НЕ "*" если credentials
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
    expose_headers=["X-Process-Time"],
    max_age=600,
)
```

- Порядок: последний `add_middleware` = внешний слой = первый на входе, последний на выходе.
- Preflight = `OPTIONS`; обрабатывается `CORSMiddleware` автоматически.
- `*` + credentials = запрещено спецификацией.
