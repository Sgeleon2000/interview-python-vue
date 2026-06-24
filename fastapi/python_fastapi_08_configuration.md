# FastAPI — Конфигурация операций и обновление данных — конспект и вопросы

## О чём раздел

Раздел охватывает две связанные темы:

1. **Path Operation Configuration** — настройка самой операции (эндпоинта):
   код ответа по умолчанию, теги, заголовок и описание для документации,
   пометка «устарело» и т.д. Это всё то, что влияет на поведение и на
   автогенерируемую документацию (OpenAPI / Swagger UI).
2. **Обновление данных и `jsonable_encoder`** — как корректно обновлять записи
   через `PUT` (полная замена) и `PATCH` (частичное обновление), как Pydantic v2
   помогает делать частичные апдейты через `exclude_unset` и
   `model_copy(update=...)`, и зачем нужен `jsonable_encoder` для преобразования
   Pydantic-моделей и нестандартных типов в JSON-совместимые структуры.

## Ключевые концепции

- **`status_code`** в декораторе задаёт код успешного ответа (например, `201`
  для создания). Используйте `fastapi.status` для читаемости.
- **`tags`** группируют операции в документации; теги можно задавать строками
  или через `Enum` для единообразия.
- **`summary` / `description` / `response_description`** наполняют OpenAPI
  человекочитаемыми текстами. `description` можно вынести в docstring функции
  (с поддержкой Markdown).
- **`deprecated=True`** помечает операцию как устаревшую — она остаётся
  рабочей, но в документации видна как deprecated.
- **`jsonable_encoder`** превращает объекты (Pydantic-модели, `datetime`,
  `UUID`, `Decimal` и т.п.) в чистые JSON-совместимые Python-структуры (`dict`,
  `list`, `str`, `int`...). Нужен перед тем, как положить данные туда, где ждут
  JSON-совместимость (например, в «БД»-словарь, в `JSONResponse`).
- **`PUT` — полная замена**, **`PATCH` — частичное обновление**.
- **`exclude_unset=True`** в `model_dump()` возвращает только те поля, которые
  клиент **явно прислал**, что критично для `PATCH`.
- **`model_copy(update=...)`** (Pydantic v2) создаёт копию модели с применёнными
  изменениями — основа реализации частичного обновления.

## Подробный разбор с примерами кода

### 1. Path Operation Configuration: `status_code`

```python
from fastapi import FastAPI, status
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    price: float


# Код 201 Created для операции создания
@app.post("/items/", status_code=status.HTTP_201_CREATED)
async def create_item(item: Item):
    return item
```

`status_code` влияет и на реальный ответ, и на схему OpenAPI (в документации
будет показан именно `201`).

### 2. `tags`: группировка операций

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/items/", tags=["items"])
async def read_items():
    return [{"name": "Меч"}]


@app.get("/users/", tags=["users"])
async def read_users():
    return [{"name": "Алиса"}]
```

В Swagger UI операции будут сгруппированы по тегам `items` и `users`.

### 3. `tags` через `Enum` (единообразие тегов)

Чтобы не плодить строковые опечатки (`"items"` vs `"Items"`), теги удобно
описать `Enum`:

```python
from enum import Enum

from fastapi import FastAPI

app = FastAPI()


class Tags(str, Enum):
    items = "items"
    users = "users"


@app.get("/items/", tags=[Tags.items])
async def read_items():
    return ["Portal gun", "Plumbus"]


@app.get("/users/", tags=[Tags.users])
async def read_users():
    return ["Rick", "Morty"]
```

`Enum`-теги дают автодополнение и защиту от опечаток в больших проектах.

### 4. `summary` и `description`

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
    tags: Annotated[set[str], Field(default_factory=set)]


@app.post(
    "/items/",
    summary="Создать item",            # краткий заголовок операции
    description="Создаёт item со всей информацией: name, description, price, tax и набором тегов",
)
async def create_item(item: Item):
    return item
```

### 5. `description` из docstring (с Markdown)

