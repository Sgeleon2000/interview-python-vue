# FastAPI — Response Model и статус-коды — конспект и вопросы

## О чём раздел

Этот раздел посвящён тому, **что и как API возвращает клиенту**. В FastAPI есть мощный механизм управления выходными данными:

- объявление модели ответа через **return type annotation** или через параметр `response_model`;
- **фильтрация выходных данных** — почему модель ответа почти никогда не должна совпадать с моделью входа (например, нельзя возвращать пароль);
- тонкая настройка сериализации: `response_model_exclude_unset`, `response_model_exclude_defaults`, `response_model_exclude_none`, `response_model_include`, `response_model_exclude`;
- разделение моделей на **Input / Output / DB** (классический паттерн `UserIn` / `UserOut` / `UserInDB`);
- использование `Union` и `list[...]` как типа ответа;
- управление HTTP **статус-кодами**: `status_code`, модуль `fastapi.status`, и возврат разных кодов через объект `Response`.

Главная идея: **модель ответа — это контракт API**. Она определяет, какие поля увидит клиент, какие типы они имеют, и что попадёт в автоматическую документацию OpenAPI/Swagger.

## Ключевые концепции

| Концепция | Что делает |
|-----------|------------|
| Return type annotation (`-> Model`) | Современный способ объявить модель ответа. FastAPI использует её для валидации, сериализации и документации. |
| `response_model=Model` | Альтернатива/дополнение к аннотации возврата. Имеет приоритет, если оба заданы. |
| Фильтрация выхода | FastAPI прогоняет возвращаемые данные через модель ответа, отбрасывая лишние поля. |
| `response_model_exclude_unset` | Не включать в ответ поля, которые НЕ были заданы явно (остались дефолтными). |
| `response_model_exclude_defaults` | Не включать поля, равные значению по умолчанию. |
| `response_model_exclude_none` | Не включать поля со значением `None`. |
| `response_model_include` / `exclude` | Включить/исключить конкретные поля (не рекомендуется как основной приём). |
| Наследование моделей | `UserIn(UserBase)`, `UserOut(UserBase)` — переиспользование общих полей. |
| `status_code` | HTTP-код успешного ответа (по умолчанию 200, для POST часто 201). |
| `fastapi.status` | Именованные константы кодов: `status.HTTP_201_CREATED`. |
| `Response` / возврат разных кодов | Динамическое управление статусом внутри обработчика. |

## Подробный разбор с примерами кода

### 1. Объявление модели ответа: два способа

**Способ 1 (рекомендуемый, современный) — return type annotation:**

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    price: float
    description: str | None = None
    tax: float | None = None


@app.post("/items/")
async def create_item(item: Item) -> Item:
    # Тип возврата "-> Item" одновременно:
    # 1) валидирует то, что мы возвращаем;
    # 2) фильтрует выходные данные по модели Item;
    # 3) документирует ответ в OpenAPI.
    return item
```

**Способ 2 — параметр `response_model`:**

```python
@app.post("/items/", response_model=Item)
async def create_item(item: Item):
    return item
```

Зачем нужен `response_model`, если есть аннотация возврата? Бывают случаи, когда **возвращаемый объект НЕ совпадает по типу с моделью ответа**. Тогда аннотация возврата была бы технически некорректной (линтеры/mypy ругаются), а `response_model` решает задачу:

```python
class UserIn(BaseModel):
    username: str
    password: str  # секрет!
    email: str


class UserOut(BaseModel):
    username: str
    email: str


# Возвращаем UserIn (с паролем), но наружу отдаём только UserOut (без пароля).
# Аннотация "-> UserOut" была бы ложью для type-checker'а, поэтому используем response_model.
@app.post("/user/", response_model=UserOut)
async def create_user(user: UserIn) -> UserIn:
    return user
