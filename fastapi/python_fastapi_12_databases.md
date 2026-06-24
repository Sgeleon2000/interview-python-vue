# FastAPI — SQL-базы данных (SQLModel) — конспект и вопросы

## О чём раздел

FastAPI рекомендует для работы с SQL-базами библиотеку **SQLModel** (написана автором FastAPI). SQLModel объединяет в одном классе модель Pydantic v2 (валидация, сериализация) и модель таблицы SQLAlchemy (ORM). Под капотом — SQLAlchemy 2.0. Это позволяет использовать одни и те же типы для валидации входных данных, ответов и таблиц БД, уменьшая дублирование.

В разделе разберём: движок (`engine`), сессии (`Session`), паттерн зависимости `get_session` с `yield`, объявление таблиц (`table=True`), CRUD-операции, `select()`, пагинацию `offset/limit`, разделение моделей (Base/Create/Public/Update), кратко миграции через Alembic и асинхронный вариант с `AsyncSession`.

## Ключевые концепции

- **`engine`** — точка подключения к БД и пул соединений. Создаётся один раз на приложение.
- **`Session`** — единица работы (unit of work): транзакция и кэш объектов. Живёт в пределах одного запроса.
- **`SQLModel`** — базовый класс; с `table=True` становится таблицей.
- **`Field(...)`** — описание колонки (`primary_key`, `index`, `default`, `foreign_key`, `nullable`).
- **`select()`** — построение запросов в стиле SQLAlchemy 2.0.
- **`get_session` с `yield`** — зависимость, открывающая и гарантированно закрывающая сессию.
- **Разделение моделей** — Base/Create/Public/Update для разных контрактов API.
- **Alembic** — инструмент миграций схемы.
- **`AsyncEngine` / `AsyncSession`** — асинхронный доступ к БД.

## Подробный разбор с примерами кода

### 1. Движок (engine)

```python
from sqlmodel import SQLModel, create_engine

sqlite_url = "sqlite:///database.db"

# check_same_thread=False нужно только для SQLite в многопоточном FastAPI
connect_args = {"check_same_thread": False}
engine = create_engine(sqlite_url, echo=True, connect_args=connect_args)


def create_db_and_tables() -> None:
    # Создаёт таблицы по всем зарегистрированным моделям table=True
    SQLModel.metadata.create_all(engine)
```

Для PostgreSQL: `postgresql+psycopg://user:pass@host:5432/dbname`, `connect_args` не нужен; добавляют параметры пула (`pool_size`, `max_overflow`, `pool_pre_ping=True`).

### 2. Сессия и зависимость get_session с yield

```python
from typing import Annotated
from fastapi import Depends
from sqlmodel import Session


def get_session():
    # yield-зависимость: код после yield выполнится после завершения запроса
    with Session(engine) as session:
        yield session
    # выход из with: сессия закрывается, соединение возвращается в пул


# Удобный алиас типа для повторного использования
SessionDep = Annotated[Session, Depends(get_session)]
```

Зависимость с `yield` гарантирует закрытие сессии даже при исключении. Контекстный менеджер `with Session(engine)` сам управляет ресурсом.

### 3. Модели таблиц (table=True)

```python
from sqlmodel import SQLModel, Field


class Hero(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str = Field(index=True)
    secret_name: str
    age: int | None = Field(default=None, index=True)
```

- `id: int | None` с `default=None` — БД сама проставит автоинкремент.
- `index=True` создаёт индекс для частых фильтров.
- Один класс = одна таблица.

### 4. Разделение моделей (Base / Create / Public / Update)

Хорошая практика — не использовать таблицу напрямую в API, а развести контракты:

```python
from sqlmodel import SQLModel, Field


# Общие поля (без id) — переиспользуемая база
class HeroBase(SQLModel):
    name: str = Field(index=True)
    secret_name: str
    age: int | None = Field(default=None, index=True)


# Таблица в БД
class Hero(HeroBase, table=True):
    id: int | None = Field(default=None, primary_key=True)


# Входные данные на создание (клиент не задаёт id)
class HeroCreate(HeroBase):
    pass


# Ответ клиенту (id обязателен и присутствует)
class HeroPublic(HeroBase):
    id: int


# Частичное обновление: все поля опциональны
class HeroUpdate(SQLModel):
    name: str | None = None
    secret_name: str | None = None
    age: int | None = None
```

Зачем: `Create` не пускает клиента задавать `id`; `Public` гарантирует наличие `id` в ответе и не светит лишние/секретные поля; `Update` делает все поля необязательными для PATCH.

