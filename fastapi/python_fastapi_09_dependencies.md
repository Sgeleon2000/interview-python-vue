# FastAPI — Зависимости (Dependency Injection) — конспект и вопросы

## О чём раздел

**Dependency Injection (DI, внедрение зависимостей)** — один из самых мощных и важных
механизмов FastAPI. Это центральная тема на собеседованиях уровня middle/senior, потому
что на ней строится практически всё реальное приложение: подключение к БД,
аутентификация пользователя, авторизация (проверка прав), пагинация, валидация общих
параметров, повторно используемая бизнес-логика.

В FastAPI зависимость — это **любой вызываемый объект** (функция, класс, асинхронная
функция, генератор), результат которого FastAPI «вычислит» до выполнения вашей
функции-обработчика и **передаст в неё как аргумент**. Вы объявляете зависимость через
`Depends(...)`, а FastAPI:

1. видит зависимость в сигнатуре обработчика;
2. вызывает её (передав туда нужные параметры из запроса/других зависимостей);
3. подставляет результат в ваш обработчик;
4. кеширует результат в рамках одного запроса (по умолчанию).

Главная идея: **«не вы зовёте зависимость, а фреймворк зовёт её за вас и отдаёт вам
готовый результат»** (это и есть «инверсия управления» / Inversion of Control).

Зачем это нужно:

- **Переиспользование** — одна и та же логика (например, получение текущего пользователя)
  используется в десятках эндпоинтов без копипаста.
- **Разделение ответственности** — обработчик занимается своей задачей, а получение
  ресурсов вынесено в отдельные функции.
- **Тестируемость** — любую зависимость можно подменить в тестах через
  `app.dependency_overrides` (мок БД, фейковый пользователь и т. д.).
- **Управление ресурсами** — зависимости с `yield` открывают ресурс (сессию БД) до
  обработчика и гарантированно закрывают его после ответа.
- **Документация** — параметры зависимостей автоматически попадают в OpenAPI/Swagger.

Современный стиль (которого придерживаемся в этом конспекте): **Pydantic v2**,
обязательный синтаксис `Annotated[Type, Depends(...)]`, `async/await`.

---

## Ключевые концепции

| Концепция | Краткое описание |
|---|---|
| `Depends(callable)` | Объявляет, что значение параметра нужно получить, вызвав `callable`. |
| `Annotated[T, Depends(f)]` | Современный способ объявить зависимость с типом `T`. |
| Функция-зависимость | Обычная (или `async`) функция, чей результат внедряется в обработчик. |
| Класс-зависимость | Класс, экземпляр которого создаётся вызовом `Depends(ClassName)`. |
| Сокращённый `Depends()` | Если класс уже указан в `Annotated`, аргумент в `Depends()` можно опустить. |
| Под-зависимости (sub-dependencies) | Зависимость может сама зависеть от других зависимостей — образуется дерево/граф. |
| `use_cache` | Кеширование результата зависимости в рамках одного запроса (по умолчанию `True`). |
| `dependencies=[...]` | Зависимости в декораторе пути — выполняются, но значение не передаётся в обработчик (для side-эффектов: проверка прав). |
| Глобальные зависимости | `FastAPI(dependencies=[...])` и `APIRouter(dependencies=[...])` — применяются ко всем маршрутам. |
| Зависимости с `yield` | Генераторы для управления ресурсами: код до `yield` — setup, после — teardown (очистка). |
| `app.dependency_overrides` | Подмена зависимостей в тестах. |

### Как FastAPI решает (resolve) зависимости

1. При старте приложения FastAPI анализирует сигнатуры обработчиков и строит **граф
   зависимостей** (зависимости, под-зависимости, под-под-зависимости...).
2. На каждый запрос FastAPI обходит граф, вычисляя зависимости в нужном порядке
   (сначала самые «глубокие»).
3. Для каждой зависимости вычисляются её собственные параметры (из query/path/body/других
   зависимостей).
4. Результаты внедряются в обработчик.
5. Зависимости с `yield` после формирования ответа «доигрываются» (teardown) в обратном
   порядке.

### Кеширование в рамках запроса (`use_cache`)