```

> Если заданы И `response_model`, И аннотация возврата — **приоритет у `response_model`**. Это позволяет, например, аннотировать возврат как реальный тип для линтера, а фильтрацию задавать через `response_model`.

### 2. Фильтрация выходных данных (модель ответа != модель входа)

Ключевой принцип безопасности: **никогда не возвращайте пользователю входную модель с чувствительными полями**. Даже если в коде вы вернули объект с паролем, FastAPI прогонит его через модель ответа и пароль не утечёт:

```python
@app.post("/user/", response_model=UserOut)
async def create_user(user: UserIn) -> UserIn:
    # user содержит password, но в JSON-ответе его НЕ будет,
    # потому что UserOut не содержит поля password.
    return user
```

Это работает потому, что FastAPI берёт возвращаемые данные и создаёт из них экземпляр модели ответа (`UserOut.model_validate(...)`), оставляя только её поля.

### 3. `response_model_exclude_unset` — отдавать только реально заданные поля

Часто у модели много опциональных полей со значениями по умолчанию. Если объект пришёл из БД и часть полей не задавалась, мы не хотим засорять ответ дефолтами.

```python
class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float = 10.5
    tags: list[str] = []


items = {
    "foo": {"name": "Foo", "price": 50.2},
    "bar": {"name": "Bar", "description": "Описание бара", "price": 62, "tax": 20.2},
    "baz": {"name": "Baz", "description": None, "price": 50.2, "tax": 10.5, "tags": []},
}


@app.get("/items/{item_id}", response_model=Item, response_model_exclude_unset=True)
async def read_item(item_id: str) -> Item:
    return items[item_id]
```

Для `foo` ответ будет `{"name": "Foo", "price": 50.2}` — поля `description`, `tax`, `tags` не были заданы и потому не попадут в JSON.

> Важно: «не заданы» (`unset`) отличается от «равны значению по умолчанию». Если поле явно передали со значением, равным дефолту, оно считается **set** и попадёт в ответ.

Связанные опции:

- `response_model_exclude_defaults=True` — убрать поля, **значение которых равно дефолту** (даже если они были заданы явно).
- `response_model_exclude_none=True` — убрать поля со значением `None`.

```python
@app.get(
    "/items/{item_id}",
    response_model=Item,
    response_model_exclude_none=True,  # поля == None не попадут в JSON
)
async def read_item_no_none(item_id: str) -> Item:
    return items[item_id]
```

### 4. `response_model_include` / `response_model_exclude`

Быстрый способ оставить/убрать конкретные поля без создания отдельной модели. Принимают `set`/`list` имён полей.

```python
class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float = 10.5


# Оставить ТОЛЬКО name и description:
@app.get(
    "/items/{item_id}/name",
    response_model=Item,
    response_model_include={"name", "description"},
)
async def read_item_name(item_id: str) -> Item:
    return items[item_id]


# Убрать поле tax:
@app.get(
    "/items/{item_id}/public",
    response_model=Item,
    response_model_exclude={"tax"},
)
async def read_item_public(item_id: str) -> Item:
    return items[item_id]
```

> Официальная рекомендация: используйте `include/exclude` лишь для быстрых правок. Для постоянного контракта **лучше завести отдельную модель** — она явно документируется и переиспользуется.

### 5. Несколько моделей: наследование и паттерн UserIn / UserOut / UserInDB

Чтобы не дублировать поля, выделяют общую базу:

```python
from pydantic import BaseModel, EmailStr


class UserBase(BaseModel):
    username: str
    email: EmailStr
    full_name: str | None = None


class UserIn(UserBase):
    password: str  # приходит от клиента (открытый пароль)


class UserOut(UserBase):
    pass  # наружу — без пароля


class UserInDB(UserBase):
    hashed_password: str  # в БД — только хеш, никогда не открытый пароль


def fake_password_hasher(raw_password: str) -> str:
    # В реальности — bcrypt/argon2 через passlib.
    return "supersecret" + raw_password


def fake_save_user(user_in: UserIn) -> UserInDB:
    hashed_password = fake_password_hasher(user_in.password)
    # **user_in.model_dump() распаковывает поля модели в kwargs.
    user_in_db = UserInDB(**user_in.model_dump(), hashed_password=hashed_password)
    print("Пользователь сохранён в БД (не на самом деле)")
    return user_in_db


