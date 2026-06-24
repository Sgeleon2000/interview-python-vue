# FastAPI Advanced — Path Operation Advanced Configuration и Additional Status Codes — конспект и вопросы

## О чём раздел

Этот раздел Advanced User Guide посвящён тонкой настройке отдельных *операций пути* (path operations — это функции, обёрнутые в `@app.get`, `@app.post` и т.д.) и тому, как вернуть из одного эндпоинта несколько разных HTTP-статусов.

Две большие темы:

1. **Path Operation Advanced Configuration** — управление тем, как операция выглядит в OpenAPI-схеме:
   - `operation_id` — уникальный идентификатор операции в OpenAPI;
   - автогенерация `operationId` из имени функции;
   - исключение операции из схемы (`include_in_schema=False`);
   - описание операции из docstring (и обрезка докстринга);
   - прямое редактирование сгенерированной OpenAPI-схемы конкретной операции (`openapi_extra`).

2. **Additional Status Codes** — возврат дополнительных статус-кодов из одной операции. Главная идея: если вы хотите вернуть статус, отличный от основного, нужно вернуть `JSONResponse` (или другой `Response`) напрямую. При этом FastAPI больше **не** прогоняет данные через Pydantic-модель `response_model` — теряется валидация и сериализация.

Современный контекст: Pydantic v2, `Annotated[...]` для зависимостей и параметров, `async def` для эндпоинтов.

## Ключевые концепции

- **`operation_id`** — строковый идентификатор операции в OpenAPI. Должен быть **уникальным** в рамках всего приложения. Используется генераторами клиентов (openapi-generator, openapi-typescript-codegen) как имя метода клиента.
- **Автогенерация `operationId` из имени функции** — по умолчанию FastAPI генерирует длинный `operationId` вида `read_items_items__get`. Можно переопределить глобально через `generate_unique_id_function`, чтобы получить чистые имена клиента (`read_items`).
- **`include_in_schema=False`** — операция работает, но скрыта из `/docs`, `/redoc` и `/openapi.json`. Удобно для служебных/внутренних/устаревших эндпоинтов.
- **Описание из docstring** — текст докстринга функции попадает в поле `description` операции. Поддерживает Markdown. Можно обрезать докстринг маркером `\f` (form feed): всё после `\f` в схему не попадает.
- **`openapi_extra`** — словарь, который сливается (merge) в сгенерированную OpenAPI-схему конкретной операции. Позволяет добавить кастомные поля, `x-`-расширения или вручную описать тело запроса, которое FastAPI сам не разобрал.
- **Additional Status Codes** — чтобы вернуть несколько статусов из одной операции, возвращают `JSONResponse(content=..., status_code=...)` напрямую.
- **Потеря валидации/сериализации** — когда вы возвращаете `Response` напрямую, FastAPI отдаёт его как есть. `response_model`, Pydantic-сериализация (включая `jsonable_encoder` под капотом), фильтрация полей — всё это пропускается. Поэтому контент нужно сериализовать вручную через `jsonable_encoder`.

## Подробный разбор с примерами кода

### 1. `operation_id` — явное задание

```python
from fastapi import FastAPI

app = FastAPI()


# Явно задаём operation_id для операции.
# В сгенерированном клиенте метод будет называться "some_specific_id_you_define".
@app.get("/items/", operation_id="some_specific_id_you_define")
async def read_items() -> list[dict]:
    return [{"item_id": "Foo"}]
```

Зачем: генераторы клиентов берут `operationId` как имя функции/метода. Дефолтный длинный `operationId` даёт уродливые имена методов, поэтому его часто переопределяют.

### 2. Автогенерация `operationId` из имени функции

Можно автоматически назначить каждой операции `operationId`, равный имени её функции. Это удобно, но требует **уникальности имён всех функций-эндпоинтов** во всём приложении.