### 5. CRUD-операции

```python
from fastapi import FastAPI, HTTPException, Query
from sqlmodel import select

app = FastAPI()


@app.on_event("startup")  # или современный lifespan
def on_startup() -> None:
    create_db_and_tables()


# CREATE
@app.post("/heroes", response_model=HeroPublic)
def create_hero(hero: HeroCreate, session: SessionDep) -> Hero:
    # Валидируем входной HeroCreate в таблицу Hero
    db_hero = Hero.model_validate(hero)
    session.add(db_hero)
    session.commit()
    session.refresh(db_hero)  # подтянуть сгенерированный id
    return db_hero


# READ (список с пагинацией)
@app.get("/heroes", response_model=list[HeroPublic])
def read_heroes(
    session: SessionDep,
    offset: int = 0,
    limit: Annotated[int, Query(le=100)] = 20,
) -> list[Hero]:
    heroes = session.exec(select(Hero).offset(offset).limit(limit)).all()
    return heroes


# READ (один по id)
@app.get("/heroes/{hero_id}", response_model=HeroPublic)
def read_hero(hero_id: int, session: SessionDep) -> Hero:
    hero = session.get(Hero, hero_id)
    if not hero:
        raise HTTPException(status_code=404, detail="Hero not found")
    return hero


# UPDATE (частичное, PATCH)
@app.patch("/heroes/{hero_id}", response_model=HeroPublic)
def update_hero(hero_id: int, hero: HeroUpdate, session: SessionDep) -> Hero:
    db_hero = session.get(Hero, hero_id)
    if not db_hero:
        raise HTTPException(status_code=404, detail="Hero not found")
    # exclude_unset=True — только реально переданные поля
    hero_data = hero.model_dump(exclude_unset=True)
    db_hero.sqlmodel_update(hero_data)
    session.add(db_hero)
    session.commit()
    session.refresh(db_hero)
    return db_hero


# DELETE
@app.delete("/heroes/{hero_id}")
def delete_hero(hero_id: int, session: SessionDep) -> dict:
    hero = session.get(Hero, hero_id)
    if not hero:
        raise HTTPException(status_code=404, detail="Hero not found")
    session.delete(hero)
    session.commit()
    return {"ok": True}
```

Ключевые приёмы:
- `session.get(Model, pk)` — быстрый доступ по первичному ключу.
- `session.exec(select(...))` — запросы; `.all()`, `.first()`, `.one()`.
- `session.add` + `commit` + `refresh` — стандартный цикл записи.
- `model_dump(exclude_unset=True)` — корректный частичный апдейт.

### 6. select(), фильтры, пагинация

```python
from sqlmodel import select

# Фильтр + сортировка + пагинация
stmt = (
    select(Hero)
    .where(Hero.age >= 18)
    .order_by(Hero.name)
    .offset(0)
    .limit(20)
)
heroes = session.exec(stmt).all()

# Первый результат или None
hero = session.exec(select(Hero).where(Hero.name == "Deadpond")).first()
```

`offset/limit` — простая пагинация по смещению (для больших таблиц лучше keyset/cursor-пагинация, но это отдельная тема).

### 7. Чистый SQLAlchemy 2.0 (упоминание)

Если не использовать SQLModel, эквивалент на SQLAlchemy 2.0:

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, sessionmaker
from sqlalchemy import create_engine, select, String

class Base(DeclarativeBase):
    pass

class Hero(Base):
    __tablename__ = "hero"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String, index=True)

engine = create_engine("sqlite:///db.sqlite")
SessionLocal = sessionmaker(engine)

with SessionLocal() as session:
    rows = session.scalars(select(Hero).limit(10)).all()
```

SQLModel — это, по сути, удобная обёртка поверх этого с интеграцией Pydantic.

### 8. Асинхронный вариант (кратко)

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlmodel import select

# Драйвер должен быть async: postgresql+asyncpg, sqlite+aiosqlite
engine = create_async_engine("sqlite+aiosqlite:///db.sqlite", echo=True)
async_session_maker = async_sessionmaker(engine, expire_on_commit=False)


async def get_async_session():
    async with async_session_maker() as session:
        yield session


AsyncSessionDep = Annotated[AsyncSession, Depends(get_async_session)]


@app.get("/heroes", response_model=list[HeroPublic])
async def read_heroes(session: AsyncSessionDep) -> list[Hero]:
    result = await session.exec(select(Hero).limit(20))
    return result.all()
```