@app.post("/user/", response_model=UserOut)
async def create_user(user_in: UserIn) -> UserInDB:
    # Принимаем UserIn (с паролем), сохраняем UserInDB (с хешем),
    # отдаём UserOut (без секретов).
    user_saved = fake_save_user(user_in)
    return user_saved
```

Здесь три разные модели для трёх разных «миров»:

- **UserIn** — то, что клиент присылает (содержит открытый пароль);
- **UserInDB** — то, что хранится в базе (содержит хеш, а не пароль);
- **UserOut** — то, что мы отдаём наружу (никаких секретов).

`response_model=UserOut` гарантирует, что `hashed_password` не утечёт, даже если мы вернули `UserInDB`.

### 6. `Union` как тип ответа

Когда эндпоинт может вернуть один из нескольких типов:

```python
from typing import Union


class BaseItem(BaseModel):
    description: str
    type: str


class CarItem(BaseItem):
    type: str = "car"


class PlaneItem(BaseItem):
    type: str = "plane"
    size: int


items_db = {
    "item1": {"description": "Гоночный автомобиль", "type": "car"},
    "item2": {"description": "Самолёт", "type": "plane", "size": 5},
}


# Union в OpenAPI описывается как anyOf.
@app.get("/items/{item_id}", response_model=Union[PlaneItem, CarItem])
async def read_item_union(item_id: str) -> Union[PlaneItem, CarItem]:
    return items_db[item_id]
```

> Порядок в `Union` важен: более специфичную/детальную модель (`PlaneItem`) ставят раньше, чтобы Pydantic при сериализации не «обрезал» поля под более общую. В Pydantic v2 можно также использовать дискриминированные `Union` (`Field(discriminator="type")`) для надёжного выбора варианта.

Современный синтаксис через `|` тоже работает: `PlaneItem | CarItem`.

### 7. `list` как тип ответа

```python
class Item(BaseModel):
    name: str
    description: str


fake_items = [
    {"name": "Foo", "description": "Описание Foo"},
    {"name": "Bar", "description": "Описание Bar"},
]


@app.get("/items/", response_model=list[Item])
async def read_items() -> list[Item]:
    return fake_items
```

Каждый элемент списка фильтруется и валидируется по модели `Item`.

### 8. Статус-коды: `status_code` и модуль `fastapi.status`

По умолчанию успешный ответ — `200 OK`. Для создания ресурса принято `201 Created`:

```python
from fastapi import status


@app.post("/items/", status_code=status.HTTP_201_CREATED)
async def create_item(name: str):
    return {"name": name}
```

Можно передать и просто число: `status_code=201`. Но именованные константы из `fastapi.status` (или `starlette.status`) **читаемее и самодокументируются** — IDE подскажет, что это за код.

Часто используемые коды:

| Код | Константа | Когда |
|-----|-----------|-------|
| 200 | `HTTP_200_OK` | Успешный GET/обычный ответ |
| 201 | `HTTP_201_CREATED` | Ресурс создан (POST) |
| 204 | `HTTP_204_NO_CONTENT` | Успех без тела ответа (DELETE) |
| 400 | `HTTP_400_BAD_REQUEST` | Некорректный запрос |
| 404 | `HTTP_404_NOT_FOUND` | Ресурс не найден |
| 422 | `HTTP_422_UNPROCESSABLE_ENTITY` | Ошибка валидации (FastAPI выдаёт сам) |

> Для `204 No Content` тело должно быть пустым. Не возвращайте JSON-объект с этим кодом — корректнее вернуть `Response(status_code=status.HTTP_204_NO_CONTENT)`.

### 9. Динамический статус-код через `Response`

Иногда код ответа зависит от логики (например, «создал» vs «обновил»). Тогда внедряют объект `Response` как параметр:

```python
from fastapi import Response, status

items = {"foo": {"name": "Fighters"}}


