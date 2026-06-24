# FastAPI — Path & Query Parameters — конспект и вопросы

## О чём раздел

Раздел про два самых распространённых источника входных данных в FastAPI: **path-параметры** (часть URL-пути, например `/items/{item_id}`) и **query-параметры** (часть строки запроса после `?`, например `/items?skip=0&limit=10`).

Разберём: как объявлять параметры с типами и автоконвертацией, как валидировать числа (`Path`, `ge/gt/le/lt`), почему важен порядок маршрутов, как ограничивать значения через `Enum`, как принимать path-параметры, содержащие слэши (`:path`), как делать query-параметры обязательными и опциональными, как валидировать строки (`Query`, `min_length/max_length/pattern`), как принимать списки в query и как добавлять метаданные (`title`, `description`, `alias`, `deprecated`). Везде используем современный синтаксис `Annotated` и Pydantic v2.

## Ключевые концепции

- **Path-параметр** — объявляется в пути в фигурных скобках `{...}` и как одноимённый аргумент функции. Всегда обязателен (он часть URL).
- **Query-параметр** — любой аргумент функции, **не** объявленный в пути и **не** являющийся Pydantic-моделью (телом). Передаётся в строке запроса.
- **Автоконвертация и валидация** — по аннотации типа (`int`, `float`, `bool`, `UUID`, `datetime`...). Невалидное значение → HTTP 422 с детальным телом ошибки.
- **`Annotated[...]`** — рекомендованный современный способ навешивать метаданные/валидацию (`Path`, `Query`, `Depends`) на параметр, не «съедая» значение по умолчанию.
- **`Path(...)`** — настройка валидации и метаданных path-параметра.
- **`Query(...)`** — то же для query-параметра.
- **Числовые ограничения**: `gt` (>), `ge` (>=), `lt` (<), `le` (<=).
- **Строковые ограничения**: `min_length`, `max_length`, `pattern` (регулярка; в Pydantic v2 — `pattern`, в v1 был `regex`).
- **`Enum`** — ограничение path/query набором допустимых значений (попадает в OpenAPI как enum).
- **`:path`-конвертер** — позволяет path-параметру содержать слэши (например, файловый путь).
- **Метаданные**: `title`, `description`, `alias`, `deprecated`, `examples`.
- **Порядок маршрутов** имеет значение: первый подходящий выигрывает.

## Подробный разбор с примерами кода

### 1. Базовый path-параметр с типом

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/items/{item_id}")
async def read_item(item_id: int):
    # item_id из URL автоматически конвертируется в int и валидируется.
    # /items/3   -> {"item_id": 3}
    # /items/foo -> 422 (не int)
    return {"item_id": item_id}
```

### 2. Порядок маршрутов — фиксированные пути выше параметрических

```python
# ВАЖНО: этот маршрут должен идти ВЫШЕ /users/{user_id},
# иначе "me" попадёт в user_id и роут /users/me никогда не сработает.
@app.get("/users/me")
async def read_current_user():
    return {"user_id": "the current user"}


@app.get("/users/{user_id}")
async def read_user(user_id: str):
    return {"user_id": user_id}
```

FastAPI проверяет маршруты по порядку объявления; срабатывает первый совпавший. Поэтому конкретные пути объявляйте раньше параметрических.

Похожая ситуация — дублирование пути: первый объявленный обработчик «затеняет» последующие с тем же путём и методом.

### 3. Path-параметр с предопределёнными значениями (Enum)

```python
from enum import Enum

from fastapi import FastAPI

app = FastAPI()


class ModelName(str, Enum):
    # Наследование от str делает значения сериализуемыми в JSON
    # и читаемыми в документации.
    alexnet = "alexnet"
    resnet = "resnet"
    lenet = "lenet"


@app.get("/models/{model_name}")
async def get_model(model_name: ModelName):
    # Допустимы только alexnet/resnet/lenet; иначе 422.
    # В Swagger UI появится выпадающий список значений.
    if model_name is ModelName.alexnet:
        return {"model_name": model_name, "message": "Deep Learning FTW!"}
    if model_name.value == "lenet":
        return {"model_name": model_name, "message": "LeCNN all the images"}
    return {"model_name": model_name, "message": "Have some residuals"}