Если **одна и та же зависимость** используется несколько раз в графе одного запроса
(например, `get_current_user` нужна и эндпоинту, и под-зависимости проверки прав), FastAPI
вызовет её **только один раз** и переиспользует результат. Это и есть кеш в рамках запроса.
Отключается через `Depends(dep, use_cache=False)` — тогда зависимость будет вызвана заново.

---

## Подробный разбор с примерами кода

### 1. Простейшая функция-зависимость и общие параметры

Классический случай — общие query-параметры пагинации, которые нужны во многих эндпоинтах.

```python
from typing import Annotated
from fastapi import FastAPI, Depends

app = FastAPI()


# Зависимость — обычная async-функция с общими query-параметрами.
async def common_parameters(
    q: str | None = None,      # необязательный поисковый запрос
    skip: int = 0,             # сколько элементов пропустить (offset)
    limit: int = 100,          # максимум элементов в ответе
) -> dict:
    # Возвращаем словарь — это и будет внедрено в обработчик.
    return {"q": q, "skip": skip, "limit": limit}


# Создаём тип-алиас, чтобы не повторять Annotated[...] в каждом эндпоинте.
CommonsDep = Annotated[dict, Depends(common_parameters)]


@app.get("/items/")
async def read_items(commons: CommonsDep):
    # commons уже содержит {"q": ..., "skip": ..., "limit": ...}
    return {"message": "items", "params": commons}


@app.get("/users/")
async def read_users(commons: CommonsDep):
    # Тот же набор параметров переиспользован без копипаста.
    return {"message": "users", "params": commons}
```

Важные моменты:

- Параметры `q`, `skip`, `limit` зависимости автоматически становятся **query-параметрами
  эндпоинта** и попадают в Swagger.
- `Annotated[dict, Depends(common_parameters)]` — современный синтаксис. Тип (`dict`)
  служит для аннотации того, что вернёт зависимость.
- Алиас `CommonsDep = Annotated[...]` — рекомендуемый приём для устранения дублирования.

### 2. Зависимость может быть синхронной или асинхронной

FastAPI поддерживает оба варианта и **смешивает их свободно**: `async`-эндпоинт может
зависеть от обычной `def`-зависимости и наоборот. Синхронные зависимости выполняются в
пуле потоков, чтобы не блокировать event loop.

```python
# Синхронная зависимость — FastAPI выполнит её в threadpool.
def sync_dep() -> str:
    return "sync"


# Асинхронная зависимость — выполнится прямо в event loop.
async def async_dep() -> str:
    return "async"
```

### 3. Классы как зависимости

Класс — это тоже callable (вызов `ClassName(...)` создаёт экземпляр через `__init__`).
Поэтому класс можно передать в `Depends`. Параметры `__init__` станут параметрами запроса.
Это удобнее словаря: вы получаете **типизированный объект** с автодополнением в IDE.

```python
from typing import Annotated
from fastapi import Depends, FastAPI

app = FastAPI()

fake_items_db = [{"name": "Foo"}, {"name": "Bar"}, {"name": "Baz"}]


class CommonQueryParams:
    # Параметры __init__ автоматически становятся query-параметрами эндпоинта.
    def __init__(
        self,
        q: str | None = None,
        skip: int = 0,
        limit: int = 100,
    ):
        self.q = q
        self.skip = skip
        self.limit = limit


@app.get("/items/")
async def read_items(
    # Полная запись: и тип, и Depends с классом.
    commons: Annotated[CommonQueryParams, Depends(CommonQueryParams)],
):
    response: dict = {}
    if commons.q:
        response.update({"q": commons.q})
    items = fake_items_db[commons.skip : commons.skip + commons.limit]
    response.update({"items": items})
    return response
```

#### Сокращённая запись `Depends()` без аргумента

Если **тип в `Annotated` уже совпадает с тем, что нужно вызвать** (как у классов), FastAPI
позволяет не повторять имя класса внутри `Depends()` — он возьмёт его из аннотации.

```python
@app.get("/items-short/")
async def read_items_short(
    # Эквивалентно Depends(CommonQueryParams) — FastAPI берёт класс из Annotated.
    commons: Annotated[CommonQueryParams, Depends()],
):
    return {"q": commons.q, "skip": commons.skip, "limit": commons.limit}
```