@app.put("/items/{item_id}")
async def upsert_item(item_id: str, name: str, response: Response):
    if item_id in items:
        items[item_id]["name"] = name
        response.status_code = status.HTTP_200_OK  # обновили
        return items[item_id]
    items[item_id] = {"name": name}
    response.status_code = status.HTTP_201_CREATED  # создали
    return items[item_id]
```

Здесь итоговый статус задаётся **в рантайме**. При этом `status_code` в декораторе всё равно полезен для документации «по умолчанию».

Можно вернуть и собственный `JSONResponse`/`Response` напрямую — тогда FastAPI отдаст его как есть (без прогона через `response_model`):

```python
from fastapi.responses import JSONResponse


@app.get("/legacy/")
async def get_legacy():
    return JSONResponse(status_code=200, content={"legacy": True})
```

## Полный рабочий пример

```python
from typing import Annotated, Union

from fastapi import FastAPI, HTTPException, Path, Response, status
from pydantic import BaseModel, EmailStr, Field

app = FastAPI(title="Users & Items API")


# ---------- Модели пользователя (Input / Output / DB) ----------
class UserBase(BaseModel):
    username: str = Field(min_length=3, max_length=50)
    email: EmailStr
    full_name: str | None = None


class UserIn(UserBase):
    password: str = Field(min_length=8)


class UserOut(UserBase):
    pass


class UserInDB(UserBase):
    hashed_password: str


# ---------- Модели товаров ----------
class Item(BaseModel):
    name: str
    description: str | None = None
    price: float = Field(gt=0)
    tax: float = 10.5
    tags: list[str] = []


# «Псевдо-БД»
fake_users_db: dict[str, UserInDB] = {}
fake_items_db: dict[str, dict] = {
    "foo": {"name": "Foo", "price": 50.2},
    "bar": {"name": "Bar", "description": "Бар", "price": 62, "tax": 20.2},
}


def hash_password(raw: str) -> str:
    # Демонстрация. В проде — passlib (bcrypt/argon2).
    return "hashed:" + raw


# ---------- Эндпоинты пользователей ----------
@app.post("/users/", response_model=UserOut, status_code=status.HTTP_201_CREATED)
async def create_user(user_in: UserIn) -> UserInDB:
    """Создаём пользователя: пароль хешируется, наружу секреты не уходят."""
    if user_in.username in fake_users_db:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Пользователь уже существует",
        )
    user_db = UserInDB(
        **user_in.model_dump(exclude={"password"}),
        hashed_password=hash_password(user_in.password),
    )
    fake_users_db[user_in.username] = user_db
    # Возвращаем UserInDB, но response_model=UserOut отфильтрует hashed_password.
    return user_db


@app.get("/users/{username}", response_model=UserOut)
async def get_user(username: Annotated[str, Path()]) -> UserInDB:
    user = fake_users_db.get(username)
    if user is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, detail="Не найден"
        )
    return user


# ---------- Эндпоинты товаров ----------
@app.get(
    "/items/{item_id}",
    response_model=Item,
    response_model_exclude_unset=True,  # только реально заданные поля
)
async def read_item(item_id: str) -> Item:
    if item_id not in fake_items_db:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Нет товара")
    return fake_items_db[item_id]


@app.get("/items/", response_model=list[Item])
async def list_items() -> list[Item]:
    return list(fake_items_db.values())


@app.put("/items/{item_id}", response_model=Item)
async def upsert_item(item_id: str, item: Item, response: Response) -> Item:
    """Создаёт или обновляет товар, динамически выставляя статус-код."""
    if item_id in fake_items_db:
        response.status_code = status.HTTP_200_OK
    else:
        response.status_code = status.HTTP_201_CREATED
    fake_items_db[item_id] = item.model_dump()
    return item


@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_item(item_id: str) -> Response:
    fake_items_db.pop(item_id, None)
    # 204 не должен иметь тела:
    return Response(status_code=status.HTTP_204_NO_CONTENT)
