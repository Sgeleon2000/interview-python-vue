# FastAPI Advanced — Прямой доступ к Request и dataclasses — конспект и вопросы

## О чём раздел

Раздел охватывает две независимые продвинутые темы FastAPI:

1. **Прямой доступ к объекту `Request`** (Starlette `Request`). Обычно FastAPI декларативно извлекает всё, что нужно: query, path, body, headers — вы просто объявляете параметры. Но иногда нужен "сырой" доступ к запросу: получить IP клиента, прочитать произвольные заголовки/cookies, сырое тело, работать со `state`, стримить тело. Для этого можно объявить параметр типа `Request` — FastAPI отдаст исходный объект запроса.

2. **Стандартные `dataclasses` вместо Pydantic-моделей.** FastAPI поддерживает не только Pydantic-модели, но и `@dataclass` из стандартной библиотеки (и `pydantic.dataclasses.dataclass`) — как для входных данных, так и в `response_model`. Это удобно для интеграции с существующим кодом, где dataclasses уже используются, хотя у них есть ограничения по сравнению с Pydantic.

## Ключевые концепции

1. **`Request` как параметр.** Объявите `request: Request` — FastAPI инъектирует объект запроса Starlette. Декларативные параметры можно использовать **одновременно** с `Request`.

2. **Что даёт `Request`:** `request.client` (host/port), `request.headers`, `request.cookies`, `request.query_params`, `await request.body()` (сырые байты), `await request.json()`, `request.stream()`, `request.url`, `request.method`, `request.state` (хранилище на время запроса).

3. **Когда использовать `Request` напрямую:** когда декларативных параметров недостаточно — нестандартный доступ к запросу, работа с `state` (например, установленным в middleware), низкоуровневая обработка тела.

4. **Цена сырого доступа:** параметры, прочитанные из `Request` вручную (например, заголовок через `request.headers["x"]`), **не валидируются** и **не документируются** в OpenAPI. Поэтому комбинируют: важное — декларативно, остальное — через `Request`.

5. **`dataclasses` как модели.** `@dataclass` можно указывать в типах параметров тела и в `response_model`. FastAPI/Pydantic v2 умеет валидировать и сериализовать их.

6. **`pydantic.dataclasses.dataclass`** даёт больше возможностей (валидаторы, вложенность) при синтаксисе dataclass, тогда как стандартный `dataclasses.dataclass` ограниченнее.

## Подробный разбор с примерами кода

### 1. Доступ к `Request`: клиент, заголовки, cookies

```python
from typing import Annotated

from fastapi import FastAPI, Request

app = FastAPI()


@app.get("/info/")
async def get_info(request: Request):
    # request — это исходный объект запроса Starlette
    client_host = request.client.host if request.client else None
    return {
        "client_host": client_host,                 # IP клиента
        "client_port": request.client.port if request.client else None,
        "method": request.method,                   # GET/POST/...
        "url": str(request.url),                    # полный URL
        "user_agent": request.headers.get("user-agent"),
        "all_cookies": request.cookies,             # dict cookies
        "query_params": dict(request.query_params), # query как dict
    }
```

### 2. Совмещение декларативных параметров и `Request`

```python
from typing import Annotated

from fastapi import FastAPI, Path, Request

app = FastAPI()


@app.get("/items/{item_id}")
async def read_item(
    item_id: Annotated[int, Path(description="ID элемента")],  # валидируется и документируется
    request: Request,                                          # сырой доступ
):
    return {
        "item_id": item_id,
        "client": request.client.host if request.client else None,
    }
```

`item_id` валидируется как `int`, попадает в OpenAPI; `request` даёт доступ к остальному запросу. Это рекомендуемый паттерн: декларативно — то, что важно для контракта; через `Request` — вспомогательное.

### 3. Сырое тело запроса

