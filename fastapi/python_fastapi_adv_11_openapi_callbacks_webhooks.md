# FastAPI Advanced — OpenAPI, Callbacks и Webhooks — конспект и вопросы

## О чём раздел

Раздел посвящён тому, как FastAPI генерирует OpenAPI-схему и как её можно расширять и кастомизировать под свои нужды. Сюда входят:

- **Кастомизация OpenAPI-схемы** — функция `custom_openapi`, утилита `get_openapi`, кеширование через `app.openapi_schema`.
- **OpenAPI Callbacks** — документирование внешнего API, который **ваш** сервис вызовет в ответ на событие (например, отправит уведомление на URL, переданный клиентом).
- **OpenAPI Webhooks** — `app.webhooks`, декларация исходящих событий, которые ваш сервис может отправлять подписчикам.
- **Метаданные и теги** — `title`, `description`, `version`, `openapi_tags`, `terms_of_service`, `contact`, `license_info`.
- **Добавление кастомных полей** в схему (`x-` расширения, `x-logo` для ReDoc и т.д.).

Главное, что нужно понимать: в FastAPI OpenAPI-схема — это обычный Python-словарь, который генерируется лениво и кешируется. Вы можете в любой момент перехватить генерацию и дополнить/изменить результат.

## Ключевые концепции

| Концепция | Что это |
|-----------|---------|
| OpenAPI schema | JSON-документ, описывающий все эндпоинты, модели, параметры. Доступен по `/openapi.json`. |
| `app.openapi()` | Метод, возвращающий схему. Внутри вызывает `get_openapi(...)` и кеширует результат. |
| `app.openapi_schema` | Атрибут-кеш. Если не `None`, `app.openapi()` сразу возвращает его, не пересоздавая. |
| `get_openapi(...)` | Функция из `fastapi.openapi.utils`, которая собственно строит словарь схемы. |
| `custom_openapi` | Ваша функция, которую вы присваиваете `app.openapi`, чтобы перехватить генерацию. |
| Callbacks | Описание того, как **ваш** API будет дёргать **внешний** URL в ответ на запрос. Документация, не реальный вызов. |
| Webhooks | Описание исходящих событий (`app.webhooks`), которые сервис рассылает подписчикам. Появились с OpenAPI 3.1. |
| Tags / metadata | Группировка эндпоинтов и описание самого приложения в Swagger UI / ReDoc. |

## Подробный разбор с примерами кода

### 1. Метаданные приложения и теги

Метаданные задаются прямо в конструкторе `FastAPI`. Они влияют только на документацию.

```python
from fastapi import FastAPI

# Описание тегов: имя + текст + ссылка на внешнюю документацию
tags_metadata = [
    {
        "name": "users",
        "description": "Операции с пользователями: создание, поиск, обновление.",
    },
    {
        "name": "items",
        "description": "Управление товарами.",
        "externalDocs": {
            "description": "Внешняя документация по товарам",
            "url": "https://example.com/docs/items",
        },
    },
]

app = FastAPI(
    title="Мой магазин API",
    description="Демонстрационный API для конспекта по FastAPI Advanced.",
    summary="Краткое описание (OpenAPI 3.1).",
    version="2.0.0",
    terms_of_service="https://example.com/terms/",
    contact={
        "name": "Команда поддержки",
        "url": "https://example.com/support",
        "email": "support@example.com",
    },
    license_info={
        "name": "Apache 2.0",
        "url": "https://www.apache.org/licenses/LICENSE-2.0.html",
    },
    openapi_tags=tags_metadata,
    # Можно поменять адреса документации или отключить их
    docs_url="/documentation",      # Swagger UI
    redoc_url=None,                 # отключить ReDoc
    openapi_url="/api/v1/openapi.json",
)


@app.get("/users/", tags=["users"])
async def read_users():
    return [{"name": "Иван"}]


@app.get("/items/", tags=["items"])
async def read_items():
    return [{"name": "Молоток"}]
```

Порядок тегов в `openapi_tags` определяет порядок секций в Swagger UI.