> Эта сокращённая форма работает **только** когда зависимость — это сам класс из аннотации.
> Для функций так делать нельзя (там тип возврата не равен callable).

### 4. Вложенные под-зависимости (sub-dependencies)

Зависимость может сама иметь зависимости. FastAPI построит граф и вычислит всё в нужном
порядке. Пример: `query_or_cookie_extractor` зависит от `query_extractor`.

```python
from typing import Annotated
from fastapi import Cookie, Depends, FastAPI

app = FastAPI()


# Под-зависимость первого уровня: достаёт query-параметр q.
def query_extractor(q: str | None = None) -> str | None:
    return q


# Зависимость, которая зависит от query_extractor (под-зависимость).
def query_or_cookie_extractor(
    q: Annotated[str | None, Depends(query_extractor)],  # вложенная зависимость
    last_query: Annotated[str | None, Cookie()] = None,  # значение из cookie
) -> str:
    # Если q не передан, берём последний запрос из cookie.
    if not q:
        return last_query
    return q


@app.get("/items/")
async def read_query(
    query_or_default: Annotated[str, Depends(query_or_cookie_extractor)],
):
    return {"q_or_cookie": query_or_default}
```

Что произойдёт на запрос:

1. FastAPI увидит, что обработчику нужен `query_or_cookie_extractor`.
2. Чтобы его вызвать, нужен `query_extractor` → сначала вызывается он.
3. Результат `query_extractor` (плюс cookie) передаётся в `query_or_cookie_extractor`.
4. Финальный результат внедряется в обработчик.

Если `query_extractor` используется ещё где-то в этом же запросе — он **не будет вызван
повторно** (кеш `use_cache`).

### 5. Зависимости в декораторе пути (`dependencies=[...]`)

Иногда зависимость нужна ради **побочного эффекта** (проверить токен, заголовок, права), а
её возвращаемое значение в обработчике не используется. Тогда её объявляют в декораторе
пути через параметр `dependencies`. Зависимость будет выполнена (и может бросить
исключение), но значение никуда не передаётся.

```python
from typing import Annotated
from fastapi import Depends, FastAPI, Header, HTTPException

app = FastAPI()


async def verify_token(x_token: Annotated[str, Header()]) -> None:
    # Проверяем кастомный заголовок. Ничего не возвращаем.
    if x_token != "fake-super-secret-token":
        raise HTTPException(status_code=400, detail="Неверный X-Token заголовок")


async def verify_key(x_key: Annotated[str, Header()]) -> str:
    if x_key != "fake-super-secret-key":
        raise HTTPException(status_code=400, detail="Неверный X-Key заголовок")
    return x_key  # даже если вернёт — значение не будет внедрено в обработчик


@app.get(
    "/items/",
    # Зависимости-«стражи». Выполнятся до обработчика, значения не передаются.
    dependencies=[Depends(verify_token), Depends(verify_key)],
)
async def read_items():
    return [{"item": "Foo"}, {"item": "Bar"}]
```

### 6. Глобальные зависимости (приложение и роутер)

Зависимости можно применить **ко всем маршрутам приложения** или **ко всем маршрутам
роутера**. Удобно для сквозной аутентификации, общих заголовков, телеметрии.

```python
from fastapi import Depends, FastAPI, APIRouter

# 1) Глобально на всё приложение:
app = FastAPI(dependencies=[Depends(verify_token), Depends(verify_key)])


@app.get("/items/")
async def read_items():
    # verify_token и verify_key уже применены автоматически.
    return [{"item": "Portal Gun"}]


# 2) Глобально на конкретный роутер:
router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(verify_token)],  # применяется ко всем маршрутам роутера
)


@router.get("/dashboard")
async def dashboard():
    return {"page": "admin dashboard"}


app.include_router(router)
```

Порядок применения: сначала глобальные зависимости приложения, затем зависимости роутера,
затем зависимости конкретного декоратора пути, затем зависимости из сигнатуры обработчика.

### 7. Зависимости с `yield` — управление ресурсами

