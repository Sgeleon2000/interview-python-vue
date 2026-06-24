# FastAPI Advanced — Продвинутые зависимости — конспект и вопросы

## О чём раздел

Раздел посвящён продвинутым техникам работы с системой внедрения зависимостей (Dependency Injection) в FastAPI. Базовые зависимости — это обычные функции, которые FastAPI вызывает за вас и подставляет результат в эндпоинт. Но иногда нужно **параметризовать** саму зависимость: создать одну переиспользуемую проверку, которую можно настраивать разными аргументами в момент объявления, а не в момент запроса.

Ключевая идея раздела: **зависимостью может быть любой "вызываемый" объект (callable)**, а не только функция. Это значит, что экземпляр класса с методом `__call__` тоже может быть зависимостью. Это открывает возможность для:

- параметризованных зависимостей (один класс — много по-разному настроенных экземпляров);
- фабрик зависимостей (функция, которая возвращает другую функцию-зависимость);
- переиспользуемых валидаторов (например, проверка, что query-параметр содержит фиксированную подстроку).

Всё это отлично сочетается с `Annotated[...]` из Pydantic v2 / современного FastAPI и с `async`/`await`.

## Ключевые концепции

1. **Callable как зависимость.** FastAPI принимает в `Depends()` любой вызываемый объект. Для функции он смотрит на её параметры. Для класса — на параметры `__init__`. Для экземпляра класса — на параметры метода `__call__`.

2. **Параметризованная зависимость через `__call__`.** Создаём класс, в `__init__` которого передаём конфигурацию (например, искомую подстроку). Метод `__call__` принимает параметры запроса (query, path, и т.д.) и выполняет логику. Затем создаём экземпляр(ы) этого класса и используем их в `Depends()`.

3. **Фабрика зависимостей.** Обычная функция, которая принимает параметры конфигурации и **возвращает** внутреннюю функцию (или корутину). Эта внутренняя функция и есть настоящая зависимость; FastAPI анализирует её сигнатуру.

4. **Разделение конфигурации и параметров запроса.** Конфигурация фиксируется в момент объявления маршрута (через `__init__` или аргументы фабрики), а параметры запроса извлекаются в момент каждого запроса (через `__call__` или возвращённую функцию).

5. **Сочетание с `Annotated`.** Современный стиль — объявлять зависимость как `Annotated[ReturnType, Depends(checker)]`. Это позволяет переиспользовать тип-алиас и делает код самодокументируемым.

6. **`FixedContentQueryChecker`** — каноничный пример из документации: проверяет, что query-параметр `q` содержит заданную при инициализации подстроку.

## Подробный разбор с примерами кода

### 1. Класс как параметризованная зависимость (`__call__`)

```python
from typing import Annotated

from fastapi import Depends, FastAPI

app = FastAPI()


class FixedContentQueryChecker:
    """
    Параметризованная зависимость.

    Конфигурация (fixed_content) передаётся в __init__ один раз,
    а сам callable (__call__) вызывается на каждый запрос и читает query-параметр.
    """

    def __init__(self, fixed_content: str):
        # Конфигурация фиксируется в момент создания экземпляра
        self.fixed_content = fixed_content

    def __call__(self, q: str = "") -> bool:
        # Этот метод выполняется как зависимость на каждый запрос.
        # FastAPI смотрит именно на сигнатуру __call__ -> увидит query-параметр q.
        if q:
            return self.fixed_content in q
        return False


# Создаём настроенный экземпляр. Это уже "вызываемый объект".
checker = FixedContentQueryChecker("bar")


@app.get("/query-checker/")
async def read_query_check(
    # FastAPI анализирует сигнатуру checker.__call__, а не __init__:
    fixed_content_included: Annotated[bool, Depends(checker)],
):
    # Возвращает True, если в q есть подстрока "bar"
    return {"fixed_content_in_query": fixed_content_included}
```

Что здесь важно:

- `checker` — это **экземпляр**, а не класс. FastAPI понимает, что объект вызываемый, и анализирует `__call__`.
- В `__call__` объявлен `q: str = ""` — для FastAPI это обычный query-параметр, он попадёт в Swagger UI.
- Можно создать несколько по-разному настроенных экземпляров: `checker_bar = FixedContentQueryChecker("bar")`, `checker_baz = FixedContentQueryChecker("baz")` — и использовать их в разных маршрутах.

### 2. Несколько настроенных экземпляров одного класса

```python
# Разные конфигурации — один и тот же класс
require_bar = FixedContentQueryChecker("bar")
require_admin = FixedContentQueryChecker("admin")


@app.get("/needs-bar/")
async def needs_bar(ok: Annotated[bool, Depends(require_bar)]):
    return {"contains_bar": ok}


@app.get("/needs-admin/")
async def needs_admin(ok: Annotated[bool, Depends(require_admin)]):
    return {"contains_admin": ok}
```

