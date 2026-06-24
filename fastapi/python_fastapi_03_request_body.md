# FastAPI — Request Body и модели Pydantic — конспект и вопросы

## О чём раздел

Тело запроса (request body) — это данные, которые клиент отправляет в API (обычно в формате JSON) методами `POST`, `PUT`, `PATCH`, `DELETE`. В отличие от path- и query-параметров, тело передаётся в полезной нагрузке HTTP-запроса.

В FastAPI тело запроса объявляется через классы-модели Pydantic (наследники `BaseModel`). FastAPI на их основе:

- **читает** тело запроса как JSON;
- **валидирует** данные (типы, обязательность, ограничения), возвращая понятную ошибку 422 при нарушении;
- **конвертирует** входные данные в нужные Python-типы;
- **документирует** структуру в OpenAPI/Swagger UI и ReDoc;
- даёт **автодополнение** и проверку типов в IDE.

Этот раздел покрывает: объявление тела через `BaseModel`, явный `Body()`, несколько тел сразу, вложенные модели, списки/словари, `Field` для валидации и метаданных, примеры данных (`json_schema_extra`, `examples`), дополнительные типы (UUID, datetime, Decimal, bytes, frozenset) и сочетание path + query + body.

> Все примеры на **Pydantic v2** и используют синтаксис `Annotated[...]`, который сейчас является рекомендуемым в FastAPI.

## Ключевые концепции

- **`BaseModel`** — базовый класс модели Pydantic. Параметр функции-обработчика с типом-наследником `BaseModel` автоматически воспринимается как тело запроса.
- **Правило определения источника параметра** в FastAPI:
  - параметр объявлен в пути (`/items/{item_id}`) → **path-параметр**;
  - тип параметра — `int`, `str`, `bool`, `float`, `list`, и т.п. (скалярный) → **query-параметр**;
  - тип параметра — модель Pydantic → **request body**.
- **`Body()`** — функция для явного указания, что скалярный параметр приходит из тела, а также для добавления метаданных, валидации и примеров.
- **`Field()`** — аналог `Query()`/`Path()`/`Body()`, но для **полей внутри модели** Pydantic: валидация и метаданные.
- **Вложенные модели** — модель может содержать другую модель как тип поля; FastAPI рекурсивно валидирует и документирует.
- **`model_config`** (Pydantic v2) — заменяет старый внутренний класс `Config`; через `json_schema_extra` можно добавить примеры в схему.
- **`Body(embed=True)`** — оборачивает единственное тело в ключ по имени параметра.
- FastAPI генерирует **JSON Schema** для каждой модели и встраивает её в общую **OpenAPI**-схему.

## Подробный разбор с примерами кода

### 1. Объявление тела через `BaseModel`

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


# Модель тела запроса
class Item(BaseModel):
    name: str                      # обязательное поле
    description: str | None = None # необязательное, значение по умолчанию None
    price: float                   # обязательное
    tax: float | None = None       # необязательное


@app.post("/items/")
async def create_item(item: Item):
    # item — уже провалидированный объект Item, а не dict
    item_dict = item.model_dump()  # Pydantic v2: model_dump() вместо .dict()
    if item.tax is not None:
        item_dict["price_with_tax"] = item.price + item.tax
    return item_dict
```

Что произойдёт:

- FastAPI прочитает тело как JSON;
- проверит, что `name` и `price` присутствуют и имеют нужные типы;
- приведёт типы при необходимости (например, `"42.0"` → `42.0`);
- при ошибке вернёт `422 Unprocessable Entity` с детальным описанием;
- сгенерирует JSON Schema и покажет её в `/docs`.

> В Pydantic v2: `.dict()` → `.model_dump()`, `.json()` → `.model_dump_json()`, `parse_obj()` → `model_validate()`.

### 2. Сочетание path + query + body

FastAPI распознаёт каждый параметр по правилам выше — можно смешивать все три источника в одном обработчике.

```python
from typing import Annotated
from fastapi import FastAPI, Path, Query
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    price: float
    tax: float | None = None