Самый важный для собеседования механизм. Зависимость-генератор позволяет выполнить код
**до** отдачи значения (setup) и **после** ответа (teardown — очистка). Идеально для
сессий БД, файлов, соединений.

```python
from typing import Annotated, AsyncGenerator
from fastapi import Depends, FastAPI

app = FastAPI()


# Псевдо-сессия БД для иллюстрации.
class DBSession:
    def __init__(self):
        self.closed = False

    def close(self):
        self.closed = True


async def get_db() -> AsyncGenerator[DBSession, None]:
    db = DBSession()          # setup: открываем "сессию"
    try:
        yield db              # отдаём сессию в обработчик; здесь FastAPI приостанавливается
    finally:
        db.close()            # teardown: закрываем сессию ПОСЛЕ ответа клиенту


DBDep = Annotated[DBSession, Depends(get_db)]


@app.get("/items/")
async def read_items(db: DBDep):
    # Здесь db — открытая сессия. После возврата ответа сработает finally (db.close()).
    return {"db_closed": db.closed}  # False, пока запрос обрабатывается
```

Ключевые правила зависимостей с `yield`:

- **Код до `yield`** выполняется до обработчика — это инициализация ресурса.
- **`yield value`** — отдаёт `value` как значение зависимости.
- **Код после `yield`** (обычно в `finally`) выполняется **после** того, как ответ
  сформирован и отправляется. Используйте `try/finally`, чтобы очистка случилась даже при
  исключении.
- Можно использовать обычные `def` с `yield` (синхронный генератор) и `async def` с `yield`
  (асинхронный генератор) — FastAPI поддерживает оба.

#### Обработка исключений в зависимостях с `yield`

Если в обработчике (или в под-зависимости ниже) возникает исключение, оно «пробрасывается»
в место `yield`. Это позволяет, например, откатить транзакцию:

```python
async def get_db_with_rollback() -> AsyncGenerator[DBSession, None]:
    db = DBSession()
    try:
        yield db
        # сюда дойдём только если НЕ было исключения — можно сделать commit
        # await db.commit()
    except Exception:
        # исключение из обработчика прилетит сюда — откатываем транзакцию
        # await db.rollback()
        raise          # ВАЖНО: пробрасываем дальше, иначе FastAPI не вернёт 500
    finally:
        db.close()     # закрываем сессию в любом случае
```

Нюанс: начиная с современных версий FastAPI, **можно ловить `HTTPException` и другие
исключения внутри `except`**, но если вы хотите, чтобы клиент получил корректный ответ об
ошибке, исключение нужно либо обработать корректно, либо пробросить (`raise`). Нельзя
поднимать новое `HTTPException` уже **после** `yield`, рассчитывая, что оно вернётся
клиенту — на этом этапе ответ уже частично сформирован.

#### Порядок выполнения teardown (несколько `yield`-зависимостей)

Если зависимостей с `yield` несколько и они вложены, teardown выполняется в **обратном
порядке** (как стек / как вложенные контекстные менеджеры):

```python
# Граф: dep_c зависит от dep_b, dep_b зависит от dep_a.
# Setup-порядок:    A -> B -> C
# Teardown-порядок: C -> B -> A
```

#### Контекстные менеджеры внутри зависимостей с `yield`

Можно использовать `with` / `async with` прямо в зависимости — FastAPI корректно
отработает выход из контекста на этапе teardown:

```python
async def get_db_cm() -> AsyncGenerator[DBSession, None]:
    # async with гарантирует корректное закрытие даже при исключении.
    async with open_async_session() as session:  # псевдокод
        yield session
    # выход из "async with" произойдёт после ответа
```

---

## Полный рабочий пример (с `get_db` и `get_current_user`)

Ниже самодостаточный пример, который собирает воедино: сессию БД через `yield`,
аутентификацию пользователя, проверку прав, пагинацию и под-зависимости. Это близко к
тому, как выглядит реальный production-код.