Это и есть смысл "параметризации": логика проверки написана один раз, а поведение варьируется через конфигурацию экземпляра.

### 3. Фабрика зависимостей (зависимость, возвращающая зависимость)

Альтернатива классу — функция-фабрика. Она принимает конфигурацию и возвращает вложенную функцию, которую FastAPI и использует как зависимость.

```python
from typing import Annotated, Callable

from fastapi import Depends, FastAPI, HTTPException, Query, status

app = FastAPI()


def make_min_length_checker(min_length: int) -> Callable[[str], str]:
    """
    Фабрика зависимостей: возвращает функцию-зависимость,
    замыкающую (closure) параметр min_length.
    """

    def checker(q: Annotated[str, Query()] = "") -> str:
        # FastAPI анализирует сигнатуру этой внутренней функции
        if len(q) < min_length:
            raise HTTPException(
                status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
                detail=f"Параметр q должен быть не короче {min_length} символов",
            )
        return q

    return checker


# Создаём конкретные зависимости из фабрики
check_min_3 = make_min_length_checker(3)
check_min_10 = make_min_length_checker(10)


@app.get("/short/")
async def short(q: Annotated[str, Depends(check_min_3)]):
    return {"q": q}


@app.get("/long/")
async def long(q: Annotated[str, Depends(check_min_10)]):
    return {"q": q}
```

Отличие от класса:

- **Класс** удобен, когда состояние сложное, есть несколько методов, нужно хранить ресурсы (например, клиент БД).
- **Фабрика-замыкание** легче и лаконичнее для простых случаев. Замыкание сохраняет `min_length`.

### 4. Async-версия фабрики

Зависимости в FastAPI могут быть синхронными или асинхронными. Если внутри есть `await` (например, обращение к БД/кешу), делаем внутреннюю функцию корутиной.

```python
from typing import Annotated, Awaitable, Callable

from fastapi import Depends, FastAPI, HTTPException, status

app = FastAPI()


def make_role_checker(required_role: str) -> Callable[..., Awaitable[str]]:
    """Фабрика, возвращающая АСИНХРОННУЮ зависимость."""

    async def checker(x_role: Annotated[str, ...] = "") -> str:
        # представим, что тут await-обращение к внешнему сервису ролей
        # role = await roles_service.resolve(x_role)
        role = x_role  # упрощённо
        if role != required_role:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Требуется роль {required_role!r}",
            )
        return role

    return checker
```

### 5. Класс, использующий зависимости внутри себя

`__call__` может сам объявлять под-зависимости. FastAPI рекурсивно разрешает их.

```python
from typing import Annotated

from fastapi import Depends, FastAPI, Header, HTTPException, status

app = FastAPI()


async def get_token(x_token: Annotated[str | None, Header()] = None) -> str:
    if not x_token:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Нет X-Token")
    return x_token


class PermissionChecker:
    def __init__(self, allowed_token: str):
        self.allowed_token = allowed_token

    def __call__(self, token: Annotated[str, Depends(get_token)]) -> bool:
        # __call__ сам зависит от get_token — FastAPI разрешит цепочку
        return token == self.allowed_token


allow_secret = PermissionChecker("secret-token")


@app.get("/protected/")
async def protected(ok: Annotated[bool, Depends(allow_secret)]):
    return {"granted": ok}
```

### 6. Переиспользование через тип-алиас (`Annotated`)

Чтобы не повторять `Annotated[bool, Depends(checker)]` во многих местах, заводим алиас:

```python
from typing import Annotated
from fastapi import Depends

# Готовый к переиспользованию тип
BarChecked = Annotated[bool, Depends(FixedContentQueryChecker("bar"))]


@app.get("/a/")
async def a(ok: BarChecked):
    return {"ok": ok}


@app.get("/b/")
async def b(ok: BarChecked):
    return {"ok": ok}
```

## Полный рабочий пример