```python
from fastapi import FastAPI, Request

app = FastAPI()


@app.post("/raw-body/")
async def raw_body(request: Request):
    raw: bytes = await request.body()        # сырые байты тела
    # Можно распарсить вручную или обработать как поток
    return {"length": len(raw), "preview": raw[:50].decode("utf-8", "ignore")}


@app.post("/stream-body/")
async def stream_body(request: Request):
    total = 0
    # Потоковое чтение тела по чанкам (для больших загрузок)
    async for chunk in request.stream():
        total += len(chunk)
    return {"streamed_bytes": total}
```

Важно: тело можно прочитать как поток ИЛИ как `body()`/`json()`, но повторное чтение потока невозможно — потоковое чтение "потребляет" тело.

### 4. `request.state` и middleware

`request.state` — место для данных на время одного запроса (например, проставленных middleware: request_id, текущий пользователь, соединение с БД).

```python
import uuid
from typing import Annotated

from fastapi import FastAPI, Request

app = FastAPI()


@app.middleware("http")
async def add_request_id(request: Request, call_next):
    # Кладём данные в state на время запроса
    request.state.request_id = str(uuid.uuid4())
    response = await call_next(request)
    response.headers["X-Request-ID"] = request.state.request_id
    return response


@app.get("/whoami/")
async def whoami(request: Request):
    # Читаем то, что положил middleware
    return {"request_id": request.state.request_id}
```

`state` можно читать и в зависимостях — частый приём для проброса контекста.

### 5. Стандартные dataclasses как модель тела запроса

```python
from dataclasses import dataclass

from fastapi import FastAPI

app = FastAPI()


@dataclass
class Item:
    name: str
    price: float
    description: str | None = None
    tax: float | None = None


@app.post("/items/")
async def create_item(item: Item):
    # FastAPI валидирует входной JSON по dataclass и отдаёт экземпляр Item
    return item
```

FastAPI принимает `@dataclass` так же, как Pydantic-модель: парсит и валидирует тело запроса.

### 6. Dataclasses как `response_model`

```python
from dataclasses import dataclass, field

from fastapi import FastAPI

app = FastAPI()


@dataclass
class Item:
    name: str
    price: float
    tags: list[str] = field(default_factory=list)
    description: str | None = None


@app.get("/items/next/", response_model=Item)
async def read_next_item():
    # Можно вернуть dict или dataclass — FastAPI приведёт к Item и провалидирует
    return {"name": "Plumbus", "price": 42.0, "tags": ["a", "b"]}
```

### 7. `pydantic.dataclasses.dataclass` для расширенных возможностей и вложенности

```python
from dataclasses import field
from typing import List

from pydantic.dataclasses import dataclass  # ВАЖНО: из pydantic, не из stdlib

from fastapi import FastAPI

app = FastAPI()


@dataclass
class Item:
    name: str
    description: str | None = None


@dataclass
class Author:
    name: str
    # Вложенные dataclasses корректно валидируются Pydantic
    items: List[Item] = field(default_factory=list)


@app.post("/authors/{author_id}/items/", response_model=Author)
async def create_author_items(author_id: int, items: List[Item]) -> Author:
    return Author(name=f"author-{author_id}", items=items)
```

`pydantic.dataclasses.dataclass` поддерживает валидаторы, более строгую валидацию вложенных структур и лучше дружит с FastAPI/OpenAPI, чем stdlib-dataclass.

## Полный рабочий пример