```python
from typing import Annotated, AsyncGenerator

from fastapi import Depends, FastAPI, HTTPException, Query, status
from fastapi.security import OAuth2PasswordBearer
from pydantic import BaseModel

app = FastAPI()

# ──────────────────────────── Модели (Pydantic v2) ────────────────────────────


class User(BaseModel):
    id: int
    username: str
    is_active: bool = True
    is_admin: bool = False


# ──────────────────────────── Фейковая "база данных" ──────────────────────────


class FakeDB:
    """Псевдо-сессия БД. В реальности это AsyncSession из SQLAlchemy."""

    def __init__(self) -> None:
        self._users = {
            "alice": User(id=1, username="alice", is_admin=True),
            "bob": User(id=2, username="bob", is_admin=False),
            "eve": User(id=3, username="eve", is_active=False),
        }
        self.closed = False

    async def get_user_by_token(self, token: str) -> User | None:
        # Упрощённо: токен совпадает с username.
        return self._users.get(token)

    async def list_items(self, skip: int, limit: int) -> list[dict]:
        all_items = [{"id": i, "name": f"item-{i}"} for i in range(1, 51)]
        return all_items[skip : skip + limit]

    def close(self) -> None:
        self.closed = True


# ──────────────────────────── Зависимость: сессия БД ──────────────────────────


async def get_db() -> AsyncGenerator[FakeDB, None]:
    db = FakeDB()           # setup: открыли соединение
    try:
        yield db            # отдали сессию обработчику/под-зависимостям
    finally:
        db.close()          # teardown: закрыли после ответа


DBDep = Annotated[FakeDB, Depends(get_db)]


# ──────────────────────── Зависимость: текущий пользователь ───────────────────

# Извлекает токен из заголовка Authorization: Bearer <token>.
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")


async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],  # под-зависимость: достаёт токен
    db: DBDep,                                       # под-зависимость: сессия БД (кеш!)
) -> User:
    user = await db.get_user_by_token(token)
    if user is None:
        # 401 — не аутентифицирован.
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Не удалось проверить учётные данные",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return user


CurrentUser = Annotated[User, Depends(get_current_user)]


# Зависимость поверх зависимости: только активный пользователь.
async def get_current_active_user(current_user: CurrentUser) -> User:
    if not current_user.is_active:
        raise HTTPException(status_code=400, detail="Пользователь неактивен")
    return current_user


ActiveUser = Annotated[User, Depends(get_current_active_user)]


# Зависимость-«страж»: проверка прав администратора.
async def require_admin(current_user: ActiveUser) -> User:
    if not current_user.is_admin:
        # 403 — аутентифицирован, но недостаточно прав.
        raise HTTPException(status_code=403, detail="Требуются права администратора")
    return current_user


AdminUser = Annotated[User, Depends(require_admin)]


# ──────────────────────────── Зависимость: пагинация ──────────────────────────


class Pagination(BaseModel):
    skip: int = 0
    limit: int = 20


async def pagination_params(
    skip: Annotated[int, Query(ge=0)] = 0,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
) -> Pagination:
    return Pagination(skip=skip, limit=limit)


PaginationDep = Annotated[Pagination, Depends(pagination_params)]


# ──────────────────────────────── Эндпоинты ───────────────────────────────────


@app.get("/users/me", response_model=User)
async def read_me(current_user: ActiveUser):
    # Возвращаем текущего активного пользователя.
    return current_user


@app.get("/items/")
async def read_items(
    db: DBDep,                 # сессия БД (тот же объект, что и в get_current_user — кеш)
    current_user: ActiveUser,  # требуется активный пользователь
    pagination: PaginationDep, # пагинация из query-параметров
):
    items = await db.list_items(pagination.skip, pagination.limit)
    return {"owner": current_user.username, "items": items}


@app.delete(
    "/items/{item_id}",
    # Значение admin-зависимости здесь не нужно — используем dependencies для проверки прав.
    dependencies=[Depends(require_admin)],
)
async def delete_item(item_id: int):
    # Сюда попадём только если require_admin не бросил исключение.
    return {"deleted": item_id}
```

Обратите внимание: в `get_current_user` и в `read_items` используется одна и та же
зависимость `get_db`. Благодаря кешу `use_cache=True` FastAPI создаст **одну** сессию
`FakeDB` на весь запрос и переиспользует её — это критично, чтобы не плодить соединения.

---

## Частые вопросы на собеседовании (Q/A)