```python
from typing import Annotated, Callable

from fastapi import Depends, FastAPI, HTTPException, Query, status
from pydantic import BaseModel

app = FastAPI(title="Продвинутые зависимости")


# --- 1. Параметризованная зависимость через __call__ -------------------------
class FixedContentQueryChecker:
    """Проверяет, что query-параметр q содержит фиксированную подстроку."""

    def __init__(self, fixed_content: str):
        self.fixed_content = fixed_content

    def __call__(self, q: Annotated[str, Query()] = "") -> bool:
        return bool(q) and self.fixed_content in q


check_bar = FixedContentQueryChecker("bar")
check_admin = FixedContentQueryChecker("admin")

# Переиспользуемые алиасы
BarChecked = Annotated[bool, Depends(check_bar)]
AdminChecked = Annotated[bool, Depends(check_admin)]


# --- 2. Фабрика зависимостей (closure) ---------------------------------------
def make_pagination(default_limit: int, max_limit: int) -> Callable[..., dict]:
    """Фабрика, создающая зависимость пагинации с разными лимитами."""

    def pagination(
        skip: Annotated[int, Query(ge=0)] = 0,
        limit: Annotated[int, Query(ge=1)] = default_limit,
    ) -> dict:
        if limit > max_limit:
            raise HTTPException(
                status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
                detail=f"limit не может превышать {max_limit}",
            )
        return {"skip": skip, "limit": limit}

    return pagination


# Две разные настройки пагинации из одной фабрики
small_pagination = make_pagination(default_limit=10, max_limit=50)
big_pagination = make_pagination(default_limit=100, max_limit=1000)

SmallPage = Annotated[dict, Depends(small_pagination)]
BigPage = Annotated[dict, Depends(big_pagination)]


# --- 3. Async-фабрика для проверки прав --------------------------------------
def require_query_contains(substr: str) -> Callable[..., "Awaitable[str]"]:
    async def checker(q: Annotated[str, Query()] = "") -> str:
        # тут мог бы быть await к БД/кешу
        if substr not in q:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail=f"q должен содержать {substr!r}",
            )
        return q

    return checker


# --- Модель ответа -----------------------------------------------------------
class Item(BaseModel):
    name: str
    matched: bool


# --- Эндпоинты ---------------------------------------------------------------
@app.get("/check-bar/")
async def check_bar_endpoint(matched: BarChecked):
    return Item(name="bar-check", matched=matched)


@app.get("/check-admin/")
async def check_admin_endpoint(matched: AdminChecked):
    return Item(name="admin-check", matched=matched)


@app.get("/items-small/")
async def items_small(page: SmallPage):
    return {"page": page, "items": [f"item-{i}" for i in range(page["limit"])]}


@app.get("/items-big/")
async def items_big(page: BigPage):
    return {"page": page, "items_count": page["limit"]}


@app.get("/strict/")
async def strict(q: Annotated[str, Depends(require_query_contains("hello"))]):
    return {"q": q}
```

Поведение:

- `GET /check-bar/?q=foobar` -> `{"name": "bar-check", "matched": true}`.
- `GET /items-small/?limit=999` -> 422 (лимит больше 50).
- `GET /items-big/?limit=999` -> ok (лимит до 1000).
- `GET /strict/?q=world` -> 400 (нет "hello").

## Частые вопросы на собеседовании (Q/A)

**1. Что может быть зависимостью в FastAPI?**
Любой "вызываемый" объект (callable): функция, корутина, класс или экземпляр класса с `__call__`. Для функции FastAPI смотрит на её параметры, для класса — на `__init__`, для экземпляра — на `__call__`.

**2. В чём разница между `Depends(SomeClass)` и `Depends(some_instance)`?**
`Depends(SomeClass)` — FastAPI вызовет конструктор класса и проанализирует параметры `__init__` как параметры запроса. `Depends(some_instance)` — экземпляр уже создан (с зашитой конфигурацией), FastAPI вызовет его как функцию и проанализирует параметры `__call__`.

**3. Зачем нужна параметризованная зависимость, если можно просто передать аргумент в функцию?**
Параметры обычной функции-зависимости FastAPI трактует как параметры **запроса** (query/path/header). Если нужно зафиксировать конфигурацию в момент объявления маршрута (а не получать её из запроса), используют либо экземпляр класса с `__call__`, либо фабрику-замыкание.

**4. Что такое фабрика зависимостей?**
Функция, которая принимает конфигурацию и возвращает другую функцию (или корутину) — настоящую зависимость. FastAPI анализирует сигнатуру именно возвращённой функции. Конфигурация сохраняется в замыкании.

**5. Класс с `__call__` или фабрика-замыкание — что выбрать?**
Замыкание — для простых случаев, лаконичнее. Класс — когда нужно сложное состояние, несколько методов, хранение ресурсов (клиент БД), наследование или более явная структура.

**6. Как `FixedContentQueryChecker` понимает, что `q` — это query-параметр?**
FastAPI анализирует сигнатуру `__call__`. Параметр `q: str = ""` без специального маркера трактуется как query-параметр (можно явно указать `Annotated[str, Query()]`).