@app.put("/items/{item_id}")
async def update_item(
    item_id: Annotated[int, Path(title="ID товара", ge=1)],  # из пути
    item: Item,                                              # из тела (модель)
    q: Annotated[str | None, Query(max_length=50)] = None,   # из query-строки
):
    result = {"item_id": item_id, **item.model_dump()}
    if q:
        result["q"] = q
    return result
```

### 3. Несколько параметров тела (Body multiple)

Если объявить **несколько моделей**, FastAPI ожидает JSON с ключами по именам параметров.

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    price: float


class User(BaseModel):
    username: str
    full_name: str | None = None


@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item, user: User):
    return {"item_id": item_id, "item": item, "user": user}
```

Ожидаемое тело:

```json
{
    "item": { "name": "Молоток", "price": 12.5 },
    "user": { "username": "ivan", "full_name": "Иван Петров" }
}
```

### 4. Скалярные значения в теле через `Body()`

По умолчанию скаляр считается query-параметром. Чтобы заставить FastAPI взять его из тела, используем `Body()`.

```python
from typing import Annotated
from fastapi import Body, FastAPI
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    price: float


class User(BaseModel):
    username: str


@app.put("/items/{item_id}")
async def update_item(
    item_id: int,
    item: Item,
    user: User,
    # importance — скаляр, но мы хотим его в теле, а не в query
    importance: Annotated[int, Body(gt=0)],
):
    return {"item_id": item_id, "item": item, "user": user, "importance": importance}
```

Тело:

```json
{
    "item": { "name": "Молоток", "price": 12.5 },
    "user": { "username": "ivan" },
    "importance": 5
}
```

### 5. `Body(embed=True)` — вложить единственное тело в ключ

Если модель одна, FastAPI ждёт её поля «на верхнем уровне» JSON. Чтобы обернуть в ключ по имени параметра, используем `embed=True`.

```python
from typing import Annotated
from fastapi import Body, FastAPI
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    price: float


@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Annotated[Item, Body(embed=True)]):
    return {"item_id": item_id, "item": item}
```

Теперь ожидается:

```json
{ "item": { "name": "Молоток", "price": 12.5 } }
```

А без `embed=True` было бы:

```json
{ "name": "Молоток", "price": 12.5 }
```

### 6. `Field` — валидация и метаданные полей модели

`Field` живёт внутри модели и описывает отдельные поля. Импортируется из `pydantic`, а не из `fastapi`.

```python
from typing import Annotated
from fastapi import Body, FastAPI
from pydantic import BaseModel, Field

app = FastAPI()


class Item(BaseModel):
    name: str = Field(
        ...,                         # обязательное (можно опустить, как и в v2)
        title="Название товара",
        max_length=100,
    )
    description: str | None = Field(
        default=None,
        title="Описание",
        max_length=300,
    )
    price: float = Field(
        gt=0,                        # больше нуля
        description="Цена должна быть строго положительной",
    )
    tax: float | None = Field(default=None, ge=0)


@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Annotated[Item, Body(embed=True)]):
    return {"item_id": item_id, "item": item}
```

Полезные ограничения `Field`:

- числовые: `gt`, `ge`, `lt`, `le`, `multiple_of`;
- строковые: `min_length`, `max_length`, `pattern` (в v2 вместо устаревшего `regex`);
- метаданные: `title`, `description`, `examples`, `deprecated`;
- управление: `default`, `default_factory`, `alias`, `frozen`.

> В Pydantic v2 можно совмещать `Annotated` и `Field` так: `name: Annotated[str, Field(max_length=100)]`. Это рекомендуемый стиль, особенно когда поле переиспользуется.

### 7. Вложенные модели (Nested Models)

Модель может содержать другую модель, списки моделей, словари и т.д.

```python
from pydantic import BaseModel, Field, HttpUrl


class Image(BaseModel):
    url: HttpUrl                     # специальный тип Pydantic с валидацией URL
    name: str


class Item(BaseModel):
    name: str
    price: float = Field(gt=0)
    tags: set[str] = set()           # set — уникальные значения
    images: list[Image] | None = None  # список вложенных моделей


class Offer(BaseModel):
    name: str
    items: list[Item]                # модель внутри модели внутри списка
```

FastAPI:

- рекурсивно валидирует все уровни;
- `set[str]` гарантирует уникальность и преобразует список с дублями в множество;
- `HttpUrl` проверит корректность URL;
- покажет всю вложенную структуру в Swagger UI.

### 8. Списки и словари как тело

Тело не обязано быть моделью — это может быть `list[Model]` или `dict`.

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


class Image(BaseModel):
    url: str
    name: str


# Тело — JSON-массив объектов
@app.post("/images/multiple/")
async def create_multiple_images(images: list[Image]):
    return images


# Тело — JSON-объект произвольной структуры (ключи str, значения float)
@app.post("/index-weights/")
async def create_index_weights(weights: dict[int, float]):
    # JSON ключи всегда строки, но Pydantic приведёт их к int
    return weights
```

### 9. Примеры данных в схеме

#### 9.1. Через `model_config` и `json_schema_extra` (Pydantic v2)

```python
from pydantic import BaseModel, ConfigDict


class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

    # Pydantic v2: вместо внутреннего class Config
    model_config = ConfigDict(
        json_schema_extra={
            "examples": [
                {
                    "name": "Молоток",
                    "description": "Слесарный молоток 500 г",
                    "price": 12.5,
                    "tax": 2.5,
                }
            ]
        }
    )
```

#### 9.2. Через `Field(examples=...)`

```python
from pydantic import BaseModel, Field


class Item(BaseModel):
    name: str = Field(examples=["Молоток"])
    price: float = Field(examples=[12.5, 35.0])
```

#### 9.3. Через `Body(examples=...)` и `openapi_examples`

`Body(examples=[...])` добавляет примеры значений в JSON Schema. А `openapi_examples` (специфично для OpenAPI) позволяет давать **несколько именованных примеров** с заголовками и описаниями, которые красиво отображаются в Swagger UI.

```python
from typing import Annotated
from fastapi import Body, FastAPI
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    price: float
    tax: float | None = None


@app.put("/items/{item_id}")
async def update_item(
    item_id: int,
    item: Annotated[
        Item,
        Body(
            openapi_examples={
                "normal": {
                    "summary": "Обычный пример",
                    "description": "Корректные данные товара",
                    "value": {"name": "Молоток", "price": 12.5, "tax": 2.5},
                },
                "converted": {
                    "summary": "Пример с конвертацией",
                    "description": "price как строка будет приведён к float",
                    "value": {"name": "Молоток", "price": "12.5"},
                },
                "invalid": {
                    "summary": "Невалидный пример",
                    "description": "price строкой, которую нельзя сконвертировать",
                    "value": {"name": "Молоток", "price": "много"},
                },
            }
        ),
    ],
):
    return {"item_id": item_id, "item": item}
```

### 10. Дополнительные типы данных

Pydantic/FastAPI поддерживают типы, которые сериализуются в/из JSON автоматически.

```python
from datetime import datetime, time, timedelta
from decimal import Decimal
from uuid import UUID
from typing import Annotated

from fastapi import Body, FastAPI
from pydantic import BaseModel

app = FastAPI()


class Event(BaseModel):
    event_id: UUID                       # строка вида "..." валидируется как UUID
    start: datetime                      # ISO 8601 строка → datetime
    end: datetime | None = None
    duration: timedelta | None = None    # ISO 8601 длительность / секунды
    start_time: time | None = None       # "14:23:55.003"
    amount: Decimal                      # точное десятичное (для денег!)
    tags: frozenset[str] = frozenset()   # неизменяемое множество уникальных
    payload: bytes | None = None         # base64 в JSON ↔ bytes в Python


@app.post("/events/")
async def create_event(event: Event):
    # Можно работать с настоящими Python-типами
    start_plus_duration = None
    if event.duration is not None:
        start_plus_duration = event.start + event.duration
    return {
        "event_id": event.event_id,
        "start_plus_duration": start_plus_duration,
        "amount_doubled": event.amount * 2,  # Decimal-арифметика без потерь
    }