```python
import uuid
from dataclasses import dataclass, field
from typing import Annotated

from fastapi import FastAPI, Path, Request
from pydantic.dataclasses import dataclass as pydantic_dataclass

app = FastAPI(title="Request + dataclasses")


# --- Middleware кладёт request_id в state -------------------------------------
@app.middleware("http")
async def request_id_middleware(request: Request, call_next):
    request.state.request_id = str(uuid.uuid4())
    response = await call_next(request)
    response.headers["X-Request-ID"] = request.state.request_id
    return response


# --- Прямой доступ к Request --------------------------------------------------
@app.get("/debug/")
async def debug(request: Request):
    return {
        "request_id": request.state.request_id,
        "client": request.client.host if request.client else None,
        "user_agent": request.headers.get("user-agent"),
        "cookies": request.cookies,
        "query": dict(request.query_params),
    }


@app.post("/upload-raw/")
async def upload_raw(request: Request):
    raw = await request.body()
    return {"received_bytes": len(raw)}


# --- Совмещение декларативного параметра и Request ----------------------------
@app.get("/items/{item_id}")
async def read_item(
    item_id: Annotated[int, Path(ge=1)],  # валидируется и документируется
    request: Request,                     # сырой доступ к запросу
):
    return {
        "item_id": item_id,
        "request_id": request.state.request_id,
        "client": request.client.host if request.client else None,
    }


# --- Стандартный dataclass как тело и response_model --------------------------
@dataclass
class SimpleItem:
    name: str
    price: float
    tags: list[str] = field(default_factory=list)
    description: str | None = None


@app.post("/simple-items/", response_model=SimpleItem)
async def create_simple_item(item: SimpleItem):
    return item


# --- pydantic.dataclasses для вложенности -------------------------------------
@pydantic_dataclass
class ChildItem:
    name: str
    price: float


@pydantic_dataclass
class Order:
    order_id: int
    items: list[ChildItem] = field(default_factory=list)


@app.post("/orders/", response_model=Order)
async def create_order(order: Order):
    # Вложенные ChildItem корректно провалидируются
    return order
```

Поведение:

- `GET /debug/?x=1` -> вернёт IP, user-agent, cookies, query и request_id из state.
- `POST /simple-items/` с `{"name": "a", "price": 1}` -> провалидирует и вернёт объект; лишние/неверные поля дадут 422.
- `POST /orders/` с вложенным списком items -> провалидирует каждый элемент.

## Частые вопросы на собеседовании (Q/A)

**1. Как получить IP клиента в FastAPI?**
Через `request.client.host` (объявив параметр `request: Request`). Учитывайте, что за прокси реальный IP может быть в заголовке `X-Forwarded-For` — нужен `ProxyHeadersMiddleware` или ручной разбор.

**2. Можно ли совмещать `Request` и обычные декларативные параметры?**
Да. Объявляете и `request: Request`, и нужные query/path/body-параметры — они работают вместе. Декларативные валидируются и документируются, `Request` даёт сырой доступ.

**3. Когда стоит обращаться к `Request` напрямую?**
Когда не хватает декларативных средств: нужен сырой/потоковый доступ к телу, произвольные заголовки/cookies, чтение `request.state` (данных от middleware), низкоуровневая логика.

**4. Какой минус у чтения данных через `Request` вручную?**
Эти данные не валидируются Pydantic и не попадают в схему OpenAPI/Swagger UI. Контракт API становится менее явным. Поэтому важные параметры объявляют декларативно.

**5. Что такое `request.state` и для чего он?**
Контейнер для произвольных данных, живущих в рамках одного запроса. Обычно заполняется в middleware (request_id, текущий пользователь, соединение) и читается в эндпоинтах/зависимостях.

**6. В чём разница между `await request.body()` и `request.stream()`?**
`body()` читает всё тело в память как `bytes`. `stream()` отдаёт тело по чанкам асинхронно — для больших загрузок без полной загрузки в память. Тело можно прочитать один раз: stream "потребляет" его.

**7. Поддерживает ли FastAPI стандартные dataclasses?**
Да. `@dataclass` из stdlib можно использовать как тип параметра тела и как `response_model`. FastAPI/Pydantic v2 валидирует и сериализует их.

**8. Чем `pydantic.dataclasses.dataclass` отличается от stdlib `dataclass`?**
Pydantic-версия добавляет полноценную валидацию, поддержку валидаторов, лучшую работу с вложенными структурами и более точную генерацию JSON Schema. Stdlib-версия ограниченнее: базовая валидация, меньше возможностей.

**9. Какие ограничения у dataclasses по сравнению с Pydantic BaseModel?**
Нет методов `.model_dump()`/`.model_validate()` (у stdlib), меньше тонкой настройки (`Field`-валидации, алиасы, кастомные валидаторы, конфиг сериализации), сложнее с продвинутыми типами и вложенностью. Для богатой валидации предпочтительнее `BaseModel`.