### 2. Кастомизация OpenAPI-схемы через `custom_openapi`

OpenAPI-схема генерируется лениво при первом обращении к `/openapi.json` и кешируется в `app.openapi_schema`. Чтобы вмешаться, переопределяем метод `app.openapi`.

```python
from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi

app = FastAPI(title="Custom OpenAPI", version="1.0.0")


@app.get("/items/")
async def read_items():
    return [{"name": "Молоток"}]


def custom_openapi():
    # 1. Если схема уже сгенерирована — отдаём из кеша (важно для производительности!)
    if app.openapi_schema:
        return app.openapi_schema

    # 2. Генерируем базовую схему через стандартную утилиту
    openapi_schema = get_openapi(
        title="Магазин — кастомная схема",
        version="3.1.0-custom",
        summary="API с кастомизированной OpenAPI-схемой",
        description="Здесь мы вручную дополнили автоматически сгенерированную схему.",
        routes=app.routes,          # ОБЯЗАТЕЛЬНО передать роуты приложения
    )

    # 3. Добавляем кастомные поля. Например, логотип для ReDoc (x-logo)
    openapi_schema["info"]["x-logo"] = {
        "url": "https://fastapi.tiangolo.com/img/logo-margin/logo-teal.png"
    }

    # 4. Добавляем своё расширение верхнего уровня (любые x-* поля разрешены)
    openapi_schema["x-internal-build"] = "build-2026-06-20"

    # 5. Сохраняем в кеш и возвращаем
    app.openapi_schema = openapi_schema
    return app.openapi_schema


# Подменяем метод. Теперь FastAPI будет звать нашу функцию.
app.openapi = custom_openapi
```

**Важные нюансы:**

- Кеш `app.openapi_schema` — это причина, почему важно проверять его в начале. Без проверки схема будет пересобираться на каждый запрос к `/openapi.json`.
- Если вы добавляете/меняете роуты динамически после старта (редкий кейс), нужно сбрасывать кеш: `app.openapi_schema = None`.
- `get_openapi` принимает множество аргументов: `title`, `version`, `openapi_version`, `summary`, `description`, `routes`, `webhooks`, `tags`, `servers`, `terms_of_service`, `contact`, `license_info`.

### 3. Добавление `servers` и кастомных полей в операции

```python
def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    schema = get_openapi(title="API", version="1.0.0", routes=app.routes)

    # Список серверов (полезно за прокси / для нескольких окружений)
    schema["servers"] = [
        {"url": "https://api.example.com", "description": "Прод"},
        {"url": "https://staging.example.com", "description": "Стейджинг"},
    ]

    # Пройтись по путям и добавить кастомное поле в конкретную операцию
    schema["paths"]["/items/"]["get"]["x-rate-limit"] = "100/min"

    app.openapi_schema = schema
    return schema


app.openapi = custom_openapi
```

### 4. OpenAPI Callbacks

**Callback** — это документация для случая, когда **ваш** API после обработки запроса сам сделает HTTP-запрос на URL, который ему передал клиент. Реальные вызовы вы реализуете сами (например, через `httpx`), а callbacks лишь описывают их в схеме.

Сценарий: клиент создаёт «счёт» (invoice) и передаёт `callback_url`. Когда счёт оплачен, ваш сервис POST-ит на этот URL. Нужно задокументировать, какой запрос придёт клиенту и какой ответ ожидается от него.