```

Часто используемые дополнительные типы:

| Тип | В JSON | Заметки |
|-----|--------|---------|
| `UUID` | строка | Стандартный UUID |
| `datetime` | ISO 8601 строка | `2026-06-20T14:23:55` |
| `date` / `time` | ISO строка | |
| `timedelta` | число секунд / ISO 8601 | |
| `Decimal` | число/строка | Для денег — без ошибок округления float |
| `bytes` | base64-строка | |
| `frozenset` / `set` | массив | Уникальность значений |
| `Path` (`pathlib`) | строка | |

## Полный рабочий пример

```python
from datetime import datetime
from decimal import Decimal
from typing import Annotated
from uuid import UUID, uuid4

from fastapi import Body, FastAPI, Path, Query
from pydantic import BaseModel, ConfigDict, Field, HttpUrl

app = FastAPI(title="Каталог товаров")


# --- Вложенные модели ---
class Image(BaseModel):
    url: HttpUrl = Field(description="Ссылка на изображение")
    name: str = Field(max_length=100)


class Item(BaseModel):
    name: str = Field(title="Название", min_length=1, max_length=100)
    description: str | None = Field(default=None, max_length=500)
    price: Decimal = Field(gt=0, description="Цена, строго положительная")
    tax: Decimal | None = Field(default=None, ge=0)
    tags: set[str] = Field(default_factory=set)
    images: list[Image] | None = None

    # Pydantic v2: примеры в схему
    model_config = ConfigDict(
        json_schema_extra={
            "examples": [
                {
                    "name": "Молоток",
                    "description": "Слесарный молоток 500 г",
                    "price": "12.50",
                    "tax": "2.50",
                    "tags": ["инструменты", "ручные"],
                    "images": [
                        {"url": "https://example.com/hammer.png", "name": "Главное фото"}
                    ],
                }
            ]
        }
    )


class User(BaseModel):
    username: str = Field(min_length=3)
    full_name: str | None = None


# In-memory "база данных"
fake_db: dict[UUID, dict] = {}


@app.post("/items/{category}", status_code=201)
async def create_item(
    # path-параметр
    category: Annotated[str, Path(title="Категория", min_length=1)],
    # тело: модель товара
    item: Item,
    # тело: модель пользователя-автора
    user: User,
    # тело: скаляр, помеченный Body
    importance: Annotated[int, Body(gt=0, le=10)],
    # query-параметр
    notify: Annotated[bool, Query(description="Отправить уведомление")] = False,
):
    """Создать товар. Демонстрирует path + query + несколько тел + скаляр в Body."""
    item_id = uuid4()
    record = {
        "id": item_id,
        "category": category,
        "item": item.model_dump(mode="json"),  # mode="json" сериализует Decimal/HttpUrl
        "author": user.model_dump(),
        "importance": importance,
        "notify": notify,
        "created_at": datetime.utcnow().isoformat(),
    }
    fake_db[item_id] = record
    return record


@app.get("/items/{item_id}")
async def get_item(item_id: UUID):
    return fake_db.get(item_id, {"error": "not found"})