**10. Можно ли вернуть dict из эндпоинта с `response_model`-dataclass?**
Да. FastAPI приведёт dict к указанному dataclass и провалидирует/отфильтрует поля согласно модели.

**11. Зачем вообще использовать dataclasses, если есть Pydantic?**
Для интеграции с существующим кодом, где dataclasses уже применяются (например, доменные модели), чтобы не дублировать схемы. Это снижает связность с Pydantic в слое домена.

**12. Как FastAPI понимает, что dataclass — это тело запроса?**
По типу параметра: сложный тип (dataclass/BaseModel) без специального маркера трактуется как тело запроса (JSON), аналогично Pydantic-моделям.

## Подводные камни (gotchas)

- **`request.client` может быть `None`** (например, в тестах/некоторых ASGI-окружениях) — проверяйте на `None` перед обращением к `.host`.
- **Реальный IP за прокси** не равен `request.client.host` — нужен `X-Forwarded-For` и доверенный прокси (`ProxyHeadersMiddleware`), иначе IP — это адрес балансировщика.
- **Однократное чтение тела.** После `request.stream()` повторно `body()`/`json()` уже не получить (тело потреблено). Не смешивайте декларативный body-параметр и ручное чтение тела одновременно.
- **Данные из `Request` не в OpenAPI.** Заголовки/параметры, прочитанные вручную, не документируются и не валидируются — теряется явность контракта.
- **`request.state` живёт только в рамках запроса** и не потокобезопасен между запросами — не используйте для глобального состояния.
- **Stdlib dataclass без валидаторов.** Не ждите от `@dataclass` сложной валидации (диапазоны, regex, кастомные правила) — для этого нужен `pydantic.dataclasses.dataclass` или `BaseModel`.
- **Мутабельные значения по умолчанию.** В dataclass для списков/словарей используйте `field(default_factory=list)`, а не `= []` (иначе ошибка/общая ссылка).
- **Вложенность в stdlib dataclass** валидируется слабее, чем в Pydantic; для надёжной валидации вложенных структур берите pydantic-вариант.

## Лучшие практики

- Декларируйте всё, что входит в контракт (валидация + OpenAPI), а `Request` используйте лишь для того, чего нельзя выразить декларативно.
- Всегда проверяйте `request.client` на `None`.
- За прокси настраивайте получение реального IP через доверенные заголовки.
- Для больших тел используйте `request.stream()`; не читайте тело дважды.
- Для богатой валидации предпочитайте `BaseModel`; dataclasses — для интеграции с существующим доменным кодом.
- Если используете dataclasses с вложенностью/валидацией — берите `pydantic.dataclasses.dataclass`.
- В dataclass для коллекций — `field(default_factory=...)`.
- Используйте `request.state` для контекста запроса (request_id, пользователь), наполняемого в middleware.

## Шпаргалка

```python
# Прямой доступ к Request
async def ep(request: Request):
    request.client.host           # IP (проверяйте на None!)
    request.headers.get("x")      # заголовки
    request.cookies               # cookies (dict)
    request.query_params          # query
    await request.body()          # сырые байты
    await request.json()          # распарсенный JSON
    request.stream()              # async-итератор чанков (большие тела)
    request.state.request_id      # данные на время запроса (из middleware)

# Совмещение декларативного и сырого
async def ep(item_id: Annotated[int, Path()], request: Request): ...

# Stdlib dataclass как тело и response_model
@dataclass
class Item:
    name: str
    price: float
    tags: list[str] = field(default_factory=list)   # mutable default!

@app.post("/", response_model=Item)
async def create(item: Item): ...

# Pydantic dataclass (валидация + вложенность)
from pydantic.dataclasses import dataclass
@dataclass
class Order:
    id: int
    items: list[Item] = field(default_factory=list)

# dataclass vs BaseModel:
#   BaseModel  -> богатая валидация, Field, валидаторы, model_dump/validate
#   dataclass  -> интеграция с существующим кодом, ограниченная валидация
```