```python
from typing import Annotated

from fastapi import APIRouter, FastAPI
from pydantic import BaseModel, HttpUrl

app = FastAPI()


# --- Модели нашего эндпоинта ---
class Invoice(BaseModel):
    id: str
    title: str | None = None
    customer: str
    total: float


# --- Модели callback (то, что МЫ отправим клиенту, и что он нам вернёт) ---
class InvoiceEvent(BaseModel):
    description: str
    paid: bool


class InvoiceEventReceived(BaseModel):
    ok: bool


# Роутер для описания callback. Он НЕ монтируется в приложение —
# он нужен только для генерации документации.
invoices_callback_router = APIRouter()


@invoices_callback_router.post(
    "{$callback_url}/invoices/{$request.body.id}",
    response_model=InvoiceEventReceived,
)
async def invoice_notification(body: InvoiceEvent):
    """
    Документирует POST-запрос, который НАШ сервис отправит на callback_url клиента.
    Выражения {$callback_url} и {$request.body.id} — это OpenAPI callback-выражения:
    они подставляются из исходного запроса.
    """
    pass


@app.post("/invoices/", callbacks=invoices_callback_router.routes)
async def create_invoice(invoice: Invoice, callback_url: Annotated[HttpUrl | None, None] = None):
    """
    Создаёт счёт.

    Когда счёт будет оплачен, сервис отправит POST на `callback_url`
    (см. секцию Callbacks в документации этого эндпоинта).
    """
    # Здесь реальная логика. Реальный вызов callback вы делаете сами, например:
    #   async with httpx.AsyncClient() as client:
    #       await client.post(f"{callback_url}/invoices/{invoice.id}",
    #                          json={"description": "...", "paid": True})
    return {"msg": "Invoice received"}
```

Ключевое:
- `callbacks=invoices_callback_router.routes` передаётся в декоратор основного эндпоинта.
- Путь в callback-роуте использует **callback-выражения** OpenAPI: `{$callback_url}`, `{$request.body.id}`, `{$request.query.param}` и т.д.
- Это **только документация** — FastAPI ничего не вызывает автоматически.

### 5. OpenAPI Webhooks

**Webhooks** (OpenAPI 3.1+) описывают исходящие события, которые ваш сервис рассылает подписчикам. В отличие от callbacks, webhook не привязан к конкретному входящему запросу — это «глобальное» событие сервиса. Для этого есть `app.webhooks`, который работает как `APIRouter`.

```python
from datetime import datetime

from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


class Subscription(BaseModel):
    username: str
    monthly_fee: float
    start_date: datetime


# app.webhooks работает как роутер, но регистрирует webhook-определения,
# а не реальные эндпоинты.
@app.webhooks.post("new-subscription")
async def new_subscription(body: Subscription):
    """
    Когда у пользователя появляется новая подписка, мы отправим вам
    POST-запрос с этими данными на ваш webhook-URL.

    (URL подписчики настраивают в личном кабинете — он не в схеме.)
    """
```

В Swagger UI появится отдельная секция **Webhooks**. Это чисто декларативная вещь: реальную отправку (через `httpx` и т.п.) вы реализуете сами.

**Callbacks vs Webhooks:**

| | Callbacks | Webhooks |
|-|-----------|----------|
| Привязка | К конкретному эндпоинту (`callbacks=...`) | Глобально к приложению (`app.webhooks`) |
| URL | Приходит в запросе от клиента (`callback_url`) | Подписчик настраивает заранее (в схеме URL нет) |
| OpenAPI версия | 3.0+ | 3.1+ |
| Семантика | «В ответ на этот запрос я дёрну ваш URL» | «При таких событиях я разошлю уведомления» |

## Полный рабочий пример

