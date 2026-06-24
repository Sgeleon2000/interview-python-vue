# FastAPI Advanced — События жизненного цикла (lifespan) — конспект и вопросы

## О чём раздел

События жизненного цикла (lifecycle / lifespan events) — это код, который выполняется **один раз** при запуске приложения (startup) и **один раз** при его остановке (shutdown). Это НЕ «на каждый запрос», а именно на весь срок жизни процесса.

Зачем это нужно:
- **Открыть дорогие ресурсы один раз** и переиспользовать их между запросами: пул соединений к БД, Redis-клиент, HTTP-клиент (`httpx.AsyncClient`), очередь сообщений.
- **Загрузить тяжёлые объекты** в память: ML-модель, эмбеддинги, справочники, кеш.
- **Прогреть кеш** / выполнить миграции / зарегистрировать сервис.
- **Корректно освободить ресурсы** при остановке: закрыть соединения, сбросить буферы, дождаться завершения фоновых задач.

Современный способ — **`lifespan` через `@asynccontextmanager`**, который передаётся в конструктор `FastAPI(lifespan=...)`. Старые декораторы `@app.on_event("startup")` и `@app.on_event("shutdown")` считаются **устаревшими (deprecated)**.

Механизм основан на ASGI-событии типа `lifespan`, которое сервер (uvicorn) посылает приложению при старте и остановке.

## Ключевые концепции

- **`lifespan` — асинхронный контекстный менеджер.** Функция с `@asynccontextmanager`, принимающая `app: FastAPI` и содержащая единственный `yield`.
  - Код **до `yield`** = startup (выполняется до приёма первого запроса).
  - Код **после `yield`** = shutdown (выполняется при остановке).
- **Передача в приложение:** `app = FastAPI(lifespan=lifespan)`.
- **`app.state`** — место для хранения общих ресурсов (`app.state.db_pool = ...`). Доступно в запросах через `request.app.state`.
- **Можно «отдавать» состояние через yield:** `yield {"ключ": значение}` — это словарь попадает в `request.state` для каждого запроса (типобезопаснее, чем `app.state`).
- **Гарантии:** startup выполняется ПОЛНОСТЬЮ до первого запроса; shutdown — после завершения обработки (при graceful shutdown).
- **Отличие от middleware/зависимостей:** lifespan — один раз на процесс; middleware и зависимости — на каждый запрос.
- **`@app.on_event` устарел**, потому что lifespan мощнее (общий контекст между startup и shutdown через замыкание/контекст-менеджер, чище управление ресурсами, корректная работа с группами ресурсов и ошибками).

## Подробный разбор с примерами кода

### 1. Базовый lifespan

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI


@asynccontextmanager
async def lifespan(app: FastAPI):
    # === STARTUP: код до yield ===
    print("Приложение запускается: инициализация ресурсов")
    # ... тут открываем соединения, грузим модели ...

    yield  # <-- здесь приложение работает и обрабатывает запросы

    # === SHUTDOWN: код после yield ===
    print("Приложение останавливается: освобождение ресурсов")
    # ... тут закрываем соединения ...


app = FastAPI(lifespan=lifespan)


@app.get("/")
async def root():
    return {"status": "ok"}
```

Поток: uvicorn запускает приложение → выполняется код до `yield` → сервер начинает принимать запросы → при остановке (Ctrl+C / SIGTERM) → выполняется код после `yield`.

### 2. Хранение состояния: app.state vs yield-словарь

**Вариант A — через `app.state`** (классический):

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.cache = {}            # кладём ресурс в state
    app.state.startup_done = True
    yield
    app.state.cache.clear()         # чистим при остановке


app = FastAPI(lifespan=lifespan)


@app.get("/cache-size")
async def cache_size(request: Request):
    # доступ через request.app.state
    return {"size": len(request.app.state.cache)}
```

**Вариант B — через `yield {...}`** (рекомендуется, типобезопаснее):

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    cache: dict = {}
    # отдаём словарь состояния — попадёт в request.state
    yield {"cache": cache}
    cache.clear()


app = FastAPI(lifespan=lifespan)


@app.get("/cache")
async def get_cache(request: Request):
    return {"size": len(request.state.cache)}
```

`request.state` (а не `request.app.state`) удобнее, потому что состояние «локально» для запроса и не размазано по глобальному объекту приложения.

### 3. Типичный кейс: пул соединений к БД

```python
from contextlib import asynccontextmanager
from typing import Annotated, AsyncIterator