Если описание длинное, удобнее вынести его в docstring функции — FastAPI
подхватит его как `description`, поддерживается Markdown.

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    price: float


@app.post("/items/", summary="Создать item")
async def create_item(item: Item):
    """
    Создать item со всей информацией:

    - **name**: каждый item должен иметь имя
    - **price**: цена обязательна
    - **tax**: налог, если применим
    """
    return item
```

### 6. `response_description`

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    price: float


@app.post(
    "/items/",
    summary="Создать item",
    response_description="Созданный item",  # описание УСПЕШНОГО ответа в OpenAPI
)
async def create_item(item: Item):
    return item
```

Примечание: OpenAPI требует описание ответа, поэтому если не задать
`response_description`, FastAPI подставит дефолтное «Successful Response».

### 7. `deprecated=True`

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/legacy/", tags=["legacy"], deprecated=True)
async def read_legacy():
    # Операция помечена как устаревшая, но продолжает работать
    return [{"item_id": "old"}]
```

В документации эндпоинт будет помечен как deprecated (перечёркнут).

### 8. `jsonable_encoder`: зачем и как

Иногда данные нужно представить в виде, совместимом с JSON (только `dict`,
`list`, `str`, `int`, `float`, `bool`, `None`). Например:

- сохранить Pydantic-модель в «БД», которая принимает только JSON-совместимые
  структуры;
- положить объект с `datetime`/`UUID`/`Decimal` в `JSONResponse`.

`jsonable_encoder` рекурсивно преобразует объект: Pydantic-модели → `dict`,
`datetime` → ISO-строка, `Decimal` → `float`/`str` и т.д.

```python
from datetime import datetime

from fastapi import FastAPI
from fastapi.encoders import jsonable_encoder
from pydantic import BaseModel

app = FastAPI()

fake_db: dict[str, dict] = {}


class Item(BaseModel):
    title: str
    timestamp: datetime  # не JSON-совместимый тип «как есть»
    description: str | None = None


@app.put("/items/{item_id}")
async def update_item(item_id: str, item: Item):
    # Превращаем модель (с datetime внутри) в JSON-совместимый dict
    json_compatible_item_data = jsonable_encoder(item)
    # Теперь это можно безопасно «сохранить» в БД, ожидающую JSON-типы
    fake_db[item_id] = json_compatible_item_data
    return json_compatible_item_data
```

`jsonable_encoder` возвращает **Python-структуру** (например, `dict`), а не
JSON-строку. JSON-строку из неё можно получить через `json.dumps`.

### 9. Body Updates: `PUT` — полная замена

`PUT` заменяет ресурс целиком. Поля, которые клиент не прислал, будут заменены
значениями по умолчанию из модели (это и есть «полная замена»).

```python
from typing import Annotated

from fastapi import FastAPI
from fastapi.encoders import jsonable_encoder
from pydantic import BaseModel, Field

app = FastAPI()


class Item(BaseModel):
    name: str | None = None
    description: str | None = None
    price: float | None = None
    tax: float = 10.5
    tags: Annotated[list[str], Field(default_factory=list)]


items = {
    "foo": {"name": "Foo", "price": 50.2},
    "bar": {"name": "Bar", "description": "Брусок", "price": 62, "tax": 20.2},
}


@app.put("/items/{item_id}", response_model=Item)
async def update_item(item_id: str, item: Item):
    # ВНИМАНИЕ: не присланные поля примут значения по умолчанию модели,
    # поэтому, например, tax станет 10.5, а tags — [] — это полная замена.
    update_item_encoded = jsonable_encoder(item)
    items[item_id] = update_item_encoded
    return update_item_encoded
```

Опасность `PUT`: если клиент пришлёт только `price`, то `tax` затрётся
дефолтом `10.5`, даже если в БД было `20.2`. Поэтому для частичных обновлений
используют `PATCH`.

### 10. Body Updates: `PATCH` — частичное обновление

`PATCH` обновляет только присланные поля. Алгоритм:

1. Достаём сохранённые данные и строим из них модель.
2. Из входной модели берём **только заданные** поля: `model_dump(exclude_unset=True)`.
3. Применяем их к сохранённой модели через `model_copy(update=...)`.
4. Сохраняем результат.

```python
from typing import Annotated