```python
from datetime import datetime
from typing import Annotated

import httpx
from fastapi import APIRouter, FastAPI
from fastapi.openapi.utils import get_openapi
from pydantic import BaseModel, HttpUrl

# --- Метаданные и теги ---
tags_metadata = [
    {"name": "invoices", "description": "Счета и оплата."},
    {"name": "health", "description": "Служебные эндпоинты."},
]

app = FastAPI(
    title="Billing API",
    version="1.0.0",
    summary="Демо: метаданные, custom OpenAPI, callbacks, webhooks.",
    openapi_tags=tags_metadata,
    contact={"name": "Team", "email": "team@example.com"},
)


# ============ МОДЕЛИ ============
class Invoice(BaseModel):
    id: str
    customer: str
    total: float


class InvoiceEvent(BaseModel):
    description: str
    paid: bool


class InvoiceEventReceived(BaseModel):
    ok: bool


class Subscription(BaseModel):
    username: str
    monthly_fee: float
    start_date: datetime


# ============ CALLBACK ROUTER ============
invoices_callback_router = APIRouter()


@invoices_callback_router.post(
    "{$callback_url}/invoices/{$request.body.id}",
    response_model=InvoiceEventReceived,
)
async def invoice_notification(body: InvoiceEvent):
    """Документирует POST, который наш сервис отправит на callback клиента."""
    pass


# ============ WEBHOOKS ============
@app.webhooks.post("new-subscription")
async def new_subscription(body: Subscription):
    """Уведомление о новой подписке, рассылаемое подписчикам webhook."""


# ============ ЭНДПОИНТЫ ============
@app.get("/health", tags=["health"])
async def health():
    return {"status": "ok"}


@app.post("/invoices/", tags=["invoices"], callbacks=invoices_callback_router.routes)
async def create_invoice(invoice: Invoice, callback_url: Annotated[HttpUrl | None, None] = None):
    """Создаёт счёт и (при наличии callback_url) уведомит клиента об оплате."""
    # Реальный вызов callback (упрощённо):
    if callback_url:
        async with httpx.AsyncClient() as client:
            await client.post(
                f"{callback_url}/invoices/{invoice.id}",
                json={"description": "Счёт оплачен", "paid": True},
            )
    return {"msg": "Invoice received"}


# ============ КАСТОМНАЯ OPENAPI-СХЕМА ============
def custom_openapi():
    if app.openapi_schema:                      # кеш
        return app.openapi_schema

    openapi_schema = get_openapi(
        title=app.title,
        version=app.version,
        summary=app.summary,
        description="Расширенная схема с логотипом и серверами.",
        routes=app.routes,
        webhooks=app.webhooks.routes,            # не забываем webhooks!
        tags=app.openapi_tags,
        contact=app.contact,
    )

    # Логотип для ReDoc
    openapi_schema["info"]["x-logo"] = {
        "url": "https://fastapi.tiangolo.com/img/logo-margin/logo-teal.png"
    }
    # Серверы
    openapi_schema["servers"] = [
        {"url": "https://api.example.com", "description": "Прод"},
    ]
    # Кастомное поле верхнего уровня
    openapi_schema["x-api-id"] = "billing-v1"

    app.openapi_schema = openapi_schema
    return app.openapi_schema


app.openapi = custom_openapi
```

## Частые вопросы на собеседовании (Q/A)

**1. Как FastAPI генерирует OpenAPI-схему и где она кешируется?**
Лениво, при первом запросе к `/openapi.json` (или вызове `app.openapi()`). Результат кешируется в `app.openapi_schema`. Повторные обращения отдают кеш.

**2. Как кастомизировать схему?**
Написать функцию, которая вызывает `get_openapi(...)`, дополняет результат и сохраняет его в `app.openapi_schema`, затем присвоить эту функцию `app.openapi`.

**3. Зачем в `custom_openapi` проверка `if app.openapi_schema`?**
Чтобы не пересобирать схему на каждый запрос. Без неё теряется кеширование и страдает производительность.

**4. Что нужно обязательно передать в `get_openapi`?**
Как минимум `title`, `version` и `routes=app.routes`. Без `routes` схема будет пустой.

**5. Чем отличаются Callbacks от Webhooks?**
Callbacks привязаны к конкретному эндпоинту и URL приходит в запросе клиента (`callback_url`). Webhooks глобальны (`app.webhooks`), URL подписчик настраивает заранее, и это фича OpenAPI 3.1.

**6. Делает ли FastAPI реальные HTTP-вызовы для callbacks/webhooks?**
Нет. Это только документация в OpenAPI. Реальную отправку вы реализуете сами (например, через `httpx`).

**7. Что означают выражения `{$callback_url}` и `{$request.body.id}`?**
Это OpenAPI callback-выражения. Они подставляют значения из исходного запроса (URL из тела/параметров и поля тела) в путь callback.

**8. Как добавить логотип в ReDoc?**
Добавить `openapi_schema["info"]["x-logo"] = {"url": "..."}` в кастомной схеме.