**1. Что такое Dependency Injection в FastAPI и какую проблему он решает?**
DI — механизм, при котором FastAPI сам вычисляет зависимости (вызываемые объекты) и
внедряет их результат в обработчик. Решает проблемы переиспользования логики (общие
параметры, аутентификация), разделения ответственности, управления ресурсами и
тестируемости (через подмену зависимостей).

**2. Что может быть зависимостью?**
Любой **вызываемый объект**: обычная функция, `async`-функция, класс (создаётся экземпляр),
функция-генератор с `yield` (синхронная или асинхронная). Главное — FastAPI должен уметь
его вызвать.

**3. Как объявить зависимость современным синтаксисом?**
Через `Annotated[ТипВозврата, Depends(callable)]` в аннотации параметра. Рекомендуется
выносить в тип-алиас: `CurrentUser = Annotated[User, Depends(get_current_user)]`.

**4. Что такое кеширование зависимостей и зачем оно?**
Если одна зависимость встречается несколько раз в графе одного запроса, FastAPI вызовет её
**один раз** и переиспользует результат (по умолчанию `use_cache=True`). Это экономит
ресурсы (например, одно соединение с БД) и обеспечивает консистентность. Отключается:
`Depends(dep, use_cache=False)`.

**5. Чем отличается зависимость в сигнатуре от `dependencies=[...]` в декораторе?**
В сигнатуре — результат **внедряется** в обработчик как аргумент. В `dependencies=[...]` —
зависимость выполняется ради побочного эффекта (проверка токена/прав), а её значение
**никуда не передаётся**. Используется, когда значение не нужно.

**6. Как сделать зависимость глобальной?**
На всё приложение: `FastAPI(dependencies=[Depends(...)])`. На роутер:
`APIRouter(dependencies=[Depends(...)])`. Применяется ко всем маршрутам автоматически.

**7. Что такое зависимость с `yield` и зачем она нужна?**
Это генератор: код до `yield` — инициализация ресурса (setup), `yield` отдаёт значение,
код после `yield` (в `finally`) — очистка (teardown). Нужна для управления ресурсами:
сессии БД, файлы, соединения. Гарантирует закрытие ресурса после ответа.

**8. Когда именно выполняется код после `yield`?**
После того, как обработчик вернул результат и FastAPI сформировал ответ — то есть на этапе
завершения запроса. Именно поэтому туда помещают закрытие сессии/коммит/откат.

**9. Как обрабатываются исключения в зависимостях с `yield`?**
Если в обработчике/под-зависимости возникло исключение, оно «прилетает» в точку `yield`.
Можно поймать его в `except` (например, сделать `rollback`) и обязательно `raise` дальше,
чтобы FastAPI вернул корректный ответ об ошибке. `finally` отрабатывает всегда.

**10. В каком порядке выполняется teardown нескольких `yield`-зависимостей?**
В обратном порядку setup — как стек вложенных контекстных менеджеров. Если setup был
A → B → C, то teardown будет C → B → A.

**11. Можно ли смешивать `async def` и `def` зависимости?**
Да. `async`-обработчик может зависеть от синхронной `def`-зависимости и наоборот.
Синхронные зависимости FastAPI выполняет в пуле потоков, чтобы не блокировать event loop.

**12. Как класс используется в качестве зависимости и что даёт сокращённый `Depends()`?**
`Depends(MyClass)` вызывает `MyClass(...)`, параметры `__init__` становятся параметрами
запроса, в обработчик попадает экземпляр (типизированный объект). Если класс уже указан в
`Annotated`, аргумент в `Depends()` можно опустить:
`Annotated[MyClass, Depends()]` — FastAPI возьмёт класс из аннотации.

**13. Что такое под-зависимости (sub-dependencies)?**
Зависимость может сама объявлять `Depends(...)` в своей сигнатуре — образуется граф.
FastAPI вычисляет их в нужном порядке (сначала самые глубокие) с учётом кеша.

**14. Как реализовать аутентификацию через зависимости?**
Зависимость `get_current_user` извлекает токен (например, через `OAuth2PasswordBearer`),
проверяет его в БД и возвращает пользователя или бросает `HTTPException(401)`. Поверх неё
строят `get_current_active_user`, `require_admin` и т. д.