from fastapi import FastAPI
from fastapi.encoders import jsonable_encoder
from pydantic import BaseModel, Field

app = FastAPI()


class Item(BaseModel):
    name: str | None = None
    description: str | None = None
    price: float | None = None
    tax: float = 10.5
    tags: Annotated[list[str], Field(default_factory=list)]


items = {
    "foo": {"name": "Foo", "price": 50.2},
    "bar": {"name": "Bar", "description": "Брусок", "price": 62, "tax": 20.2},
    "baz": {"name": "Baz", "description": None, "price": 50.2, "tax": 10.5, "tags": []},
}


@app.get("/items/{item_id}", response_model=Item)
async def read_item(item_id: str):
    return items[item_id]


@app.patch("/items/{item_id}", response_model=Item)
async def update_item(item_id: str, item: Item):
    # 1. Текущие сохранённые данные
    stored_item_data = items[item_id]
    stored_item_model = Item(**stored_item_data)

    # 2. Только реально присланные поля (ключевой момент PATCH!)
    update_data = item.model_dump(exclude_unset=True)

    # 3. Копия модели с применёнными изменениями (Pydantic v2)
    updated_item = stored_item_model.model_copy(update=update_data)

    # 4. Сохраняем JSON-совместимое представление
    items[item_id] = jsonable_encoder(updated_item)
    return updated_item
```

### 11. Почему `PATCH` использует `exclude_unset`

`exclude_unset=True` возвращает в `dict` **только те поля, которые клиент явно
установил** при создании входной модели. Это позволяет отличить:

- «поле не прислали» (его не нужно трогать) — оно не попадёт в `update_data`;
- от «прислали значение по умолчанию» (например, `tax=10.5`).

Без `exclude_unset` `model_dump()` вернул бы **все** поля, включая дефолты, и
`PATCH` затёр бы существующие значения значениями по умолчанию — то есть повёл
бы себя как `PUT`. Поэтому `exclude_unset` — обязательный элемент корректного
`PATCH`.

Связанные опции `model_dump()`:

- `exclude_unset=True` — только явно заданные поля.
- `exclude_defaults=True` — исключить поля, равные значению по умолчанию.
- `exclude_none=True` — исключить поля со значением `None`.

### 12. Различие `PUT` и `PATCH` — таблица

| | `PUT` | `PATCH` |
|---|---|---|
| Семантика | полная замена ресурса | частичное обновление |
| Непришедшие поля | заменяются дефолтами модели | остаются без изменений |
| Идемпотентность | да | формально не гарантируется |
| Типичный приём | `jsonable_encoder(item)` | `exclude_unset` + `model_copy(update=...)` |

## Полный рабочий пример

```python
from enum import Enum
from typing import Annotated

from fastapi import FastAPI, HTTPException, status
from fastapi.encoders import jsonable_encoder
from pydantic import BaseModel, Field

app = FastAPI(title="Демо конфигурации операций и обновлений")


# ---------- Теги через Enum ----------
class Tags(str, Enum):
    items = "items"


# ---------- Модель ----------
class Item(BaseModel):
    name: str | None = None
    description: str | None = None
    price: float | None = None
    tax: float = 10.5
    tags: Annotated[list[str], Field(default_factory=list)]


# «БД» в памяти
items: dict[str, dict] = {
    "foo": {"name": "Foo", "price": 50.2},
}


@app.get("/items/{item_id}", response_model=Item, tags=[Tags.items])
async def read_item(item_id: str):
    if item_id not in items:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Item не найден")
    return items[item_id]


@app.post(
    "/items/{item_id}",
    response_model=Item,
    status_code=status.HTTP_201_CREATED,     # код успешного ответа
    tags=[Tags.items],
    summary="Создать item",                  # заголовок в документации
    response_description="Созданный item",   # описание успешного ответа
)
async def create_item(item_id: str, item: Item):
    """
    Создать **item** по идентификатору.

    Если item уже существует — вернётся ошибка `409`.
    """
    if item_id in items:
        raise HTTPException(status.HTTP_409_CONFLICT, "Item уже существует")
    items[item_id] = jsonable_encoder(item)  # полная JSON-совместимая запись
    return item


