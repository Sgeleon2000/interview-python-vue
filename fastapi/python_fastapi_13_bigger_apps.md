# FastAPI — Структура большого приложения — конспект и вопросы

## О чём раздел

По мере роста приложения держать все эндпоинты в одном файле становится неудобно. FastAPI предлагает **`APIRouter`** — «мини-приложение», в которое выносят группу связанных маршрутов. Роутеры объявляются в отдельных модулях и подключаются к главному приложению через `include_router`. Это даёт модульность, переиспользование зависимостей и чистую структуру проекта.

В разделе разберём: `APIRouter` и его параметры (`prefix`, `tags`, `dependencies`, `responses`), `include_router`, организацию проекта (`routers/`, `models/`, `dependencies/`), относительные импорты, глобальные и роутер-уровневые зависимости, переиспользование кода.

## Ключевые концепции

- **`APIRouter`** — контейнер маршрутов, подключаемый к `FastAPI` или другому роутеру.
- **`prefix`** — общий префикс пути для всех маршрутов роутера (без завершающего слеша).
- **`tags`** — группировка эндпоинтов в Swagger UI.
- **`dependencies`** — зависимости, применяемые ко всем маршрутам роутера (без внедрения значения).
- **`responses`** — дополнительные описания ответов для документации.
- **`include_router`** — подключение роутера к приложению (можно переопределить prefix/tags/dependencies).
- **Уровни зависимостей** — приложение -> роутер -> эндпоинт.
- **Относительные импорты** — `from ..dependencies import ...` внутри пакета.

## Подробный разбор с примерами кода

### 1. Рекомендуемая структура проекта

```
app/
├── __init__.py
├── main.py                # создаёт FastAPI, подключает роутеры
├── dependencies.py        # общие зависимости (auth, БД, пагинация)
├── config.py              # настройки (pydantic-settings)
├── database.py            # engine, get_session
├── models/
│   ├── __init__.py
│   ├── user.py
│   └── item.py
├── routers/
│   ├── __init__.py
│   ├── users.py           # APIRouter для пользователей
│   └── items.py           # APIRouter для товаров
└── internal/
    └── admin.py           # внутренние/админские роуты
```

Каждая папка с `__init__.py` — это пакет, что включает относительные импорты.

### 2. APIRouter с параметрами

```python
# app/routers/items.py
from typing import Annotated
from fastapi import APIRouter, Depends, HTTPException
from ..dependencies import get_token_header  # относительный импорт

router = APIRouter(
    prefix="/items",                       # все пути начинаются с /items
    tags=["items"],                        # группа в Swagger
    dependencies=[Depends(get_token_header)],  # проверка на каждом маршруте
    responses={404: {"description": "Не найдено"}},  # для документации
)

fake_items_db = {"plumbus": {"name": "Plumbus"}, "gun": {"name": "Portal Gun"}}


@router.get("/")
async def read_items() -> dict:
    return fake_items_db


@router.get("/{item_id}")
async def read_item(item_id: str) -> dict:
    if item_id not in fake_items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    return {"item_id": item_id, **fake_items_db[item_id]}


@router.put(
    "/{item_id}",
    tags=["custom"],                       # доп. тег только для этого маршрута
    responses={403: {"description": "Запрещено"}},
)
async def update_item(item_id: str) -> dict:
    if item_id != "plumbus":
        raise HTTPException(status_code=403, detail="You can only update plumbus")
    return {"item_id": item_id, "name": "The great Plumbus"}
```

Параметры в `APIRouter(...)` применяются ко всем маршрутам роутера; на уровне отдельного маршрута их можно дополнить (например, добавить тег).

### 3. Общие зависимости в отдельном модуле

```python
# app/dependencies.py
from typing import Annotated
from fastapi import Header, HTTPException


async def get_token_header(x_token: Annotated[str, Header()]) -> None:
    # Зависимость без возврата значения — проверка заголовка
    if x_token != "fake-super-secret-token":
        raise HTTPException(status_code=400, detail="X-Token header invalid")


async def get_query_token(token: str) -> None:
    if token != "jessica":
        raise HTTPException(status_code=400, detail="No Jessica token provided")


# Переиспользуемая пагинация
async def pagination_params(
    skip: int = 0, limit: int = 100
) -> dict[str, int]:
    return {"skip": skip, "limit": limit}


PaginationDep = Annotated[dict[str, int], Depends(pagination_params)]
```

### 4. Подключение роутеров в main.py

```python
# app/main.py
from fastapi import Depends, FastAPI
from .dependencies import get_query_token
from .routers import items, users
from .internal import admin

# Глобальная зависимость — применяется ко ВСЕМ маршрутам приложения
app = FastAPI(dependencies=[Depends(get_query_token)])

# Простое подключение (prefix/tags берутся из самого роутера)
app.include_router(users.router)
app.include_router(items.router)

# Подключение с переопределением: добавим prefix, tags, dependencies, responses
app.include_router(
    admin.router,
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(get_token_header)],
    responses={418: {"description": "I'm a teapot"}},
)


@app.get("/")
async def root() -> dict:
    return {"message": "Hello Bigger Applications!"}
```