Особенности async: нужен async-драйвер (`asyncpg`, `aiosqlite`), все операции через `await`, `expire_on_commit=False` чтобы объекты не «протухали» после commit, обработчики — `async def`.

### 9. Миграции (Alembic, кратко)

`SQLModel.metadata.create_all()` хорош для прототипа, но не управляет изменениями схемы в продакшене. Для этого — Alembic:

```bash
alembic init migrations          # инициализация
alembic revision --autogenerate -m "create hero table"   # сгенерировать миграцию
alembic upgrade head             # применить
alembic downgrade -1             # откатить на шаг
```

В `env.py` указывают `target_metadata = SQLModel.metadata`, чтобы автогенерация видела модели. Каждое изменение схемы — новая ревизия, версионируется в git.

## Полный рабочий пример

```python
from contextlib import asynccontextmanager
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, Query
from sqlmodel import Field, Session, SQLModel, create_engine, select


# --- Модели ---
class HeroBase(SQLModel):
    name: str = Field(index=True)
    secret_name: str
    age: int | None = Field(default=None, index=True)


class Hero(HeroBase, table=True):
    id: int | None = Field(default=None, primary_key=True)


class HeroCreate(HeroBase):
    pass


class HeroPublic(HeroBase):
    id: int


class HeroUpdate(SQLModel):
    name: str | None = None
    secret_name: str | None = None
    age: int | None = None


# --- БД ---
engine = create_engine(
    "sqlite:///heroes.db",
    echo=True,
    connect_args={"check_same_thread": False},
)


def get_session():
    with Session(engine) as session:
        yield session


SessionDep = Annotated[Session, Depends(get_session)]


@asynccontextmanager
async def lifespan(app: FastAPI):
    SQLModel.metadata.create_all(engine)  # создать таблицы на старте
    yield


app = FastAPI(lifespan=lifespan, title="Heroes API")


@app.post("/heroes", response_model=HeroPublic)
def create_hero(hero: HeroCreate, session: SessionDep) -> Hero:
    db_hero = Hero.model_validate(hero)
    session.add(db_hero)
    session.commit()
    session.refresh(db_hero)
    return db_hero


@app.get("/heroes", response_model=list[HeroPublic])
def read_heroes(
    session: SessionDep,
    offset: int = 0,
    limit: Annotated[int, Query(le=100)] = 20,
) -> list[Hero]:
    return session.exec(select(Hero).offset(offset).limit(limit)).all()


@app.get("/heroes/{hero_id}", response_model=HeroPublic)
def read_hero(hero_id: int, session: SessionDep) -> Hero:
    hero = session.get(Hero, hero_id)
    if not hero:
        raise HTTPException(status_code=404, detail="Hero not found")
    return hero


@app.patch("/heroes/{hero_id}", response_model=HeroPublic)
def update_hero(hero_id: int, hero: HeroUpdate, session: SessionDep) -> Hero:
    db_hero = session.get(Hero, hero_id)
    if not db_hero:
        raise HTTPException(status_code=404, detail="Hero not found")
    db_hero.sqlmodel_update(hero.model_dump(exclude_unset=True))
    session.add(db_hero)
    session.commit()
    session.refresh(db_hero)
    return db_hero


@app.delete("/heroes/{hero_id}")
def delete_hero(hero_id: int, session: SessionDep) -> dict:
    hero = session.get(Hero, hero_id)
    if not hero:
        raise HTTPException(status_code=404, detail="Hero not found")
    session.delete(hero)
    session.commit()
    return {"ok": True}
```

## Частые вопросы на собеседовании (Q/A)

**1. Что такое SQLModel и зачем он во FastAPI?**
Библиотека от автора FastAPI поверх SQLAlchemy 2.0 и Pydantic v2. Один класс с `table=True` служит и ORM-моделью, и Pydantic-схемой, что убирает дублирование.

**2. Зачем нужен `engine` и сколько их создавать?**
`engine` управляет соединениями и пулом. Создаётся один раз на приложение и переиспользуется.

**3. Что такое `Session` и какой у неё жизненный цикл?**
Единица работы (транзакция + кэш объектов). Открывается на один запрос и закрывается после него через зависимость с `yield`.

**4. Почему зависимость `get_session` использует `yield`?**
Чтобы гарантированно закрыть сессию (вернуть соединение в пул) после обработки запроса, в том числе при исключении.

**5. Зачем `session.refresh()` после `commit`?**
Чтобы подтянуть сгенерированные БД значения (например, автоинкрементный `id`) обратно в объект.