import asyncpg
from fastapi import FastAPI, Depends, Request


@asynccontextmanager
async def lifespan(app: FastAPI):
    # STARTUP: создаём пул соединений один раз на всё приложение
    pool = await asyncpg.create_pool(
        dsn="postgresql://user:pass@localhost/db",
        min_size=5,
        max_size=20,
    )
    app.state.db_pool = pool
    print("Пул соединений к БД создан")

    yield  # приложение работает

    # SHUTDOWN: закрываем пул, чтобы не оставить висящие соединения
    await pool.close()
    print("Пул соединений к БД закрыт")


app = FastAPI(lifespan=lifespan)


# Зависимость, которая выдаёт соединение из пула на время запроса
async def get_db(request: Request) -> AsyncIterator[asyncpg.Connection]:
    pool: asyncpg.Pool = request.app.state.db_pool
    async with pool.acquire() as connection:
        yield connection


@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    db: Annotated[asyncpg.Connection, Depends(get_db)],
):
    row = await db.fetchrow("SELECT id, name FROM users WHERE id = $1", user_id)
    return dict(row) if row else {"error": "not found"}
```

Здесь хорошо видно разделение ответственности: **lifespan** создаёт/закрывает пул (один раз), **зависимость** выдаёт соединение из пула на каждый запрос.

### 4. Типичный кейс: ML-модель

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Загружаем тяжёлую модель один раз при старте
    import joblib
    app.state.model = joblib.load("model.pkl")
    print("ML-модель загружена")
    yield
    # Освобождаем ссылку (опционально)
    app.state.model = None


app = FastAPI(lifespan=lifespan)


@app.post("/predict")
async def predict(request: Request, features: list[float]):
    model = request.app.state.model
    prediction = model.predict([features])[0]
    return {"prediction": float(prediction)}
```

Загрузка модели на каждый запрос была бы катастрофой по производительности — поэтому делаем это один раз в startup.

### 5. Типичный кейс: HTTP-клиент и Redis

```python
import httpx
import redis.asyncio as redis


@asynccontextmanager
async def lifespan(app: FastAPI):
    # переиспользуемый HTTP-клиент (важно для keep-alive пула соединений)
    app.state.http = httpx.AsyncClient(timeout=10.0)
    # асинхронный Redis-клиент
    app.state.redis = redis.from_url("redis://localhost:6379")
    yield
    # закрываем оба клиента
    await app.state.http.aclose()
    await app.state.redis.aclose()


app = FastAPI(lifespan=lifespan)


@app.get("/proxy")
async def proxy(request: Request):
    client: httpx.AsyncClient = request.app.state.http
    resp = await client.get("https://httpbin.org/get")
    return resp.json()
```

Создавать `httpx.AsyncClient` на каждый запрос — антипаттерн: теряется пул keep-alive соединений. Создаём один раз в lifespan.

### 6. Управление несколькими ресурсами и обработка ошибок

Контекстный менеджер позволяет аккуратно открывать и закрывать несколько ресурсов, в том числе с гарантией закрытия при ошибке.

```python
from contextlib import asynccontextmanager, AsyncExitStack


@asynccontextmanager
async def lifespan(app: FastAPI):
    async with AsyncExitStack() as stack:
        # каждый ресурс закроется автоматически в обратном порядке
        db = await stack.enter_async_context(open_db_pool())
        cache = await stack.enter_async_context(open_redis())
        app.state.db = db
        app.state.cache = cache
        yield
    # выход из AsyncExitStack корректно закроет всё, даже при исключении
```

Если в startup-коде (до `yield`) возникнет исключение, приложение не стартует, а сервер сообщит об ошибке lifespan.

### 7. Почему `@app.on_event` устарел

```python
# УСТАРЕВШИЙ способ (deprecated) — не использовать в новом коде
@app.on_event("startup")
async def on_startup():
    app.state.db = await create_pool()


@app.on_event("shutdown")
async def on_shutdown():
    await app.state.db.close()
```

Проблемы старого подхода:
- startup и shutdown — **две разные функции**, нет общего локального контекста; приходится складывать всё в глобальный `app.state`.
- труднее гарантировать корректное закрытие ресурса при ошибке инициализации (нет естественного `try/finally` вокруг `yield`);
- хуже сочетается с группами ресурсов и context-manager'ами.

`lifespan` решает это: один менеджер, общий контекст через замыкание, естественный `try/finally`/`AsyncExitStack`, чище код.