```python
from fastapi import FastAPI
from fastapi.routing import APIRoute

app = FastAPI()


@app.get("/items/")
async def read_items() -> list[dict]:
    return [{"item_id": "Foo"}]


# Проходим по всем маршрутам и назначаем operation_id = имя функции.
# ВАЖНО: вызывать ПОСЛЕ добавления всех маршрутов.
def use_route_names_as_operation_ids(app: FastAPI) -> None:
    for route in app.routes:
        if isinstance(route, APIRoute):
            # route.name по умолчанию равно имени функции-обработчика
            route.operation_id = route.name


use_route_names_as_operation_ids(app)
```

Подводный камень: если две функции называются одинаково (например, две `read_items` в разных роутерах) — `operationId` перестанет быть уникальным, и генераторы клиента сломаются.

### 2b. Глобальная генерация уникального ID (рекомендуемый способ)

Более «правильный» путь — задать `generate_unique_id_function` на уровне приложения. Тогда ID будут уникальными (включают тег), но при этом короткими и предсказуемыми.

```python
from fastapi import FastAPI
from fastapi.routing import APIRoute


# Возвращаем "тег-имяФункции", это и уникально, и читаемо.
def custom_generate_unique_id(route: APIRoute) -> str:
    tag = route.tags[0] if route.tags else "default"
    return f"{tag}-{route.name}"


app = FastAPI(generate_unique_id_function=custom_generate_unique_id)


@app.get("/items/", tags=["items"])
async def read_items() -> list[dict]:
    # operationId станет "items-read_items"
    return [{"item_id": "Foo"}]
```

### 3. Исключение из OpenAPI-схемы — `include_in_schema=False`

```python
from fastapi import FastAPI

app = FastAPI()


# Операция работает (её можно вызвать), но её НЕТ в /docs, /redoc и /openapi.json.
@app.get("/items/", include_in_schema=False)
async def read_items() -> list[dict]:
    return [{"item_id": "Foo"}]
```

Применение: внутренние health-check'и, технические эндпоинты, устаревшие маршруты, которые не нужно публиковать клиентам.

### 4. Описание из docstring и обрезка через `\f`

Текст докстринга идёт в `description` операции и поддерживает Markdown.

```python
from typing import Annotated
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI()


class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None
    tags: set[str] = set()


@app.post("/items/")
async def create_item(item: Item) -> Item:
    """
    Создать item со всеми данными:

    - **name**: обязательное имя
    - **description**: длинное описание
    - **price**: цена, обязательно
    - **tax**: налог (опционально)
    - **tags**: уникальные теги

    \f
    Текст после символа \\f в схему НЕ попадает.
    Здесь можно держать заметки для разработчиков,
    которые не нужны в публичной документации.
    """
    return item
```

Маркер `\f` (form feed, `\x0c`) обрезает докстринг для целей OpenAPI: до `\f` — публичное описание, после — служебный текст для разработчиков.

### 5. Прямое редактирование OpenAPI-схемы операции — `openapi_extra`

`openapi_extra` сливается в схему именно этой операции.

```python
from fastapi import FastAPI

app = FastAPI()


# Добавляем кастомное x-расширение прямо в схему операции.
@app.get("/items/", openapi_extra={"x-aperture-labs-portal": "blue"})
async def read_items() -> list[dict]:
    return [{"item_id": "portal-gun"}]
```

Более продвинутый сценарий — описать тело запроса вручную, когда вы парсите его сами (например, нестандартный content-type), а FastAPI не должен валидировать его как Pydantic-модель:

```python
from fastapi import FastAPI, Request

app = FastAPI()


@app.post(
    "/items/",
    openapi_extra={
        "requestBody": {
            "content": {
                "application/json": {
                    "schema": {
                        "type": "object",
                        "required": ["name", "price"],
                        "properties": {
                            "name": {"type": "string"},
                            "price": {"type": "number"},
                        },
                    }
                }
            },
            "required": True,
        }
    },
)
async def create_item(request: Request) -> dict:
    # Читаем и разбираем тело сами, минуя автоматическую валидацию FastAPI.
    raw_body = await request.json()
    # ... здесь могла бы быть кастомная валидация ...
    return {"received": raw_body}
```