Важно: `include_router` **не изменяет** оригинальный роутер, а создаёт копию его маршрутов с применёнными параметрами. Поэтому один роутер можно подключить несколько раз с разными префиксами.

### 5. Три уровня зависимостей

```python
# 1) Уровень приложения — на все запросы
app = FastAPI(dependencies=[Depends(get_query_token)])

# 2) Уровень роутера — на все маршруты роутера
router = APIRouter(dependencies=[Depends(get_token_header)])
# или при подключении:
app.include_router(router, dependencies=[Depends(get_token_header)])

# 3) Уровень эндпоинта — на конкретный маршрут
@router.get("/secret", dependencies=[Depends(verify_admin)])
async def secret() -> dict:
    return {"ok": True}
```

Зависимости из `dependencies=[...]` выполняются ради побочного эффекта (проверки), их значение не внедряется в обработчик. Порядок: сначала зависимости приложения, потом роутера, потом эндпоинта.

### 6. Относительные импорты

```python
from .dependencies import get_token_header     # тот же пакет (app/)
from ..dependencies import get_token_header    # на уровень выше
from ...models.user import User                # на два уровня выше
```

Точки означают «текущий пакет» (`.`), «родительский» (`..`) и т.д. Относительные импорты работают только внутри пакетов (есть `__init__.py`) и при запуске как модуль (`uvicorn app.main:app`), а не `python routers/items.py`.

### 7. Вложенные роутеры

Роутер можно подключить к другому роутеру для иерархии:

```python
# app/routers/users.py
from fastapi import APIRouter
from .profile import router as profile_router

router = APIRouter(prefix="/users", tags=["users"])
router.include_router(profile_router)  # итоговый путь: /users/profile/...
```

## Полный рабочий пример

```python
# app/dependencies.py
from typing import Annotated
from fastapi import Depends, Header, HTTPException


async def get_token_header(x_token: Annotated[str, Header()]) -> None:
    if x_token != "fake-super-secret-token":
        raise HTTPException(status_code=400, detail="X-Token header invalid")


async def pagination_params(skip: int = 0, limit: int = 100) -> dict[str, int]:
    return {"skip": skip, "limit": limit}


PaginationDep = Annotated[dict[str, int], Depends(pagination_params)]
```

```python
# app/routers/users.py
from fastapi import APIRouter
from ..dependencies import PaginationDep

router = APIRouter(prefix="/users", tags=["users"])

fake_users = [{"username": "alice"}, {"username": "bob"}, {"username": "carol"}]


@router.get("/")
async def read_users(pagination: PaginationDep) -> list[dict]:
    skip, limit = pagination["skip"], pagination["limit"]
    return fake_users[skip : skip + limit]


@router.get("/me")
async def read_current_user() -> dict:
    return {"username": "current"}


@router.get("/{username}")
async def read_user(username: str) -> dict:
    return {"username": username}
```

```python
# app/routers/items.py
from fastapi import APIRouter, Depends, HTTPException
from ..dependencies import get_token_header, PaginationDep

router = APIRouter(
    prefix="/items",
    tags=["items"],
    dependencies=[Depends(get_token_header)],
    responses={404: {"description": "Не найдено"}},
)

fake_items = {"plumbus": {"name": "Plumbus"}, "gun": {"name": "Portal Gun"}}


@router.get("/")
async def read_items(pagination: PaginationDep) -> dict:
    return fake_items


@router.get("/{item_id}")
async def read_item(item_id: str) -> dict:
    if item_id not in fake_items:
        raise HTTPException(status_code=404, detail="Item not found")
    return {"item_id": item_id, **fake_items[item_id]}
```

```python
# app/internal/admin.py
from fastapi import APIRouter

router = APIRouter()


@router.post("/")
async def update_admin() -> dict:
    return {"message": "Admin getting schwifty"}
```

```python
# app/main.py
from fastapi import Depends, FastAPI
from .dependencies import get_token_header
from .routers import items, users
from .internal import admin

app = FastAPI(title="Bigger Applications")

app.include_router(users.router)
app.include_router(items.router)
app.include_router(
    admin.router,
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(get_token_header)],
    responses={418: {"description": "I'm a teapot"}},
)


@app.get("/")
async def root() -> dict:
    return {"message": "Hello Bigger Applications!"}
```

Запуск: `uvicorn app.main:app --reload` (из директории, содержащей пакет `app`).

## Частые вопросы на собеседовании (Q/A)

**1. Что такое `APIRouter` и зачем он нужен?**
Контейнер для группы связанных маршрутов («мини-приложение»), который позволяет разбить большое приложение на модули и подключить их через `include_router`.

**2. Какие параметры можно задать у `APIRouter`?**
`prefix`, `tags`, `dependencies`, `responses` (плюс `default_response_class`, `deprecated` и др.). Они применяются ко всем маршрутам роутера.