## Полный рабочий пример

```python
from contextlib import asynccontextmanager
from typing import Annotated, AsyncIterator

import httpx
from fastapi import FastAPI, Request, Depends


# --- Имитация «тяжёлых» ресурсов ---
class FakeDBPool:
    async def connect(self) -> "FakeDBPool":
        print("DB: пул открыт")
        return self

    async def fetch_user(self, user_id: int) -> dict:
        return {"id": user_id, "name": f"user-{user_id}"}

    async def close(self) -> None:
        print("DB: пул закрыт")


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ===== STARTUP =====
    print("STARTUP: инициализация ресурсов")
    db = await FakeDBPool().connect()
    http = httpx.AsyncClient(timeout=10.0)
    model = {"weights": [0.1, 0.2, 0.3]}  # имитация ML-модели

    # отдаём состояние через yield -> попадёт в request.state
    yield {"db": db, "http": http, "model": model}

    # ===== SHUTDOWN =====
    print("SHUTDOWN: освобождение ресурсов")
    await http.aclose()
    await db.close()


app = FastAPI(lifespan=lifespan, title="Lifespan demo")


# --- Зависимости, достающие ресурсы из request.state ---
def get_db(request: Request) -> FakeDBPool:
    return request.state.db


def get_http(request: Request) -> httpx.AsyncClient:
    return request.state.http


@app.get("/users/{user_id}")
async def read_user(
    user_id: int,
    db: Annotated[FakeDBPool, Depends(get_db)],
):
    return await db.fetch_user(user_id)


@app.get("/external")
async def external(http: Annotated[httpx.AsyncClient, Depends(get_http)]):
    resp = await http.get("https://httpbin.org/uuid")
    return resp.json()


@app.get("/model")
async def model_info(request: Request):
    return {"weights": request.state.model["weights"]}
```

Тест жизненного цикла через `TestClient` (контекстный менеджер запускает lifespan):

```python
from fastapi.testclient import TestClient
from main import app


def test_lifespan_runs():
    # вход в контекст -> startup; выход -> shutdown
    with TestClient(app) as client:
        resp = client.get("/users/1")
        assert resp.status_code == 200
        assert resp.json() == {"id": 1, "name": "user-1"}
    # после выхода из with shutdown уже выполнен
```

ВАЖНО: lifespan запускается только при использовании `TestClient` как контекстного менеджера (`with TestClient(app) as client:`). Если создать `TestClient(app)` и дёргать без `with`, startup/shutdown не выполнятся.

## Частые вопросы на собеседовании (Q/A)

**1. Что такое lifespan и зачем он нужен?**
Это код, выполняемый один раз при старте и один раз при остановке приложения. Нужен для инициализации и освобождения дорогих ресурсов (пулы БД, HTTP-клиенты, ML-модели, кеши), которые переиспользуются между запросами.

**2. Как реализовать lifespan в современном FastAPI?**
Через `@asynccontextmanager`-функцию, принимающую `app`, с единственным `yield`: код до `yield` — startup, после — shutdown. Передать в конструктор: `FastAPI(lifespan=lifespan)`.

**3. Чем lifespan лучше `@app.on_event`?**
`@app.on_event("startup"/"shutdown")` устарел. Lifespan даёт общий локальный контекст между startup и shutdown, естественный `try/finally` вокруг `yield`, удобнее управляет группами ресурсов и ошибками инициализации.

**4. Где хранить ресурсы, созданные в lifespan?**
В `app.state` (доступ через `request.app.state`) или отдавать словарь через `yield {...}`, тогда они попадут в `request.state` — это типобезопаснее и локальнее.

**5. Чем lifespan отличается от middleware и зависимостей?**
Lifespan выполняется один раз на процесс. Middleware и зависимости — на каждый запрос. Lifespan создаёт ресурс, зависимость выдаёт его на время запроса, middleware оборачивает обработку запроса.

**6. Что произойдёт, если в startup-коде возникнет исключение?**
Приложение не стартует — сервер сообщит об ошибке lifespan и не начнёт принимать запросы. Поэтому критичную инициализацию (подключение к БД) логично делать тут: «падать быстро».

**7. Гарантируется ли, что startup завершится до первого запроса?**
Да. Сервер (uvicorn) не начнёт принимать запросы, пока не завершится код до `yield`. И при graceful shutdown код после `yield` выполнится после завершения обработки.