### 6. Additional Status Codes — несколько статусов из одной операции

Базовая ситуация: PUT, который **создаёт** объект (201), если его нет, и **обновляет** (200), если есть. Основной статус-код у операции один, поэтому для второго возвращаем `JSONResponse` напрямую.

```python
from typing import Annotated
from fastapi import Body, FastAPI, status
from fastapi.responses import JSONResponse

app = FastAPI()

# Простейшее «хранилище» в памяти.
items = {"foo": {"name": "Fighters", "size": 6}}


@app.put("/items/{item_id}")
async def upsert_item(
    item_id: str,
    name: Annotated[str | None, Body()] = None,
    size: Annotated[int | None, Body()] = None,
):
    if item_id in items:
        # Объект есть — обновляем. Статус по умолчанию 200.
        item = items[item_id]
        item["name"] = name if name is not None else item["name"]
        item["size"] = size if size is not None else item["size"]
        return item  # обычный возврат -> проходит сериализацию FastAPI

    # Объекта нет — создаём и возвращаем ДРУГОЙ статус 201.
    item = {"name": name, "size": size}
    items[item_id] = item
    # Возвращаем JSONResponse напрямую, чтобы выставить status_code=201.
    return JSONResponse(status_code=status.HTTP_201_CREATED, content=item)
```

### 7. Почему теряется валидация/сериализация и как чинить

Когда вы возвращаете `JSONResponse` (или любой `Response`) напрямую, FastAPI отдаёт его клиенту **как есть**:

- `response_model` не применяется — фильтрации/преобразования полей нет;
- Pydantic-сериализация (которая под капотом использует `jsonable_encoder`) не выполняется;
- если в `content` положить объект, который стандартный JSON не умеет сериализовать (`datetime`, `UUID`, `Decimal`, Pydantic-модель), будет ошибка.

Решение — сериализовать вручную через `jsonable_encoder`:

```python
from datetime import datetime
from typing import Annotated
from fastapi import Body, FastAPI, status
from fastapi.encoders import jsonable_encoder
from fastapi.responses import JSONResponse
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    created_at: datetime  # datetime «голый» JSON не умеет сериализовать


@app.put("/items/{item_id}")
async def upsert_item(item_id: str, item: Item):
    # jsonable_encoder превращает datetime в ISO-строку,
    # Pydantic-модель -> в dict, и т.д. -> можно класть в JSONResponse.
    compatible = jsonable_encoder(item)
    return JSONResponse(status_code=status.HTTP_201_CREATED, content=compatible)
```

Без `jsonable_encoder` строка `JSONResponse(content=item)` с Pydantic-моделью или `datetime` упадёт с ошибкой сериализации.

## Полный рабочий пример