**15. Чем отличается 401 от 403 в зависимостях-стражах?**
`401 Unauthorized` — пользователь не аутентифицирован (нет/невалиден токен). `403 Forbidden`
— аутентифицирован, но не имеет прав на действие (например, не админ).

**16. Как подменить зависимость в тестах?**
Через `app.dependency_overrides[оригинальная_зависимость] = подмена`. Например, подменить
`get_db` на тестовую сессию, а `get_current_user` — на фейкового пользователя. После теста
очистить: `app.dependency_overrides.clear()`. (Подробнее — в теме «Тестирование».)

**17. Почему важно выносить `Annotated[...]` в алиас?**
Чтобы избежать дублирования и опечаток: один источник истины
(`CurrentUser = Annotated[User, Depends(get_current_user)]`), используемый во всех
эндпоинтах. Упрощает рефакторинг.

**18. Можно ли в `yield`-зависимости после `yield` поднять `HTTPException`, чтобы вернуть
ошибку клиенту?**
Нет, надёжно вернуть HTTP-ответ из кода после `yield` нельзя — ответ уже сформирован.
Проверки, влияющие на ответ, делайте **до** `yield`. После `yield` — только очистка
(`close`, `rollback`, `commit`).

**19. Зависят ли зависимости от порядка объявления в декораторе?**
Зависимости в `dependencies=[...]` выполняются в порядке списка. Глобальные → роутерные →
декораторные → из сигнатуры. Внутри графа порядок определяется зависимостями друг от друга.

**20. Как зависимости попадают в документацию OpenAPI?**
Параметры зависимостей (query/path/header/cookie) автоматически отражаются в схеме
OpenAPI и видны в Swagger UI, как если бы они были объявлены прямо в эндпоинте.

---

## Подводные камни (gotchas)

- **Забыли `use_cache` и ждёте повторный вызов.** По умолчанию зависимость кешируется в
  рамках запроса. Если действительно нужно вызвать её дважды с разным результатом —
  `use_cache=False`.
- **Открытие нескольких сессий БД.** Если из-за `use_cache=False` или разных функций
  `get_db` создаётся несколько сессий — это утечка соединений. Используйте одну зависимость
  `get_db` повсеместно.
- **Проверки после `yield`.** Любые проверки, которые должны вернуть ошибку клиенту,
  делайте **до** `yield`. После `yield` ответ уже сформирован.
- **Забытый `raise` в `except` у `yield`-зависимости.** Если поймали исключение для
  отката, но не пробросили — FastAPI «проглотит» ошибку и не вернёт 500. Всегда `raise`.
- **Блокирующий код в `async`-зависимости.** Синхронный блокирующий вызов (например,
  тяжёлый IO без `await`) внутри `async def` заблокирует event loop. Либо делайте
  зависимость обычной `def` (пойдёт в threadpool), либо используйте `await`/`run_in_executor`.
- **Сокращённый `Depends()` для функций.** Работает только для классов из `Annotated`. Для
  функций нужно явно: `Depends(my_func)`.
- **`HTTPException` из глобальной зависимости.** Глобальные зависимости применяются ко
  **всем** маршрутам, включая служебные. Будьте осторожны, навешивая туда аутентификацию —
  можно случайно закрыть `/docs` или health-check.
- **Зависимости с `yield` и фоновые задачи (BackgroundTasks).** Teardown
  `yield`-зависимости выполняется **до** фоновых задач в большинстве версий — не
  рассчитывайте на открытую сессию БД внутри background task, переданную из зависимости;
  открывайте ресурс внутри самой задачи.
- **Изменяемые значения по умолчанию.** Не используйте mutable default (`def dep(x=[])`) —
  стандартная ловушка Python.
- **Тип в `Annotated` не влияет на вызов.** FastAPI вызывает то, что в `Depends(...)`, а
  тип — лишь аннотация результата. Несоответствие типа и реального возврата не вызовет
  ошибку, но запутает читателя и IDE.

---

## Лучшие практики

- Используйте **`Annotated[Type, Depends(...)]`** везде — это рекомендованный современный
  стиль (лучше типизация, переиспользование, совместимость).