```

## Частые вопросы на собеседовании (Q/A)

**1. В чём разница между объявлением модели ответа через `-> Model` и через `response_model=Model`?**
Оба задают модель ответа (валидация + фильтрация + документация). Аннотация возврата современнее и дружит с type-checker'ами. `response_model` нужен, когда реально возвращаемый тип не совпадает с моделью ответа (например, вернули `UserInDB`, а отдаём `UserOut`), либо когда нужно подавить аннотацию для линтера. Если заданы оба — **побеждает `response_model`**.

**2. Зачем разделять модель входа и модель ответа?**
Безопасность и чистота контракта. Входная модель может содержать пароль, служебные поля и т.п. Модель ответа фильтрует выход, поэтому секреты не утекают. Это паттерн `UserIn` / `UserOut` / `UserInDB`.

**3. Как FastAPI гарантирует, что пароль не утечёт, если я вернул объект с паролем?**
FastAPI берёт возвращаемые данные и создаёт из них экземпляр модели ответа (по сути `UserOut.model_validate(...)`), оставляя только поля этой модели. Поля, которых нет в модели ответа, отбрасываются.

**4. Чем `response_model_exclude_unset` отличается от `response_model_exclude_defaults`?**
`exclude_unset` убирает поля, которые **не задавались явно** (остались на дефолте «по умолчанию», их не передавали). `exclude_defaults` убирает поля, **значение которых равно дефолтному**, даже если их передали явно. То есть `unset` про «было ли задано», а `defaults` про «равно ли дефолту».

**5. Когда использовать `response_model_exclude_none`?**
Когда не хотим засорять JSON ключами со значением `null`. Полезно для PATCH-подобных частичных представлений и для компактных ответов.

**6. Почему `response_model_include/exclude` считают «не лучшей практикой»?**
Они задают набор полей «ad hoc» в декораторе, что хуже документируется и переиспользуется. Для стабильного контракта лучше завести отдельную Pydantic-модель — она явно описана в OpenAPI и видна в коде.

**7. Какой статус-код по умолчанию у POST и стоит ли его менять?**
По умолчанию FastAPI отдаёт `200 OK`. Для создания ресурса семантически правильнее `201 Created` (`status_code=status.HTTP_201_CREATED`).

**8. Откуда брать константы статус-кодов?**
Из `fastapi.status` (реэкспорт `starlette.status`): `from fastapi import status; status.HTTP_201_CREATED`. Числа тоже работают, но именованные константы читаемее.

**9. Как вернуть разные коды из одного обработчика?**
Внедрить `response: Response` в параметры и выставлять `response.status_code` в рантайме, либо вернуть готовый `JSONResponse`/`Response` с нужным кодом. Учтите: если вернуть `Response` напрямую, фильтрация по `response_model` не применяется.

**10. Что особенного в коде 204 No Content?**
У ответа не должно быть тела. Возвращайте `Response(status_code=204)` без содержимого, не передавайте JSON-объект.

**11. Как описать ответ, который может быть одним из нескольких типов?**
Использовать `Union[A, B]` (или `A | B`) как тип/`response_model`. В OpenAPI это `anyOf`. Более специфичную модель ставьте раньше; для надёжности используйте дискриминированный Union (`Field(discriminator=...)`).

**12. Что произойдёт, если возвращаемые данные не проходят валидацию модели ответа?**
FastAPI выбросит ошибку сериализации на стороне сервера (`ResponseValidationError`, обычно 500), потому что это баг сервера — он отдаёт данные, не соответствующие задекларированному контракту.

**13. Влияет ли `response_model` на входную валидацию?**
Нет. Входные данные валидируются по моделям параметров (`Body`, `Query` и т.д.). `response_model` управляет только выходом.

**14. Можно ли отключить модель ответа?**
Да: `response_model=None`. Это нужно, например, когда возвращаете «сырой» `Response` и не хотите, чтобы FastAPI пытался строить схему из аннотации возврата.

## Подводные камни (gotchas)

- **`exclude_unset` vs `exclude_defaults`** регулярно путают на собеседованиях — запомните различие «не задано» против «равно дефолту».
- **Возврат `Response`/`JSONResponse` напрямую обходит `response_model`** — фильтрация и валидация не сработают, секреты могут утечь. Будьте внимательны.
- **Порядок типов в `Union`** влияет на сериализацию: общая модель раньше специфичной «съест» дополнительные поля.
- **`204 No Content` с телом** — частая ошибка; тело должно быть пустым.
- **`response_model` + аннотация возврата одновременно**: помните, что приоритет у `response_model`.
- **`EmailStr` требует пакета** `email-validator` (`pip install pydantic[email]`).
- **`response_model_exclude_unset` и данные из dict**: если вы возвращаете `dict` (а не экземпляр модели), FastAPI сначала валидирует его в модель — поля, отсутствующие в dict и имеющие дефолт, будут считаться unset.
- **Чувствительные поля по умолчанию НЕ исключаются** — если поле есть в модели ответа, оно попадёт в JSON. Безопасность обеспечивается именно отдельной выходной моделью.
- **`response_model=None` нужен** для эндпоинтов, возвращающих `Response`, иначе FastAPI попытается построить схему и может упасть.

## Лучшие практики

1. **Всегда отделяйте выходную модель от входной**, особенно при наличии секретов. Паттерн `XxxIn` / `XxxOut` / `XxxInDB`.
2. **Предпочитайте отдельные модели** опциям `include/exclude` для постоянного контракта.
3. **Используйте именованные константы** `fastapi.status` вместо «магических чисел».
4. **Ставьте корректные коды**: 201 для создания, 204 для удаления без тела, 404 для отсутствующего ресурса (через `HTTPException`).
5. **`response_model_exclude_unset=True`** — хороший дефолт для GET-ответов из БД с множеством опциональных полей.
6. **Для разнотипных ответов** применяйте дискриминированные `Union` (Pydantic v2) — это надёжнее простого `Union`.
7. **Аннотируйте возврат реальным типом** (`-> UserInDB`) и фильтруйте через `response_model=UserOut` — так и линтер доволен, и контракт безопасен.
8. **Не возвращайте `Response` напрямую**, если нужна фильтрация — это обходит модель ответа.

## Шпаргалка

```python
from fastapi import FastAPI, Response, status
from pydantic import BaseModel