```

Пример тела запроса для `POST /items/instruments?notify=true`:

```json
{
    "item": {
        "name": "Молоток",
        "price": "12.50",
        "tax": "2.50",
        "tags": ["инструменты"],
        "images": [{"url": "https://example.com/h.png", "name": "Фото"}]
    },
    "user": { "username": "ivan", "full_name": "Иван Петров" },
    "importance": 5
}
```

## Частые вопросы на собеседовании (Q/A)

**1. Как FastAPI понимает, что параметр — это тело запроса, а не query?**
По типу. Если тип параметра — наследник `BaseModel`, FastAPI берёт данные из тела. Скалярные типы (`int`, `str`, `float`, `bool`) по умолчанию считаются query-параметрами, а параметры из пути — path-параметрами. Чтобы заставить скаляр читаться из тела, используют `Body()`.

**2. В чём разница между `Field`, `Body`, `Query`, `Path`?**
`Field` — из `pydantic`, описывает поля **внутри** модели. `Body`, `Query`, `Path`, `Cookie`, `Header` — из `fastapi`, описывают **параметры функции-обработчика** и указывают источник данных. Набор валидаторов у них схожий (`gt`, `max_length`, и т.д.).

**3. Что делает `Body(embed=True)` и когда он нужен?**
Когда тело — единственная модель, FastAPI ждёт её поля на верхнем уровне JSON. `embed=True` оборачивает их в ключ по имени параметра. Нужен, если фронтенд присылает `{"item": {...}}`, либо если хочется единообразия при наличии нескольких тел.

**4. Как изменился способ задания примеров и конфигурации в Pydantic v2?**
Вместо внутреннего `class Config` используется `model_config = ConfigDict(...)`. Примеры в схему добавляют через `json_schema_extra={"examples": [...]}` или через `Field(examples=[...])`. На уровне параметра — `Body(examples=[...])` или богатый `Body(openapi_examples={...})`.

**5. Чем `examples` отличается от `openapi_examples`?**
`examples` попадают в JSON Schema (стандарт). `openapi_examples` — расширение OpenAPI, позволяет задать **несколько именованных** примеров с `summary`, `description`, `value`, которые Swagger UI показывает выпадающим списком. Это удобнее для демонстрации валидных/невалидных кейсов.

**6. Почему для денежных сумм рекомендуют `Decimal`, а не `float`?**
`float` (двоичная плавающая точка) даёт ошибки округления (`0.1 + 0.2 != 0.3`). `Decimal` хранит десятичные числа точно. Pydantic поддерживает `Decimal` и сериализует его корректно (при `model_dump(mode="json")` — в строку/число).

**7. Как валидировать вложенные структуры?**
Просто указать модель как тип поля: `images: list[Image]`. FastAPI/Pydantic рекурсивно валидируют каждый уровень и формируют вложенную JSON Schema. Поддерживаются `list[Model]`, `dict[str, Model]`, `Model | None` и т.д.

**8. Что вернёт FastAPI при невалидном теле?**
HTTP `422 Unprocessable Entity` с JSON, где в массиве `detail` для каждой ошибки указаны `loc` (путь до поля, например `["body", "price"]`), `msg` (сообщение) и `type` (код ошибки). Это поведение из коробки.

**9. Как сделать поле обязательным/необязательным?**
Обязательное — поле без значения по умолчанию: `name: str`. Необязательное — со значением по умолчанию: `description: str | None = None`. Важно: `str | None` без `= None` остаётся **обязательным**, просто допускает `null`. Для изменяемых дефолтов (списки, множества) используйте `Field(default_factory=list)`.

**10. Можно ли принимать произвольный JSON без жёсткой схемы?**
Да: `dict[str, Any]` или `dict[int, float]` (ключи приведутся), либо отдельно прочитать `Request.json()`. Но это теряет валидацию и документацию — используется как исключение.

**11. Как разрешить/запретить лишние поля в теле?**
Через `model_config = ConfigDict(extra="forbid")` — лишние поля вызовут ошибку 422; `extra="ignore"` (по умолчанию) — игнорируются; `extra="allow"` — сохраняются. На собеседовании важно знать про `extra="forbid"` для строгих API.

**12. Чем `.model_dump()` отличается от `.model_dump(mode="json")`?**
`model_dump()` возвращает Python-объекты (`Decimal`, `datetime`, `UUID` остаются как есть). `model_dump(mode="json")` сериализует их в JSON-совместимые типы (строки/числа). Второй вариант нужен, если результат пойдёт в `json.dumps` или в БД, ожидающую примитивы.

**13. Как переиспользовать общие ограничения полей?**
Через `Annotated` + `Field`: `Price = Annotated[Decimal, Field(gt=0)]`, затем `price: Price`. Это DRY-подход, рекомендуемый в Pydantic v2.

**14. Зачем разделять модели на входную и выходную (Item / ItemOut)?**
Чтобы не принимать на вход поля, которые должен задавать сервер (`id`, `created_at`), и не отдавать наружу чувствительные данные (`password_hash`). Разные модели для запроса и ответа (`response_model`) — стандартная практика безопасности и чистоты контракта.

## Подводные камни (gotchas)

- **`str | None` без дефолта — обязательное поле.** Чтобы сделать необязательным, нужен `= None`. Частая ошибка.
- **Изменяемые значения по умолчанию.** Никогда не пишите `tags: list[str] = []` — используйте `Field(default_factory=list)`. (В Pydantic это безопаснее, чем в обычных функциях, но `default_factory` — корректный путь.)
- **`Field` из `pydantic`, а не из `fastapi`.** Путаница с импортами — типичная ошибка.
- **Скаляр без `Body()` уходит в query.** Если ждали его в теле, а получили в query — забыли `Body()`.
- **`embed` меняет ожидаемую форму JSON.** Добавление/удаление `embed=True` ломает контракт с клиентом.
- **`float` для денег.** Ведёт к ошибкам округления; используйте `Decimal`.
- **JSON-ключи всегда строки.** Для `dict[int, float]` Pydantic приведёт ключи, но помните об этом при отладке.
- **`set`/`frozenset` теряют порядок и дубликаты.** Если порядок важен — используйте `list`.
- **Pydantic v1 → v2 API.** `.dict()`/`.json()`/`Config` устарели; на собеседовании ценят знание v2 (`model_dump`, `ConfigDict`).
- **`pattern` вместо `regex`.** В Pydantic v2 параметр `Field(regex=...)` переименован в `pattern=...`.
- **`HttpUrl` — это не `str`.** В Pydantic v2 это объект; при сериализации в БД используйте `str(url)` или `model_dump(mode="json")`.

## Лучшие практики

- Используйте **`Annotated[...]`** для всех параметров и для полей с `Field` — это рекомендованный современный стиль FastAPI/Pydantic v2.
- Разделяйте **входные и выходные** модели; задавайте `response_model` явно.
- Для денег и точных вычислений — **`Decimal`**, а не `float`.
- Документируйте поля через `title`/`description`/`examples` — это улучшает OpenAPI и DX.
- Включайте `extra="forbid"` для строгих публичных API, чтобы ловить опечатки в именах полей.
- Выносите повторяющиеся ограничения в **типы-алиасы** через `Annotated`.
- Предпочитайте `openapi_examples` для нескольких сценариев в Swagger UI.
- Не принимайте серверные поля (`id`, `created_at`) во входной модели.
- Используйте `model_dump(mode="json")` перед отправкой в системы, ждущие примитивы.
- Держите модели в отдельном модуле `schemas.py` для переиспользования и тестируемости.

## Шпаргалка

```python
from typing import Annotated
from decimal import Decimal
from uuid import UUID
from datetime import datetime
from fastapi import Body, FastAPI, Path, Query
from pydantic import BaseModel, ConfigDict, Field, HttpUrl