```python
from datetime import datetime
from typing import Annotated

from fastapi import Body, FastAPI, Request, status
from fastapi.encoders import jsonable_encoder
from fastapi.responses import JSONResponse
from fastapi.routing import APIRoute
from pydantic import BaseModel, Field


# --- Кастомная генерация operationId на уровне приложения ---
def custom_generate_unique_id(route: APIRoute) -> str:
    tag = route.tags[0] if route.tags else "default"
    return f"{tag}-{route.name}"


app = FastAPI(
    title="Advanced Path Operations",
    generate_unique_id_function=custom_generate_unique_id,
)


class Item(BaseModel):
    name: str
    description: str | None = Field(default=None, description="Длинное описание")
    price: float
    created_at: datetime = Field(default_factory=datetime.utcnow)


# Псевдо-хранилище.
db: dict[str, dict] = {}


@app.post("/items/", tags=["items"], operation_id="items-create-explicit")
async def create_item(item: Item) -> Item:
    """
    Создать новый item.

    Поддерживает **Markdown** в описании.

    \f
    Этот текст после \\f не попадёт в OpenAPI — заметка для разработчиков.
    """
    db[item.name] = jsonable_encoder(item)
    return item


@app.put("/items/{item_id}", tags=["items"])
async def upsert_item(
    item_id: str,
    name: Annotated[str | None, Body()] = None,
    price: Annotated[float | None, Body()] = None,
):
    """PUT, который создаёт (201) или обновляет (200) item."""
    if item_id in db:
        item = db[item_id]
        if name is not None:
            item["name"] = name
        if price is not None:
            item["price"] = price
        return item  # 200, обычная сериализация FastAPI

    new_item = Item(
        name=name or item_id,
        price=price or 0.0,
    )
    db[item_id] = jsonable_encoder(new_item)
    # Дополнительный статус-код: сериализуем вручную через jsonable_encoder.
    return JSONResponse(
        status_code=status.HTTP_201_CREATED,
        content=jsonable_encoder(new_item),
    )


# Скрытый из схемы служебный эндпоинт.
@app.get("/internal/health", include_in_schema=False, tags=["internal"])
async def health() -> dict:
    return {"status": "ok"}


# Операция с ручным разбором тела и кастомной схемой в OpenAPI.
@app.post(
    "/raw-items/",
    tags=["items"],
    openapi_extra={
        "requestBody": {
            "content": {
                "application/json": {
                    "schema": {
                        "type": "object",
                        "required": ["name"],
                        "properties": {"name": {"type": "string"}},
                    }
                }
            }
        },
        "x-internal-flag": True,
    },
)
async def create_raw_item(request: Request) -> dict:
    body = await request.json()
    return {"received": body}
```

## Частые вопросы на собеседовании (Q/A)

**1. Что такое `operationId` в OpenAPI и зачем он нужен?**
Это уникальный строковый идентификатор операции в OpenAPI-схеме. Генераторы клиентского кода используют его как имя метода/функции клиента. Должен быть уникальным во всём приложении.

**2. Как FastAPI генерирует `operationId` по умолчанию и почему его часто меняют?**
По умолчанию ID получается длинным и «шумным» (например, `read_items_items__get` — имя функции + путь + метод). Это даёт уродливые имена методов в сгенерированном клиенте, поэтому ID переопределяют — либо на имя функции, либо через `generate_unique_id_function`.

**3. Как сделать `operationId` равным имени функции и какой тут риск?**
Пройти по `app.routes`, для каждого `APIRoute` присвоить `route.operation_id = route.name`. Риск: имена функций-эндпоинтов должны быть уникальны во всём приложении, иначе ID перестанут быть уникальными и генераторы клиента сломаются.

**4. В чём преимущество `generate_unique_id_function` перед ручным переименованием?**
Это глобальный, декларативный механизм. Можно включить тег в ID (`items-read_items`), что гарантирует уникальность даже при совпадении имён функций в разных роутерах, и при этом имена остаются короткими и читаемыми.

**5. Что делает `include_in_schema=False`?**
Скрывает операцию из OpenAPI-схемы (`/openapi.json`, `/docs`, `/redoc`), но сама операция продолжает работать и быть доступной по своему URL. Это про документацию, а не про доступ.

**6. Как описание операции попадает в документацию и что делает `\f`?**
Описание берётся из docstring функции (с поддержкой Markdown). Символ `\f` (form feed) обрезает докстринг: всё до него идёт в OpenAPI `description`, всё после — остаётся внутренней заметкой для разработчиков.

**7. Зачем нужен `openapi_extra`?**
Это словарь, который сливается в OpenAPI-схему конкретной операции. Позволяет добавить кастомные поля, `x-`-расширения или вручную описать тело запроса, когда вы парсите его сами (нестандартный формат), минуя автоматическую валидацию.

**8. Как вернуть из одной операции два разных статус-кода (например, 200 и 201)?**
Основной статус возвращается обычным `return data`. Для дополнительного статуса возвращают `JSONResponse(status_code=..., content=...)` напрямую.