**8. Почему `httpx.AsyncClient` создают в lifespan, а не в эндпоинте?**
Чтобы переиспользовать пул keep-alive соединений. Создание клиента на каждый запрос — антипаттерн: потеря пула, лишние TCP/TLS-рукопожатия, деградация производительности.

**9. Как тестировать lifespan?**
Использовать `TestClient` как контекстный менеджер: `with TestClient(app) as client:` — вход запускает startup, выход — shutdown. Без `with` события жизненного цикла не выполнятся.

**10. Можно ли инициализировать несколько ресурсов и гарантировать их закрытие?**
Да, например через `AsyncExitStack`: ресурсы регистрируются и закрываются в обратном порядке автоматически, в том числе при исключении.

**11. Запускается ли lifespan на каждом воркере?**
Да. При нескольких воркерах (процессах) lifespan выполняется в каждом процессе отдельно — значит, каждый создаёт свой пул/клиент/модель. Учитывайте это по памяти и по числу соединений к БД.

**12. Чем `app.state` отличается от `yield {...}` в lifespan?**
`app.state` — глобальный объект приложения, доступ через `request.app.state`. `yield {...}` помещает значения в `request.state` для каждого запроса — это локальнее и удобнее с точки зрения типизации и изоляции.

## Подводные камни (gotchas)

- **Lifespan не запустится без `with` в TestClient.** `TestClient(app)` без контекстного менеджера пропустит startup/shutdown.
- **`@app.on_event` устарел** — не использовать в новом коде; не смешивать с `lifespan` (FastAPI предупредит, поведение может конфликтовать).
- **Несколько воркеров = несколько lifespan.** Каждый процесс создаёт свои ресурсы; пул БД умножается на число воркеров.
- **Тяжёлая/блокирующая инициализация** в startup задерживает готовность приложения; выносите CPU-bound в executor или делайте ленивую загрузку при необходимости.
- **Незакрытые ресурсы в shutdown** ведут к утечкам соединений; всегда закрывайте то, что открыли (или используйте `AsyncExitStack`).
- **Исключение в startup роняет приложение** — иногда это желаемо («fail fast»), но к нему нужно быть готовым в деплое (health-check, рестарты).
- **Долгий shutdown может быть прерван** SIGKILL'ом при превышении grace-периода оркестратора (k8s `terminationGracePeriodSeconds`).
- **Хранение мутабельного состояния в `app.state`** при нескольких воркерах не разделяется между процессами — это не общий кеш.
- **Забыли `await` при закрытии async-ресурса** (`await client.aclose()`) — ресурс не освободится корректно.

## Лучшие практики

- Используйте `lifespan` через `@asynccontextmanager`, не `@app.on_event`.
- Открывайте дорогие ресурсы (пулы, клиенты, модели) один раз в startup и закрывайте в shutdown.
- Предпочитайте `yield {...}` -> `request.state` для типобезопасного доступа к ресурсам.
- Разделяйте ответственность: lifespan создаёт ресурс, зависимость выдаёт его на запрос.
- Для нескольких ресурсов используйте `AsyncExitStack`, чтобы гарантировать закрытие.
- Делайте инициализацию идемпотентной и устойчивой к ретраям подключения.
- Учитывайте число воркеров при расчёте `max_size` пулов БД (общее число соединений = пул × воркеры).
- Тестируйте жизненный цикл через `with TestClient(app) as client:`.
- Для общего между процессами состояния используйте внешние хранилища (Redis, БД), а не `app.state`.

## Шпаргалка

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request


@asynccontextmanager
async def lifespan(app: FastAPI):
    # STARTUP (до yield): создать ресурсы
    pool = await create_pool()
    client = httpx.AsyncClient()
    # отдать состояние в request.state
    yield {"pool": pool, "client": client}
    # SHUTDOWN (после yield): закрыть ресурсы
    await client.aclose()
    await pool.close()


app = FastAPI(lifespan=lifespan)

# Доступ из запроса:
#   request.state.pool        (если отдали через yield {...})
#   request.app.state.x       (если положили в app.state.x)

# Зависимость поверх ресурса:
def get_pool(request: Request):
    return request.state.pool

# Тестирование жизненного цикла:
from fastapi.testclient import TestClient
with TestClient(app) as client:     # вход -> startup, выход -> shutdown
    client.get("/")

# УСТАРЕЛО (не использовать):
# @app.on_event("startup") / @app.on_event("shutdown")
```