```

### 4. Path-параметр, содержащий путь (`:path`)

```python
@app.get("/files/{file_path:path}")
async def read_file(file_path: str):
    # Конвертер :path разрешает слэши внутри значения.
    # /files/home/user/a.txt -> file_path = "home/user/a.txt"
    # Чтобы передать абсолютный путь, в URL будет двойной слэш:
    # /files//home/user/a.txt -> file_path = "/home/user/a.txt"
    return {"file_path": file_path}
```

Без `:path` слэш разделил бы сегменты пути и параметр получил бы только первый сегмент.

### 5. Валидация числовых path-параметров (Path + ge/gt/le/lt) — Annotated

```python
from typing import Annotated

from fastapi import FastAPI, Path

app = FastAPI()


@app.get("/items/{item_id}")
async def read_item(
    # Annotated: тип + метаданные валидации.
    # Path(...) — Ellipsis означает «обязателен» (path всегда обязателен).
    item_id: Annotated[int, Path(title="ID элемента", ge=1, le=1000)],
):
    # ge=1 -> item_id >= 1; le=1000 -> item_id <= 1000
    return {"item_id": item_id}
```

Числовые ограничители:

- `gt` — строго больше (>),
- `ge` — больше или равно (>=),
- `lt` — строго меньше (<),
- `le` — меньше или равно (<=).

Работают и для `int`, и для `float`.

### 6. Старый vs новый синтаксис (рекомендуется Annotated)

```python
# СТАРЫЙ стиль (значение по умолчанию = Path/Query). Работает, но не рекомендуется:
async def old_style(item_id: int = Path(ge=1)): ...

# НОВЫЙ стиль (рекомендован): метаданные в Annotated, дефолт отдельно.
async def new_style(item_id: Annotated[int, Path(ge=1)]): ...
```

Почему Annotated лучше:

- не «занимает» позицию значения по умолчанию — функцию можно вызывать как обычную (важно для тестов/переиспользования);
- один и тот же `Annotated`-тип можно переиспользовать в нескольких эндпоинтах;
- избегает путаницы «обязателен/опционален»;
- единый стиль с `Depends`.

### 7. Query-параметры: базово, опциональные, обязательные, дефолты

```python
from fastapi import FastAPI

app = FastAPI()

fake_db = [{"item_name": "Foo"}, {"item_name": "Bar"}, {"item_name": "Baz"}]


@app.get("/items")
async def list_items(skip: int = 0, limit: int = 10):
    # skip и limit НЕ объявлены в пути -> это query-параметры.
    # У них есть дефолты -> они опциональны.
    # /items?skip=1&limit=1
    return fake_db[skip : skip + limit]
```

Опциональный параметр (может отсутствовать):

```python
from typing import Annotated

from fastapi import FastAPI, Query

app = FastAPI()


@app.get("/search")
async def search(q: Annotated[str | None, Query()] = None):
    # str | None и дефолт None -> параметр опционален.
    # /search           -> q is None
    # /search?q=fast    -> q == "fast"
    if q:
        return {"q": q}
    return {"q": None}
```

Обязательный query-параметр (нет дефолта):

```python
@app.get("/need")
async def need(q: str):
    # Нет значения по умолчанию -> q обязателен.
    # Отсутствие -> 422.
    return {"q": q}


# Явно обязательный через Query и Ellipsis:
@app.get("/need2")
async def need2(q: Annotated[str, Query(min_length=3)] = ...):
    # ... (Ellipsis) делает параметр обязательным, сохраняя валидацию.
    return {"q": q}
```

### 8. Валидация строк (Query: min_length / max_length / pattern)

```python
from typing import Annotated

from fastapi import FastAPI, Query

app = FastAPI()


@app.get("/items")
async def read_items(
    q: Annotated[
        str | None,
        Query(
            min_length=3,        # минимум 3 символа
            max_length=50,       # максимум 50 символов
            pattern="^fixedquery$",  # Pydantic v2: pattern (в v1 было regex)
        ),
    ] = None,
):
    # Любое нарушение длины/паттерна -> 422.
    results = {"items": [{"item_id": "Foo"}, {"item_id": "Bar"}]}
    if q:
        results.update({"q": q})
    return results
```

Примечание Pydantic v2: используется `pattern`, а не устаревший `regex`. Если увидите `regex=` — это код под Pydantic v1.

### 9. Списки в query-параметрах

```python
from typing import Annotated

from fastapi import FastAPI, Query

app = FastAPI()