# --- Модель тела ---
class Item(BaseModel):
    name: str = Field(min_length=1, max_length=100)        # обязательное + строковые ограничения
    description: str | None = Field(default=None)          # необязательное
    price: Decimal = Field(gt=0)                            # деньги -> Decimal
    tags: set[str] = Field(default_factory=set)            # уникальные, изменяемый дефолт
    model_config = ConfigDict(
        extra="forbid",                                    # запретить лишние поля
        json_schema_extra={"examples": [{"name": "X", "price": "9.99"}]},
    )

# --- Источники параметров ---
# path: в пути /{x}        | query: скаляр       | body: модель / Body()
@app.put("/items/{item_id}")
async def update(
    item_id: Annotated[int, Path(ge=1)],                   # path
    item: Item,                                            # body (модель)
    user_id: Annotated[int, Body(gt=0)],                   # body (скаляр)
    q: Annotated[str | None, Query(max_length=50)] = None, # query
): ...

# --- Несколько тел -> JSON с ключами по именам параметров ---
async def h(item: Item, user: User): ...   # {"item": {...}, "user": {...}}

# --- Одно тело в ключе ---
item: Annotated[Item, Body(embed=True)]    # {"item": {...}}

# --- Списки/словари как тело ---
images: list[Image]
weights: dict[int, float]

# --- Примеры ---
Field(examples=[...])                       # на уровне поля
Body(examples=[...])                        # на уровне параметра
Body(openapi_examples={"ok": {"summary": ..., "value": {...}}})  # несколько именованных

# --- Pydantic v2 API ---
item.model_dump()              # -> Python-объекты
item.model_dump(mode="json")   # -> JSON-совместимые типы
Item.model_validate(data)      # парсинг из dict
model_config = ConfigDict(...) # вместо class Config

# --- Дополнительные типы ---
UUID, datetime, date, time, timedelta, Decimal, bytes, frozenset, HttpUrl
```