**7. Можно ли в `__call__` использовать другие зависимости?**
Да. Внутри `__call__` (или возвращённой фабрикой функции) можно объявлять параметры с `Depends(...)`, и FastAPI рекурсивно разрешит всю цепочку.

**8. Кэшируются ли параметризованные зависимости в рамках одного запроса?**
Да, действует то же правило: одна и та же зависимость в рамках одного запроса по умолчанию вычисляется один раз (`use_cache=True`). Но два **разных** экземпляра класса считаются разными зависимостями и кэшируются отдельно.

**9. Как переиспользовать одну и ту же параметризованную зависимость в нескольких маршрутах?**
Через тип-алиас: `Checked = Annotated[bool, Depends(checker)]`, затем `def endpoint(ok: Checked)`.

**10. Может ли параметризованная зависимость быть асинхронной?**
Да. `__call__` можно объявить как `async def`, либо в фабрике вернуть `async def`. FastAPI корректно вызовет её через `await`.

**11. Где создавать настроенные экземпляры — внутри функции маршрута или на уровне модуля?**
На уровне модуля (или приложения). Создание экземпляра на каждый запрос бессмысленно и неэффективно — конфигурация фиксируется один раз.

**12. Как параметризованная зависимость отражается в Swagger UI?**
Её query/path/header-параметры (из `__call__` или возвращённой функции) автоматически попадают в схему OpenAPI и отображаются в Swagger UI как обычные параметры эндпоинта.

## Подводные камни (gotchas)

- **Путаница `__init__` vs `__call__`.** Если передать `Depends(MyClass)` (класс), FastAPI прочитает параметры `__init__` как параметры запроса. Чтобы зафиксировать конфигурацию, нужен именно **экземпляр** `Depends(MyClass("config"))`, и тогда читается `__call__`.
- **Создание экземпляра в сигнатуре каждый раз.** `Depends(FixedContentQueryChecker("bar"))` прямо в параметре эндпоинта создаёт новый экземпляр при импорте модуля — это нормально, но если делать так в нескольких местах с одной конфигурацией, лучше вынести в переменную/алиас ради читаемости и единства кэширования.
- **Замыкание захватывает изменяемую переменную.** В цикле-фабрике легко поймать классическую ошибку позднего связывания (late binding). Передавайте конфигурацию через аргумент функции, а не через внешнюю переменную цикла.
- **Забыли `Annotated`/маркер для не-query параметров.** Если параметр в `__call__` должен быть телом или заголовком, не забудьте `Header()`, `Body()` и т.д., иначе он станет query или (для сложных типов) телом.
- **Смешение sync и async.** Синхронная зависимость выполняется в threadpool; если внутри блокирующий I/O, это нормально, но для истинно асинхронного I/O делайте зависимость `async`.
- **Состояние в экземпляре класса разделяется между запросами.** Экземпляр создаётся один раз, поэтому изменяемые атрибуты (`self.counter += 1`) станут общим состоянием для всех запросов — потенциальная гонка. Храните в экземпляре только конфигурацию (read-only).

## Лучшие практики

- Конфигурацию зависимости держите неизменяемой и задавайте её один раз (в `__init__` или аргументах фабрики).
- Для переиспользования заводите типы-алиасы через `Annotated[..., Depends(...)]`.
- Выбирайте класс для сложного состояния/ресурсов, замыкание — для простых проверок.
- Делайте зависимость `async`, если внутри настоящий асинхронный I/O.
- Не храните per-request изменяемое состояние в экземпляре класса-зависимости.
- Бросайте `HTTPException` прямо из зависимости для отказов авторизации/валидации — это идиоматично и работает до входа в тело эндпоинта.
- Используйте говорящие имена настроенных экземпляров (`require_admin`, `small_pagination`).

## Шпаргалка

```python
# Класс как параметризованная зависимость
class Checker:
    def __init__(self, cfg): self.cfg = cfg            # конфигурация (один раз)
    def __call__(self, q: str = ""): ...               # параметры запроса (каждый раз)

checker = Checker("bar")                                # настроенный экземпляр
def ep(x: Annotated[bool, Depends(checker)]): ...       # читается __call__

# Фабрика зависимостей (замыкание)
def make_dep(cfg):
    def dep(q: str = ""): ...                           # настоящая зависимость
    return dep
my_dep = make_dep("bar")

# Async-зависимость
def make_async_dep(cfg):
    async def dep(...): ...                             # await внутри
    return dep

# Переиспользуемый алиас
Checked = Annotated[bool, Depends(checker)]

# Правила анализа сигнатуры:
#   Depends(func)            -> параметры func
#   Depends(SomeClass)       -> параметры __init__  (как параметры запроса!)
#   Depends(instance)        -> параметры __call__
```