**9. Почему при возврате `JSONResponse` напрямую теряется валидация и сериализация?**
Потому что FastAPI отдаёт `Response` клиенту «как есть», не прогоняя контент через `response_model` и Pydantic-сериализацию. Фильтрация полей, преобразование типов и автоматическая JSON-совместимость не выполняются.

**10. Как тогда корректно сериализовать данные для `JSONResponse`?**
Через `jsonable_encoder(...)` — он превращает Pydantic-модели, `datetime`, `UUID`, `Decimal` и т.п. в JSON-совместимые структуры, которые можно безопасно положить в `content`.

**11. Что произойдёт, если положить Pydantic-модель или `datetime` прямо в `JSONResponse(content=...)` без `jsonable_encoder`?**
Будет ошибка сериализации, т.к. стандартный JSON-энкодер не знает, как сериализовать эти типы. `ORJSONResponse` частично спасает (умеет datetime/UUID), но для `JSONResponse` нужен `jsonable_encoder`.

**12. Покажется ли дополнительный статус-код (201) в OpenAPI-схеме автоматически?**
Нет. FastAPI не знает, что вы вернёте `JSONResponse` с другим статусом — для логики это нормально, но в документации этот статус нужно объявить вручную через параметр `responses={...}` операции.

## Подводные камни (gotchas)

- **Уникальность `operationId`.** При автогенерации из имени функции легко получить коллизию, если функции называются одинаково в разных модулях/роутерах. Используйте теги в `generate_unique_id_function`.
- **Порядок вызова `use_route_names_as_operation_ids`.** Назначать ID нужно ПОСЛЕ добавления всех маршрутов (включая `include_router`), иначе часть операций не получит ID.
- **`JSONResponse` ломает `response_model`.** Любой прямой возврат `Response` отключает фильтрацию/сериализацию по модели. Не забывайте `jsonable_encoder`.
- **Дополнительный статус-код не попадает в схему сам.** Чтобы он отобразился в `/docs`, его нужно явно объявить в `responses={...}`.
- **`include_in_schema=False` не защищает эндпоинт.** Он остаётся доступным по URL — это не безопасность, а только сокрытие из документации.
- **`openapi_extra` сливается, а не заменяет.** Если задать `requestBody` вручную, нужно следить, чтобы он не конфликтовал с тем, что FastAPI выводит из сигнатуры.
- **`\f` — именно form feed.** В строке Python это `\f` / `\x0c`. Случайно вписанный текст «\f» как обычные символы не сработает.

## Лучшие практики

- Для чистых имён клиента используйте `generate_unique_id_function` с тегом в ID, а не точечные `operation_id`.
- Давайте функциям-эндпоинтам осмысленные уникальные имена — это и `operationId`, и `route.name`.
- Документацию операции пишите в docstring (Markdown), а внутренние заметки прячьте за `\f`.
- Для дополнительных статус-кодов всегда оборачивайте контент в `jsonable_encoder` и объявляйте статусы в `responses={...}`.
- Скрывайте служебные эндпоинты `include_in_schema=False`, но не полагайтесь на это как на защиту.
- `openapi_extra` применяйте точечно — для `x-`-расширений и ручного описания нестандартных тел запроса.

## Шпаргалка

```python
# operationId явно
@app.get("/x", operation_id="my_id")

# operationId = имя функции (после всех маршрутов)
for r in app.routes:
    if isinstance(r, APIRoute):
        r.operation_id = r.name

# глобальная генерация ID
FastAPI(generate_unique_id_function=lambda r: f"{r.tags[0]}-{r.name}")

# скрыть из схемы
@app.get("/x", include_in_schema=False)

# обрезка docstring
"""публичное описание \f внутренняя заметка"""

# кастом в схему операции
@app.get("/x", openapi_extra={"x-flag": True})

# доп. статус-код + ручная сериализация
return JSONResponse(
    status_code=status.HTTP_201_CREATED,
    content=jsonable_encoder(data),
)

# объявить доп. статус в OpenAPI
@app.put("/x", responses={201: {"description": "Создано"}})
```