@app.get("/items")
async def read_items(
    # list[str] -> можно передать параметр несколько раз:
    # /items?q=a&q=b&q=c  -> q == ["a", "b", "c"]
    q: Annotated[list[str] | None, Query()] = None,
):
    return {"q": q}


@app.get("/defaults")
async def with_defaults(
    # Список со значениями по умолчанию:
    q: Annotated[list[str], Query()] = ["foo", "bar"],
):
    # /defaults -> q == ["foo", "bar"]
    return {"q": q}
```

Важно: чтобы FastAPI понял, что это список query-параметров, а не тело запроса, нужно явно указать `Query()` (для `list` это обязательно).

### 10. Метаданные query-параметров (title / description / alias / deprecated)

```python
from typing import Annotated

from fastapi import FastAPI, Query

app = FastAPI()


@app.get("/items")
async def read_items(
    q: Annotated[
        str | None,
        Query(
            title="Поисковый запрос",            # заголовок в OpenAPI
            description="Строка для поиска items",  # описание в документации
            alias="item-query",  # внешнее имя параметра в URL: ?item-query=...
            deprecated=True,      # пометить как устаревший (видно в Swagger UI)
            min_length=3,
        ),
    ] = None,
):
    # alias нужен, когда внешнее имя невалидно для Python-идентификатора
    # (item-query содержит дефис). В URL: /items?item-query=foo
    results = {"items": [{"item_id": "Foo"}]}
    if q:
        results.update({"q": q})
    return results
```

- `title` / `description` — для документации.
- `alias` — другое имя параметра в URL (когда внешнее имя нельзя сделать именем Python-аргумента, например с дефисом).
- `deprecated=True` — помечает параметр устаревшим в документации (но он продолжает работать).
- `include_in_schema=False` — скрыть параметр из OpenAPI-схемы целиком.
- `examples=[...]` — примеры значений в документации.

### 11. Метаданные одновременно для Path и Query

```python
from typing import Annotated

from fastapi import FastAPI, Path, Query

app = FastAPI()


@app.get("/items/{item_id}")
async def read_items(
    item_id: Annotated[int, Path(title="ID элемента", ge=1)],
    q: Annotated[str | None, Query(alias="item-query")] = None,
    size: Annotated[float, Query(gt=0, le=10.5)] = 1.0,
):
    result = {"item_id": item_id, "size": size}
    if q:
        result["q"] = q
    return result
```

Замечание про порядок аргументов: при использовании `Annotated` можно ставить аргументы в любом порядке — Python-ограничение «аргумент без дефолта не может идти после аргумента с дефолтом» обходится, потому что метаданные в `Annotated`, а не в значении по умолчанию. (В старом стиле это была реальная проблема, иногда решавшаяся через `*` в сигнатуре.)

### 12. bool-конвертация в query

```python
@app.get("/flag")
async def flag(active: bool = False):
    # Значения 1/true/True/on/yes -> True; 0/false/off/no -> False.
    # /flag?active=1 -> True
    return {"active": active}
```

## Полный рабочий пример

```python
# main.py
from enum import Enum
from typing import Annotated

from fastapi import FastAPI, Path, Query

app = FastAPI(title="Path & Query Params API", version="1.0.0")


class ModelName(str, Enum):
    """Допустимые модели (Enum -> enum в OpenAPI)."""
    alexnet = "alexnet"
    resnet = "resnet"
    lenet = "lenet"


# Конкретный путь ВЫШЕ параметрического, иначе "me" уйдёт в user_id.
@app.get("/users/me")
async def read_current_user():
    return {"user_id": "the current user"}


@app.get("/users/{user_id}")
async def read_user(user_id: str):
    return {"user_id": user_id}


@app.get("/models/{model_name}")
async def get_model(model_name: ModelName):
    """Path-параметр, ограниченный Enum."""
    return {"model_name": model_name, "value": model_name.value}


@app.get("/files/{file_path:path}")
async def read_file(file_path: str):
    """Path-параметр со слэшами через конвертер :path."""
    return {"file_path": file_path}