- **Выносите зависимости в алиасы:** `CurrentUser`, `DBDep`, `PaginationDep` — один
  источник истины, меньше дублирования.
- **Стройте зависимости слоями:** `get_db` → `get_current_user` → `get_current_active_user`
  → `require_admin`. Каждый слой добавляет одну проверку.
- Для управления ресурсами **всегда используйте `yield` с `try/finally`** (или
  `async with`), чтобы гарантировать очистку.
- Для **проверок прав без возвращаемого значения** используйте
  `dependencies=[Depends(...)]` в декораторе пути.
- **Сквозную логику** (аутентификация на закрытой части API, телеметрия) выносите в
  глобальные/роутерные зависимости, но не закрывайте случайно `/docs` и health-эндпоинты.
- Пишите зависимости **тестируемыми**: они должны легко подменяться через
  `app.dependency_overrides` (избегайте глобального состояния, скрытых импортов внутри).
- **Не делайте зависимости тяжёлыми без необходимости** — они выполняются на каждый запрос.
  Кешируйте дорогие неизменяемые вычисления (`lru_cache` для конфигов/настроек).
- Различайте **401 (не аутентифицирован)** и **403 (нет прав)** — корректные коды важны.
- Держите зависимости **чистыми по ответственности**: одна зависимость — одна задача
  (получить сессию / получить пользователя / проверить право), а не всё сразу.

---

## Шпаргалка

```python
from typing import Annotated, AsyncGenerator
from fastapi import Depends, FastAPI, APIRouter, HTTPException, Query, status

# ── Функция-зависимость + алиас ───────────────────────────────────────────────
async def pagination(skip: int = 0, limit: Annotated[int, Query(le=100)] = 20):
    return {"skip": skip, "limit": limit}

Pg = Annotated[dict, Depends(pagination)]

@app.get("/a")
async def a(p: Pg): ...                      # p внедрён как результат pagination()

# ── Класс как зависимость + сокращённая запись ────────────────────────────────
class Common:
    def __init__(self, q: str | None = None): self.q = q

@app.get("/b")
async def b(c: Annotated[Common, Depends()]): ...   # Depends() без аргумента (класс из Annotated)

# ── Под-зависимости (граф) ────────────────────────────────────────────────────
def base(q: str | None = None): return q
def derived(v: Annotated[str | None, Depends(base)]): return v or "default"

# ── Зависимость с yield (ресурс) ──────────────────────────────────────────────
async def get_db() -> AsyncGenerator[object, None]:
    db = open_session()          # setup до yield
    try:
        yield db                 # значение зависимости
    finally:
        db.close()               # teardown после ответа

DB = Annotated[object, Depends(get_db)]

# ── Зависимость-страж (значение не нужно) ─────────────────────────────────────
async def verify(x_token: str): 
    if x_token != "secret": raise HTTPException(400)

@app.get("/c", dependencies=[Depends(verify)])      # выполняется, значение не передаётся
async def c(): ...

# ── Глобальные зависимости ────────────────────────────────────────────────────
app = FastAPI(dependencies=[Depends(verify)])        # на всё приложение
router = APIRouter(dependencies=[Depends(verify)])   # на весь роутер

# ── Отключить кеш в рамках запроса ────────────────────────────────────────────
async def d(x: Annotated[int, Depends(dep, use_cache=False)]): ...

# ── Подмена в тестах ──────────────────────────────────────────────────────────
app.dependency_overrides[get_db] = lambda: fake_db
# ... после теста:
app.dependency_overrides.clear()
```

**Главное, что нужно помнить:**

1. Зависимость = любой callable; объявляется через `Annotated[T, Depends(f)]`.
2. FastAPI строит **граф** зависимостей и вычисляет их за вас.
3. По умолчанию результат **кешируется** в рамках запроса (`use_cache=True`).
4. `dependencies=[...]` — для зависимостей без возвращаемого значения (проверки).
5. **Глобальные** зависимости — на приложение и на роутер.
6. **`yield`** — управление ресурсами: setup до, teardown (в `finally`) после ответа.
7. Исключение из обработчика прилетает в `yield` → можно `rollback`, но обязательно `raise`.
8. Тесты: подмена через `app.dependency_overrides`.