**9. Как сгруппировать эндпоинты в Swagger UI и описать группы?**
Тегами: `tags=["users"]` в декораторе + `openapi_tags=[...]` в конструкторе `FastAPI` для описаний и порядка.

**10. Как поменять URL документации или отключить её?**
Параметры конструктора: `docs_url`, `redoc_url`, `openapi_url`. Передача `None` отключает соответствующую страницу.

**11. Можно ли добавлять произвольные поля в схему?**
Да, любые поля с префиксом `x-` (расширения) валидны по спецификации OpenAPI. Также можно менять любые стандартные части словаря.

**12. Как корректно учесть webhooks в кастомной схеме?**
Передать `webhooks=app.webhooks.routes` в `get_openapi`, иначе раздел webhooks в схему не попадёт.

## Подводные камни (gotchas)

- **Забытая проверка кеша** в `custom_openapi` → схема пересобирается каждый запрос. Медленно.
- **Забыли `routes=app.routes`** → в `get_openapi` получите пустую схему без эндпоинтов.
- **Кастомизация до объявления роутов**: `app.openapi = custom_openapi` можно ставить когда угодно, но `app.routes` читаются в момент вызова, поэтому порядок присваивания не критичен — критичен момент первого запроса.
- **Динамическое добавление роутов** после первого `/openapi.json`: кеш не сбросится сам. Нужно вручную `app.openapi_schema = None`.
- **Callbacks не вызываются автоматически** — частая ошибка ожидать, что FastAPI сам пошлёт запрос.
- **Webhooks требуют OpenAPI 3.1** — на старых инструментах (некоторые генераторы клиентов) раздел может игнорироваться.
- **Callback-роутер не монтируется** в приложение (`app.include_router` не нужен) — он передаётся только через `callbacks=...`.
- **x-поля и валидаторы**: некоторые сторонние линтеры OpenAPI могут ругаться на нестандартные расширения, если они не с префиксом `x-`.

## Лучшие практики

- Всегда кешируйте кастомную схему (`if app.openapi_schema: return ...`).
- Выносите метаданные (title, version, contact, tags) в конфиг/переменные окружения для разных окружений.
- Используйте `servers` в схеме при работе за прокси или несколькими окружениями.
- Документируйте callbacks и webhooks — это резко повышает ценность вашего API для интеграторов.
- Держите модели событий (`InvoiceEvent`, `Subscription`) рядом с основными моделями и переиспользуйте их в реальной отправке.
- Для расширений придерживайтесь префикса `x-`, чтобы не сломать совместимость с инструментами.
- Не отключайте `/openapi.json` в проде без необходимости — он нужен генераторам клиентов и мониторингу. Если нужно скрыть — закрывайте авторизацией, а не удалением.

## Шпаргалка

```python
# Метаданные
app = FastAPI(title="...", version="1.0.0", summary="...", description="...",
              openapi_tags=[{"name": "x", "description": "..."}],
              contact={"name": "...", "email": "..."},
              license_info={"name": "MIT"},
              docs_url="/docs", redoc_url="/redoc", openapi_url="/openapi.json")

# Кастомная схема
from fastapi.openapi.utils import get_openapi
def custom_openapi():
    if app.openapi_schema:                       # кеш!
        return app.openapi_schema
    s = get_openapi(title=app.title, version=app.version,
                    routes=app.routes, webhooks=app.webhooks.routes)
    s["info"]["x-logo"] = {"url": "..."}         # кастомное поле
    s["servers"] = [{"url": "https://api.example.com"}]
    app.openapi_schema = s
    return s
app.openapi = custom_openapi

# Сброс кеша при динамических изменениях
app.openapi_schema = None

# Callbacks (привязан к эндпоинту, URL из запроса)
cb = APIRouter()
@cb.post("{$callback_url}/invoices/{$request.body.id}")
async def _(body: InvoiceEvent): ...
@app.post("/invoices/", callbacks=cb.routes)
async def create_invoice(...): ...

# Webhooks (глобально, URL у подписчика)
@app.webhooks.post("new-subscription")
async def new_subscription(body: Subscription): ...
```