app = FastAPI()

# --- модель ответа: два способа ---
@app.get("/a")
async def a() -> Model: ...                       # через аннотацию возврата
@app.get("/b", response_model=Model)              # через параметр (приоритетнее)
async def b(): ...

# --- разделение моделей ---
class UserBase(BaseModel): ...
class UserIn(UserBase):    password: str          # вход
class UserOut(UserBase):   pass                   # выход (без секретов)
class UserInDB(UserBase):  hashed_password: str   # БД

@app.post("/users", response_model=UserOut)       # фильтрует секреты
async def create(u: UserIn) -> UserInDB: ...

# --- настройка сериализации ---
response_model_exclude_unset=True     # убрать НЕзаданные поля
response_model_exclude_defaults=True  # убрать поля == дефолту
response_model_exclude_none=True      # убрать поля == None
response_model_include={"name"}       # оставить только эти
response_model_exclude={"tax"}        # убрать эти

# --- список / union ---
response_model=list[Item]
response_model=PlaneItem | CarItem    # anyOf; специфичную модель — раньше

# --- статус-коды ---
status_code=status.HTTP_201_CREATED   # дефолтный код ответа
async def upsert(r: Response):        # динамический код
    r.status_code = status.HTTP_200_OK

@app.delete("/x", status_code=status.HTTP_204_NO_CONTENT)
async def d() -> Response:
    return Response(status_code=204)  # без тела

response_model=None                   # отключить модель ответа (для сырого Response)
```

| Код | Константа | Смысл |
|-----|-----------|-------|
| 200 | `HTTP_200_OK` | OK |
| 201 | `HTTP_201_CREATED` | Создано |
| 204 | `HTTP_204_NO_CONTENT` | Без тела |
| 400 | `HTTP_400_BAD_REQUEST` | Плохой запрос |
| 404 | `HTTP_404_NOT_FOUND` | Не найдено |
| 422 | `HTTP_422_UNPROCESSABLE_ENTITY` | Ошибка валидации |
```