**3. Чем отличается `dependencies` у роутера от `Depends` в сигнатуре обработчика?**
`dependencies=[...]` выполняются ради побочного эффекта (проверки) и не внедряют значение. `Depends` в параметрах внедряет результат в обработчик.

**4. Что делает `include_router` и меняет ли он исходный роутер?**
Подключает маршруты роутера к приложению/другому роутеру. Не мутирует оригинал, а копирует маршруты с применёнными параметрами — поэтому роутер можно подключить несколько раз.

**5. Можно ли переопределить prefix/tags/dependencies при подключении?**
Да, `include_router(router, prefix=..., tags=..., dependencies=...)` добавляет их поверх заданных в самом роутере.

**6. Какие есть уровни зависимостей и в каком порядке они выполняются?**
Приложение -> роутер -> эндпоинт. Сначала глобальные (`FastAPI(dependencies=...)`), затем роутера, затем конкретного маршрута.

**7. Как объявить глобальную зависимость на всё приложение?**
`app = FastAPI(dependencies=[Depends(...)])` — выполнится на каждом запросе.

**8. Почему важны относительные импорты и когда они ломаются?**
Они делают пакет переносимым внутри проекта. Ломаются при запуске файла как скрипта (`python routers/items.py`) вместо запуска как модуля (`uvicorn app.main:app`) и при отсутствии `__init__.py`.

**9. Можно ли вкладывать роутеры друг в друга?**
Да, `router.include_router(subrouter)` создаёт иерархию путей (префиксы складываются).

**10. Должен ли `prefix` заканчиваться слешем?**
Нет, `prefix` не должен иметь завершающего слеша (например, `/items`, а внутри маршруты начинаются со слеша).

**11. Как организовать переиспользование зависимостей между роутерами?**
Вынести их в общий модуль (`dependencies.py`) и импортировать; удобно объявлять алиасы `Annotated[..., Depends(...)]`.

**12. Чем `tags` помогают?**
Группируют эндпоинты в Swagger UI/OpenAPI по разделам, улучшая навигацию документации.

**13. Можно ли один роутер подключить к нескольким приложениям/префиксам?**
Да, поскольку `include_router` копирует маршруты, один роутер переиспользуется с разными префиксами/тегами.

**14. Где лучше держать модели и настройки в большом приложении?**
Модели — в пакете `models/`, настройки — в `config.py` (`pydantic-settings`), подключение к БД — в `database.py`; роуты — в `routers/`.

## Подводные камни (gotchas)

- **Завершающий слеш в `prefix`** — приведёт к двойному слешу в путях; не ставьте его.
- **Запуск файла напрямую** (`python items.py`) ломает относительные импорты — запускайте как модуль.
- **Отсутствие `__init__.py`** — папка не пакет, относительные импорты не работают.
- **Циклические импорты** между роутерами и зависимостями — выносите общие части в отдельный модуль.
- **Ожидание мутации исходного роутера** — `include_router` копирует, поэтому изменения после подключения не подхватятся.
- **Дублирование путей** при подключении одного роутера дважды без разных префиксов — конфликт маршрутов.
- **Глобальные зависимости с тяжёлой логикой** замедляют все запросы — применяйте точечно, где возможно.
- **Порядок маршрутов** внутри роутера важен: `/users/me` должен идти до `/users/{username}`, иначе `me` попадёт в параметр.

## Лучшие практики

- Группируйте маршруты по доменам в `routers/`, по одному роутеру на ресурс.
- Выносите общие зависимости в `dependencies.py` и используйте `Annotated`-алиасы.
- Задавайте `prefix` и `tags` на уровне роутера, а не повторяйте в каждом маршруте.
- Используйте `dependencies=[...]` роутера/приложения для сквозной авторизации.
- Держите `main.py` тонким: только создание `app` и `include_router`.
- Настройки и подключение к БД — в отдельных модулях (`config.py`, `database.py`).
- Запускайте приложение как модуль: `uvicorn app.main:app`.
- Документируйте ответы через `responses` для лучшего OpenAPI.

## Шпаргалка

```python
# Роутер
router = APIRouter(
    prefix="/items",                       # без завершающего слеша
    tags=["items"],
    dependencies=[Depends(get_token_header)],
    responses={404: {"description": "Не найдено"}},
)

@router.get("/{item_id}")
async def read_item(item_id: str): ...

# Подключение
app.include_router(router)                              # как есть
app.include_router(router, prefix="/admin",            # с переопределением
                   tags=["admin"], dependencies=[Depends(dep)])

# Уровни зависимостей
FastAPI(dependencies=[...])        # приложение
APIRouter(dependencies=[...])      # роутер
@router.get("/x", dependencies=[...])  # эндпоинт

# Относительные импорты
from ..dependencies import get_token_header
```

- `include_router` копирует маршруты (не мутирует роутер) -> можно подключать многократно.
- Порядок зависимостей: приложение -> роутер -> эндпоинт.
- Запуск: `uvicorn app.main:app --reload`.