@app.put("/items/{item_id}", response_model=Item, tags=[Tags.items])
async def replace_item(item_id: str, item: Item):
    # PUT: полная замена — непришедшие поля примут дефолты модели
    items[item_id] = jsonable_encoder(item)
    return item


@app.patch("/items/{item_id}", response_model=Item, tags=[Tags.items])
async def patch_item(item_id: str, item: Item):
    # PATCH: частичное обновление
    if item_id not in items:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Item не найден")
    stored = Item(**items[item_id])
    update_data = item.model_dump(exclude_unset=True)   # только присланные поля
    updated = stored.model_copy(update=update_data)     # копия с изменениями
    items[item_id] = jsonable_encoder(updated)
    return updated


@app.get("/legacy/", tags=[Tags.items], deprecated=True)
async def legacy():
    # Устаревшая операция: работает, но помечена deprecated
    return {"detail": "use /items instead"}
```

## Частые вопросы на собеседовании (Q/A)

**1. Как задать код ответа по умолчанию для операции?**
Через аргумент `status_code` декоратора, например
`@app.post("/items/", status_code=status.HTTP_201_CREATED)`. Он влияет и на
ответ, и на OpenAPI-схему.

**2. Зачем нужны `tags` и как лучше их задавать?**
`tags` группируют операции в документации. В крупных проектах их задают через
`Enum` (`class Tags(str, Enum): ...`), чтобы избежать опечаток и получить
автодополнение.

**3. Чем `summary` отличается от `description`?**
`summary` — краткий заголовок операции, `description` — развёрнутое описание
(поддерживает Markdown). `description` удобно писать в docstring функции.

**4. Что делает `response_description`?**
Задаёт описание **успешного** ответа в OpenAPI. Если не указать, FastAPI
подставит «Successful Response», так как OpenAPI требует описание ответа.

**5. Что значит `deprecated=True`?**
Операция остаётся рабочей, но помечается в документации как устаревшая. Это
способ мягко выводить эндпоинты из эксплуатации.

**6. Что такое `jsonable_encoder` и зачем он нужен?**
Функция, рекурсивно преобразующая объект (Pydantic-модель, `datetime`, `UUID`,
`Decimal` и т.п.) в JSON-совместимую Python-структуру (`dict`/`list`/
примитивы). Нужен, чтобы сохранить данные туда, где ждут JSON-совместимость, или
вернуть их в `JSONResponse`.

**7. Возвращает ли `jsonable_encoder` JSON-строку?**
Нет, он возвращает Python-структуру (например, `dict`). Строку из неё можно
получить через `json.dumps`.

**8. В чём разница между `PUT` и `PATCH`?**
`PUT` — полная замена ресурса (непришедшие поля заменяются дефолтами модели),
`PATCH` — частичное обновление (трогаются только присланные поля).

**9. Почему `PUT` может «затереть» данные?**
Потому что при `PUT` поля, которые клиент не прислал, получают значения по
умолчанию модели. Например, `tax` станет дефолтным `10.5`, даже если в БД было
другое значение.

**10. Как реализовать частичное обновление в Pydantic v2?**
Взять только присланные поля `item.model_dump(exclude_unset=True)` и применить
их к сохранённой модели через `stored.model_copy(update=update_data)`.

**11. Зачем именно `exclude_unset=True` при `PATCH`?**
Чтобы в обновление попали только поля, которые клиент **явно** прислал. Без
этого `model_dump()` вернёт все поля (включая дефолты), и `PATCH` затрёт
существующие значения — поведёт себя как `PUT`.

**12. Чем отличаются `exclude_unset`, `exclude_defaults`, `exclude_none`?**
`exclude_unset` — убрать поля, которые не задавались явно; `exclude_defaults` —
убрать поля, равные значению по умолчанию; `exclude_none` — убрать поля со
значением `None`.

**13. Что делает `model_copy(update=...)`?**
Создаёт **новую** копию модели с применёнными изменениями из словаря `update`,
не мутируя исходный объект. Это идиоматичный способ частичного обновления в
Pydantic v2 (замена устаревшего `.copy(update=...)`).

**14. Зачем `jsonable_encoder` при `PATCH/PUT`, если есть `model_dump()`?**
`model_dump()` может оставить типы вроде `datetime` как объекты Python.
`jsonable_encoder` гарантирует полную JSON-совместимость результата перед
сохранением в хранилище, ожидающее JSON-типы.

## Подводные камни (gotchas)

- **`PUT` с неполным телом стирает поля.** Непришедшие поля заменяются
  дефолтами модели. Для частичных правок используйте `PATCH`.
- **Забыли `exclude_unset` в `PATCH`** — обновление превратится в полную замену
  дефолтами. Это классическая ошибка.
- **`jsonable_encoder` возвращает не строку, а структуру.** Не путайте с
  `json.dumps`.
- **`description` в docstring и в декораторе одновременно** — приоритет у
  аргумента `description` декоратора; docstring используется, только если
  аргумент не задан.
- **Pydantic v1 → v2.** В v2 это `model_dump()` и `model_copy()`; старые
  `dict()` и `copy()` устарели. На собеседовании про современный стиль ждут
  именно v2-методы.
- **`exclude_unset` и значения по умолчанию.** Если поле имеет дефолт и клиент
  прислал ровно дефолтное значение, оно всё равно считается «set» и попадёт в
  `update_data` — `exclude_unset` смотрит на факт присвоения, не на совпадение
  с дефолтом.
- **`status_code` не отменяет `raise HTTPException`.** Код из декоратора
  применяется к успешному ответу; ошибки задают свой код через исключение.

## Лучшие практики

- Явно задавайте `status_code` для операций создания (`201`) и удаления (`204`).
- Описывайте `summary`, `description`, `response_description` и теги — это
  заметно улучшает автодокументацию.
- Теги задавайте через `Enum` в средних и крупных проектах.
- Для обновлений по умолчанию выбирайте `PATCH` с `exclude_unset` +
  `model_copy(update=...)`; `PUT` — когда действительно нужна полная замена.
- Перед сохранением в хранилище приводите данные через `jsonable_encoder`.
- Используйте `Field(default_factory=...)` для изменяемых дефолтов (`list`,
  `set`, `dict`), чтобы избежать общего изменяемого состояния.
- Помечайте устаревшие эндпоинты `deprecated=True` вместо резкого удаления.

## Шпаргалка

```python
# Конфигурация операции
@app.post(
    "/items/",
    status_code=status.HTTP_201_CREATED,     # код успешного ответа
    tags=[Tags.items],                       # теги (лучше через Enum)
    summary="Создать item",                  # краткий заголовок
    description="Подробное описание...",      # или docstring функции (Markdown)
    response_description="Созданный item",   # описание успешного ответа
    deprecated=False,                        # пометка «устарело»
)
async def create_item(item: Item):
    ...

# Теги через Enum
class Tags(str, Enum):
    items = "items"
    users = "users"

# JSON-совместимое представление
from fastapi.encoders import jsonable_encoder
data = jsonable_encoder(item)   # dict/list/примитивы, datetime -> str

# PUT — полная замена
items[item_id] = jsonable_encoder(item)

# PATCH — частичное обновление (Pydantic v2)
stored = Item(**items[item_id])
update_data = item.model_dump(exclude_unset=True)  # только присланные поля
updated = stored.model_copy(update=update_data)    # копия с изменениями
items[item_id] = jsonable_encoder(updated)
```

| Опция `model_dump()` | Что исключает |
|---|---|
| `exclude_unset=True` | поля, не заданные явно |
| `exclude_defaults=True` | поля, равные значению по умолчанию |
| `exclude_none=True` | поля со значением `None` |
```