**6. Зачем разделять модели Base/Create/Public/Update?**
Разные контракты: `Create` без `id`, `Public` с гарантированным `id` и без секретных полей, `Update` со всеми опциональными полями для PATCH. Это безопаснее, чем светить таблицу напрямую.

**7. Как сделать частичное обновление корректно?**
`hero.model_dump(exclude_unset=True)` — берём только реально переданные поля, затем `sqlmodel_update()`.

**8. Как работает пагинация offset/limit и её минусы?**
`offset` пропускает N строк, `limit` ограничивает количество. Минус — на больших offset БД всё равно сканирует пропущенные строки; для глубоких страниц лучше keyset/cursor.

**9. Чем отличается `session.get` от `session.exec(select(...))`?**
`get` — прямой доступ по первичному ключу (использует identity map). `exec(select)` — произвольные запросы с фильтрами, сортировкой, пагинацией.

**10. Зачем `check_same_thread=False` для SQLite?**
SQLite по умолчанию запрещает использование соединения из разных потоков; FastAPI может обрабатывать запросы в разных потоках, поэтому ограничение снимают. Для PostgreSQL не нужно.

**11. Чем отличается async-вариант?**
`create_async_engine` + `AsyncSession`, async-драйвер (`asyncpg`/`aiosqlite`), все операции с `await`, `expire_on_commit=False`, обработчики `async def`.

**12. Почему `create_all()` не подходит для продакшена?**
Он только создаёт отсутствующие таблицы и не отслеживает изменения схемы (новые колонки, переименования). Для эволюции схемы нужны миграции (Alembic).

**13. Что делает Alembic и как он видит модели?**
Инструмент версионируемых миграций. Через `target_metadata = SQLModel.metadata` автогенерация сравнивает модели с реальной схемой и создаёт ревизии.

**14. Что произойдёт, если не вызвать `commit`?**
Изменения не зафиксируются в БД; при выходе из сессии незакоммиченная транзакция откатится.

## Подводные камни (gotchas)

- **Использование `table=True`-модели напрямую как `response_model`** — может протечь лишними полями и связями; используйте `Public`-модель.
- **Забыли `refresh`** — после `commit` объект может не иметь `id` или иметь устаревшие данные.
- **`offset` на больших таблицах** — медленно; используйте keyset-пагинацию.
- **Одна сессия на всё приложение** — нельзя; сессия не потокобезопасна и копит состояние. Сессия на запрос.
- **Тяжёлые синхронные запросы в `async def`-обработчике** — блокируют event loop; либо синхронный обработчик (FastAPI вынесет в threadpool), либо полноценный async-стек.
- **`expire_on_commit=True` в async** — после `commit` доступ к атрибутам инициирует ленивую загрузку и падает; ставьте `False`.
- **Смешивание sync и async engine** — нельзя; драйверы и API разные.
- **`model_dump()` без `exclude_unset`** в PATCH — затрёт незаданные поля значениями по умолчанию.
- **Забытый индекс** на часто фильтруемых колонках — медленные запросы.

## Лучшие практики

- Разделяйте модели: Base/Create/Public/Update.
- Сессия — на один запрос, через зависимость с `yield`.
- Используйте миграции (Alembic) с самого начала продакшен-проекта.
- Ставьте `index=True` на колонки в `WHERE`/`ORDER BY`.
- Ограничивайте `limit` сверху (`Query(le=100)`) от перегрузки.
- Конфигурацию подключения и пула берите из `pydantic-settings`/переменных окружения.
- Для PostgreSQL включайте `pool_pre_ping=True`.
- Для частичного обновления — `exclude_unset=True`.
- Выбирайте sync или async стек целиком, не смешивайте.

## Шпаргалка

```python
# Движок
engine = create_engine(url, connect_args={"check_same_thread": False})  # SQLite

# Зависимость-сессия
def get_session():
    with Session(engine) as session:
        yield session
SessionDep = Annotated[Session, Depends(get_session)]

# CRUD
session.add(obj); session.commit(); session.refresh(obj)   # create
session.get(Model, pk)                                      # read by pk
session.exec(select(Model).offset(o).limit(l)).all()       # list
obj.sqlmodel_update(data); session.commit()                # update
session.delete(obj); session.commit()                      # delete

# Модели
class Base(SQLModel): ...
class Tbl(Base, table=True): id: int | None = Field(default=None, primary_key=True)
class Create(Base): ...
class Public(Base): id: int
class Update(SQLModel): name: str | None = None
```

- Alembic: `revision --autogenerate` -> `upgrade head`.
- Async: `create_async_engine` + `AsyncSession` + `await`, `expire_on_commit=False`.