@app.get("/items/{item_id}")
async def read_item(
    # Path: числовая валидация (ge=1, le=1000) + метаданные.
    item_id: Annotated[int, Path(title="ID элемента", ge=1, le=1000)],
    # Query: опциональная строка с валидацией длины и alias.
    q: Annotated[
        str | None,
        Query(
            title="Поиск",
            description="Поисковая строка",
            alias="item-query",
            min_length=3,
            max_length=50,
        ),
    ] = None,
    # Список значений: /items/1?tag=a&tag=b
    tags: Annotated[list[str] | None, Query()] = None,
    # Пагинация: опциональные query с дефолтами и числовой валидацией.
    skip: Annotated[int, Query(ge=0)] = 0,
    limit: Annotated[int, Query(ge=1, le=100)] = 10,
    # float c ограничениями.
    size: Annotated[float, Query(gt=0, le=10.5)] = 1.0,
):
    """Сводный пример path + query параметров с валидацией и метаданными."""
    result = {
        "item_id": item_id,
        "skip": skip,
        "limit": limit,
        "size": size,
        "tags": tags,
    }
    if q:
        result["q"] = q
    return result
```

Проверка:

```bash
curl "http://127.0.0.1:8000/users/me"                  # текущий пользователь
curl "http://127.0.0.1:8000/users/john"                # user_id=john
curl "http://127.0.0.1:8000/models/resnet"             # ok
curl "http://127.0.0.1:8000/models/foo"                # 422 (не в Enum)
curl "http://127.0.0.1:8000/files/home/user/a.txt"     # :path со слэшами
curl "http://127.0.0.1:8000/items/5?item-query=fast&tag=a&tag=b&limit=2"
curl "http://127.0.0.1:8000/items/0"                   # 422 (ge=1 нарушен)
curl "http://127.0.0.1:8000/items/5?item-query=ab"     # 422 (min_length=3)
```

## Частые вопросы на собеседовании (Q/A)

**1. Чем path-параметр отличается от query-параметра?**
Path-параметр — часть пути URL (`/items/{id}`), всегда обязателен. Query-параметр — после `?` (`?skip=0`), объявляется как аргумент функции, которого нет в пути; обычно опционален (если есть дефолт). FastAPI определяет тип параметра по тому, есть ли имя в строке пути.

**2. Как FastAPI решает, что аргумент — это query, а что — path или body?**
Если имя есть в шаблоне пути — это path. Если нет и тип — простой скаляр (int, str, bool...) — это query. Если тип — Pydantic-модель — это тело запроса (body).

**3. Как сделать query-параметр обязательным?**
Не давать ему значения по умолчанию (`q: str`), либо явно `q: Annotated[str, Query()] = ...` (Ellipsis). Без дефолта параметр обязателен → его отсутствие даёт 422.

**4. Почему важен порядок объявления маршрутов?**
FastAPI матчит маршруты по порядку; срабатывает первый совпавший. `/users/me` нужно объявить раньше `/users/{user_id}`, иначе `me` попадёт в `user_id`.

**5. Что такое `:path`-конвертер и зачем он нужен?**
По умолчанию слэш разделяет сегменты пути. Конвертер `{file_path:path}` разрешает параметру содержать слэши — нужно для передачи файловых путей или вложенных ресурсов.

**6. В чём разница между `ge/gt/le/lt`?**
`ge` — >=, `gt` — >, `le` — <=, `lt` — <. Применяются в `Path()`/`Query()` для числовой валидации int/float.

**7. Почему рекомендуют `Annotated` вместо значения по умолчанию (`= Query(...)`)?**
`Annotated` отделяет метаданные от дефолта: функцию можно вызвать как обычную (тесты/переиспользование), нет путаницы с обязательностью, можно переиспользовать аннотированный тип, единый стиль с `Depends`, не страдает порядок аргументов.

**8. Как валидировать строку по длине и шаблону?**
`Query(min_length=3, max_length=50, pattern="^...$")`. В Pydantic v2 — параметр `pattern`; в v1 был `regex` (устарел).

**9. Как принять список значений в query?**
`q: Annotated[list[str] | None, Query()] = None`. Тогда `?q=a&q=b` даст `["a", "b"]`. Для списка `Query()` обязателен, иначе FastAPI решит, что это тело.

**10. Зачем нужен `alias` у параметра?**
Когда внешнее имя параметра в URL нельзя сделать именем Python-аргумента (например, содержит дефис `item-query`). `Query(alias="item-query")` связывает Python-имя с внешним.

**11. Что делает `deprecated=True` у Query?**
Помечает параметр устаревшим в OpenAPI/Swagger UI. Параметр продолжает работать, но клиентам видно, что его использовать не стоит.

**12. Как ограничить значения параметра фиксированным набором?**
Через `Enum` (наследник `str, Enum`). FastAPI валидирует значение и выводит выпадающий список в Swagger UI; неподходящее значение → 422.

**13. Что вернёт API при невалидном параметре и какой HTTP-код?**
HTTP 422 Unprocessable Entity с JSON-телом, где в `detail` указаны путь до поля (`loc`), сообщение (`msg`) и тип ошибки (`type`).

**14. Как скрыть параметр из документации, но оставить рабочим?**
`Query(include_in_schema=False)` — параметр не попадёт в OpenAPI-схему, но продолжит приниматься.

**15. Как преобразуется `bool` из query?**
`true/True/1/on/yes` → `True`; `false/False/0/off/no` → `False`. Прочее → 422.

## Подводные камни (gotchas)

- **Порядок маршрутов**: `/users/{user_id}` выше `/users/me` «съест» `me`. Конкретные пути — раньше.
- **Список без `Query()`**: `q: list[str]` без `Query()` FastAPI воспримет как тело запроса. Для query нужен явный `Query()`.
- **`regex` vs `pattern`**: в Pydantic v2 — только `pattern`; `regex` устарел/удалён.
- **Слэши в path**: без `:path` параметр получит лишь первый сегмент пути.
- **Старый стиль и порядок аргументов**: при `= Query(...)`/`= Path(...)` обязательные без дефолта не могут идти после параметров с дефолтом; раньше спасал `*` в сигнатуре. С `Annotated` проблема исчезает.
- **Опциональность**: `q: str | None` без дефолта `= None` всё равно делает параметр **обязательным** (просто разрешает значение None как тип). Чтобы был опциональным — нужен дефолт.
- **Ellipsis `...`**: `= ...` означает «обязателен», не «значение по умолчанию пустое».
- **Path всегда обязателен**: задавать ему дефолт бессмысленно — он часть URL.
- **`alias` и тело**: alias меняет внешнее имя; не путать с `serialization_alias`/`validation_alias` в моделях.

## Лучшие практики

- Используйте `Annotated[...]` для всех параметров с валидацией/метаданными — это современный рекомендованный стиль.
- Объявляйте конкретные маршруты раньше параметрических.
- Валидируйте числа (`ge/gt/le/lt`) и строки (`min_length/max_length/pattern`) на границе API — это бесплатная защита и понятные 422.
- Ограничивайте перечислимые значения через `Enum` вместо проверок вручную.
- Давайте параметрам `title`/`description` для качественной автодокументации.
- Для пагинации используйте опциональные query с разумными дефолтами и верхними границами (`limit: Query(ge=1, le=100) = 10`).
- Под Pydantic v2 используйте `pattern`, а не `regex`.
- Для списков всегда явно указывайте `Query()`.

## Шпаргалка

```python
from enum import Enum
from typing import Annotated
from fastapi import FastAPI, Path, Query

# Path: число с границами
item_id: Annotated[int, Path(ge=1, le=1000)]

# Path со слэшами
@app.get("/files/{file_path:path}")

# Query опциональный
q: Annotated[str | None, Query()] = None

# Query обязательный
q: str                              # без дефолта
q: Annotated[str, Query()] = ...    # явно

# Строковая валидация (Pydantic v2)
Query(min_length=3, max_length=50, pattern="^fixedquery$")

# Числовая валидация
Query(gt=0, ge=1, lt=100, le=10.5)

# Список в query: ?q=a&q=b -> ["a","b"]
q: Annotated[list[str] | None, Query()] = None

# Метаданные
Query(title="...", description="...", alias="item-query",
      deprecated=True, include_in_schema=False, examples=["foo"])

# Enum в path
class M(str, Enum): a = "a"; b = "b"
model: M
```

| Ограничитель | Смысл |
|---|---|
| `gt` | строго больше (>) |
| `ge` | больше или равно (>=) |
| `lt` | строго меньше (<) |
| `le` | меньше или равно (<=) |
| `min_length` / `max_length` | длина строки |
| `pattern` | регулярка (v2; в v1 — `regex`) |

| Параметр | Источник | Обязателен? |
|---|---|---|
| Path | часть URL `{...}` | всегда да |
| Query с дефолтом | после `?` | нет |
| Query без дефолта / `= ...` | после `?` | да |

Ключевое правило: `Annotated[<тип>, Path/Query(...)] = <дефолт>` — тип и валидация в `Annotated`, обязательность — наличием/отсутствием дефолта.
