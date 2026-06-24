# FastAPI Advanced — Продвинутое тестирование — конспект и вопросы

## О чём раздел

Этот раздел посвящён продвинутому тестированию FastAPI-приложений. Базовое тестирование (`TestClient`, простые assert'ы) — это только начало. На уровне middle/senior нужно уметь:

- подменять зависимости (`app.dependency_overrides`) — мокать БД, аутентификацию, внешние сервисы;
- организовывать тестовую базу данных (отдельный engine, SQLite in-memory, фикстуры `setup`/`teardown`);
- писать настоящие async-тесты через `httpx.AsyncClient` + `ASGITransport`, `pytest-asyncio` / `anyio`;
- тестировать события жизненного цикла (`lifespan`, `startup`/`shutdown`) через контекстный менеджер `with TestClient(app) as client`;
- тестировать WebSocket'ы, загрузку файлов и форм;
- грамотно структурировать тесты и фикстуры в `conftest.py`.

Главная идея FastAPI-тестирования: **система внедрения зависимостей (DI) спроектирована так, чтобы быть тестируемой**. Любую зависимость (`Depends(...)`) можно заменить на тестовую реализацию без изменения кода приложения. Это и есть основной инструмент изоляции.

## Ключевые концепции

- **`TestClient`** — синхронный клиент (на базе `httpx`), который под капотом гоняет ASGI-приложение в памяти, без поднятия реального сервера. Импортируется из `fastapi.testclient`. Сам `TestClient` синхронный, но внутри корректно прокручивает event loop приложения.
- **`app.dependency_overrides`** — словарь `{оригинальная_зависимость: подменная_зависимость}`. FastAPI смотрит в него при резолве каждого `Depends(...)`. Ключ — это **функция-зависимость как объект** (не строка), значение — функция-замена. После теста словарь обязательно нужно очищать.
- **`httpx.AsyncClient` + `ASGITransport`** — способ дёргать приложение асинхронно прямо в памяти. `ASGITransport(app=app)` подключает приложение как транспорт, без сети. Нужен, когда тест сам должен быть `async` (например, чтобы шарить async-сессию БД с приложением).
- **`pytest-asyncio` / `anyio`** — плагины, позволяющие писать `async def test_...`. FastAPI-доки используют `anyio` (`@pytest.mark.anyio`), но в реальных проектах чаще встречается `pytest-asyncio` (`@pytest.mark.asyncio` или `asyncio_mode = auto`).
- **`lifespan` / события** — код в `lifespan` (или старых `@app.on_event`) выполняется только когда `TestClient` используется как контекстный менеджер: `with TestClient(app) as client:`. Без `with` startup/shutdown НЕ запускаются.
- **Тестовая БД** — отдельный движок (engine), часто SQLite (`sqlite:///./test.db` или in-memory `sqlite://`), с пересозданием схемы на каждый тест/сессию. Чтобы in-memory SQLite не «терялась» между подключениями, нужен `StaticPool`.
- **Фикстуры pytest** — функции с `@pytest.fixture`, дающие тестам подготовленные объекты (клиент, сессия БД, тестовый пользователь). Имеют scope (`function`, `module`, `session`) и механизм teardown через `yield`.
- **`conftest.py`** — файл с общими фикстурами, который pytest автоматически подхватывает для всех тестов в каталоге и подкаталогах. Импортировать его не нужно.

## Подробный разбор с примерами кода

### 1. Переопределение зависимостей — `app.dependency_overrides`

Это сердце тестирования FastAPI. Допустим, приложение:

```python
from typing import Annotated
from fastapi import FastAPI, Depends, Header, HTTPException

app = FastAPI()


# Реальная зависимость: проверяет токен через внешний сервис / БД.
async def get_current_user(authorization: Annotated[str | None, Header()] = None):
    if authorization != "Bearer real-token":
        raise HTTPException(status_code=401, detail="Не авторизован")
    # В реальности тут поход в БД/сервис.
    return {"id": 1, "username": "alice"}


@app.get("/me")
async def read_me(user: Annotated[dict, Depends(get_current_user)]):
    return user
```

В тесте мы не хотим гонять реальную проверку токена. Подменяем зависимость:

```python
from fastapi.testclient import TestClient
from main import app, get_current_user


# Тестовая замена: всегда возвращает фиктивного пользователя.
async def override_get_current_user():
    return {"id": 99, "username": "test_user"}


def test_read_me_with_override():
    # Регистрируем подмену: ключ — оригинальная функция-зависимость.
    app.dependency_overrides[get_current_user] = override_get_current_user

    client = TestClient(app)
    response = client.get("/me")  # Заголовок Authorization не нужен!

    assert response.status_code == 200
    assert response.json()["username"] == "test_user"

    # ОБЯЗАТЕЛЬНО очищаем, иначе подмена «протечёт» в другие тесты.
    app.dependency_overrides.clear()
```

Ключевые моменты:

- Ключ словаря — **сам объект функции** `get_current_user`, импортированный из того же места, что используется в приложении. Если импортировать «не ту» копию функции, подмена не сработает.
- Подмена работает на любом уровне вложенности: если `get_current_user` сам зависит от `get_db`, можно подменить либо верхнюю, либо нижнюю зависимость.
- Сигнатура замены может отличаться от оригинала, но её собственные `Depends`/параметры FastAPI тоже отрезолвит.

### 2. Чистая очистка overrides через фикстуру

Ручной `clear()` легко забыть. Лучше — фикстура:

```python
import pytest
from fastapi.testclient import TestClient
from main import app, get_current_user


async def override_get_current_user():
    return {"id": 99, "username": "test_user"}


@pytest.fixture
def client():
    # Настраиваем подмены ДО теста.
    app.dependency_overrides[get_current_user] = override_get_current_user
    with TestClient(app) as c:  # with -> запустятся lifespan/startup.
        yield c
    # teardown: гарантированно чистим состояние после теста.
    app.dependency_overrides.clear()


def test_me(client):
    assert client.get("/me").json()["username"] == "test_user"
```

### 3. Мок внешнего сервиса как зависимости

Любой внешний вызов (платёжка, email, S3) удобно прятать за зависимостью, чтобы потом подменить:

```python
from typing import Annotated, Protocol
from fastapi import FastAPI, Depends

app = FastAPI()


class PaymentGateway(Protocol):
    async def charge(self, amount: int) -> str: ...


class RealPaymentGateway:
    async def charge(self, amount: int) -> str:
        # ... реальный HTTP-запрос к платёжной системе ...
        return "real-charge-id"


def get_payment_gateway() -> PaymentGateway:
    return RealPaymentGateway()


@app.post("/pay")
async def pay(amount: int, gw: Annotated[PaymentGateway, Depends(get_payment_gateway)]):
    charge_id = await gw.charge(amount)
    return {"charge_id": charge_id}
```

Тест с фейковым шлюзом:

```python
class FakePaymentGateway:
    def __init__(self):
        self.charges: list[int] = []

    async def charge(self, amount: int) -> str:
        self.charges.append(amount)  # запоминаем для проверки
        return "fake-charge-id"


def test_pay(client_factory):
    fake = FakePaymentGateway()
    app.dependency_overrides[get_payment_gateway] = lambda: fake

    client = TestClient(app)
    resp = client.post("/pay", params={"amount": 500})

    assert resp.json()["charge_id"] == "fake-charge-id"
    assert fake.charges == [500]  # проверяем, что вызвали с нужной суммой
    app.dependency_overrides.clear()
```

### 4. Тестовая БД (синхронная, SQLAlchemy + SQLite)

Приложение использует зависимость `get_db`:

```python
# database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base

engine = create_engine("sqlite:///./app.db", connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

В тестах поднимаем **отдельный** движок на in-memory SQLite. Важный нюанс: in-memory база живёт, пока есть подключение, а пул создаёт новые соединения → таблицы «исчезают». Решение — `StaticPool` (одно соединение на весь движок):

```python
# conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import StaticPool
from fastapi.testclient import TestClient

from main import app
from database import Base, get_db


@pytest.fixture(scope="function")
def db_session():
    # Отдельный тестовый движок: in-memory SQLite + StaticPool.
    engine = create_engine(
        "sqlite://",  # in-memory
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,  # одно соединение -> данные не теряются
    )
    TestingSessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)

    # setup: создаём схему перед каждым тестом -> полная изоляция.
    Base.metadata.create_all(bind=engine)

    db = TestingSessionLocal()
    try:
        yield db
    finally:
        # teardown: закрываем сессию и сносим схему.
        db.close()
        Base.metadata.drop_all(bind=engine)


@pytest.fixture(scope="function")
def client(db_session):
    # Подменяем get_db так, чтобы приложение и тест работали в ОДНОЙ сессии.
    def override_get_db():
        try:
            yield db_session
        finally:
            pass  # сессию закроет фикстура db_session

    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

Теперь тест получает чистую БД и клиент, разделяющий с ним сессию:

```python
def test_create_user(client, db_session):
    resp = client.post("/users", json={"username": "bob", "email": "b@x.io"})
    assert resp.status_code == 201

    # Можем проверить состояние БД напрямую через ту же сессию.
    from models import User
    user = db_session.query(User).filter_by(username="bob").first()
    assert user is not None
```

### 5. Async-тесты: `httpx.AsyncClient` + `ASGITransport`

`TestClient` синхронный. Если ваши зависимости/код используют **async-сессию БД** (`AsyncSession`), удобнее (а иногда необходимо) делать тест полностью асинхронным, чтобы шарить один event loop и одну async-сессию.

```python
import pytest
from httpx import AsyncClient, ASGITransport
from main import app


# Вариант с anyio (как в официальных доках FastAPI).
@pytest.fixture
def anyio_backend():
    return "asyncio"  # гоняем только под asyncio, без trio


@pytest.mark.anyio
async def test_root_async():
    transport = ASGITransport(app=app)  # подключаем ASGI-приложение в память
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        response = await ac.get("/")
    assert response.status_code == 200
```

С `pytest-asyncio` это выглядит так (в `pyproject.toml`/`pytest.ini` ставим `asyncio_mode = auto`):

```python
import pytest
from httpx import AsyncClient, ASGITransport
from main import app


@pytest.mark.asyncio
async def test_root_async():
    async with AsyncClient(
        transport=ASGITransport(app=app), base_url="http://test"
    ) as ac:
        resp = await ac.get("/")
    assert resp.json() == {"message": "Hello World"}
```

> Важно: в новых версиях httpx (0.27+) нельзя передавать `app=...` напрямую в `AsyncClient` — только через `ASGITransport(app=app)`.

### 6. Async тестовая БД (SQLAlchemy AsyncSession)

```python
# conftest.py (async-вариант)
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.pool import StaticPool

from main import app
from database import Base, get_db


@pytest_asyncio.fixture
async def async_session():
    engine = create_async_engine(
        "sqlite+aiosqlite://",  # async-драйвер aiosqlite
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    # setup схемы в async-режиме.
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    Session = async_sessionmaker(engine, expire_on_commit=False, class_=AsyncSession)
    async with Session() as session:
        yield session

    # teardown.
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()


@pytest_asyncio.fixture
async def async_client(async_session):
    async def override_get_db():
        yield async_session

    app.dependency_overrides[get_db] = override_get_db
    async with AsyncClient(
        transport=ASGITransport(app=app), base_url="http://test"
    ) as ac:
        yield ac
    app.dependency_overrides.clear()


# Пример теста:
import pytest


@pytest.mark.asyncio
async def test_create_item(async_client):
    resp = await async_client.post("/items", json={"name": "Меч", "price": 100})
    assert resp.status_code == 201
    assert resp.json()["name"] == "Меч"
```

### 7. Тестирование событий / lifespan

Код запуска/остановки выполняется только когда `TestClient` используется как контекстный менеджер.

```python
# main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI

ml_models: dict = {}


@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup: «загружаем модель» / открываем пул соединений.
    ml_models["greeter"] = lambda name: f"Привет, {name}"
    yield
    # shutdown: освобождаем ресурсы.
    ml_models.clear()


app = FastAPI(lifespan=lifespan)


@app.get("/greet/{name}")
async def greet(name: str):
    return {"msg": ml_models["greeter"](name)}
```

Тест:

```python
from fastapi.testclient import TestClient
from main import app, ml_models


def test_lifespan_runs():
    # БЕЗ with блок lifespan НЕ выполнится -> ml_models будет пустым.
    with TestClient(app) as client:
        # startup уже отработал -> модель загружена.
        assert "greeter" in ml_models
        resp = client.get("/greet/Аня")
        assert resp.json() == {"msg": "Привет, Аня"}
    # После выхода из with отработал shutdown.
    assert "greeter" not in ml_models
```

### 8. Тестирование WebSocket

`TestClient` умеет работать с WebSocket через `client.websocket_connect(...)` (тоже как контекстный менеджер).

```python
# main.py
from fastapi import FastAPI, WebSocket

app = FastAPI()


@app.websocket("/ws")
async def websocket_endpoint(ws: WebSocket):
    await ws.accept()
    while True:
        data = await ws.receive_text()
        await ws.send_text(f"Эхо: {data}")


# test_ws.py
from fastapi.testclient import TestClient
from main import app


def test_websocket_echo():
    client = TestClient(app)
    with client.websocket_connect("/ws") as ws:
        ws.send_text("ping")
        data = ws.receive_text()
        assert data == "Эхо: ping"
```

Можно отправлять/получать JSON: `ws.send_json({...})` / `ws.receive_json()`. Закрытие соединения проверяется через `WebSocketDisconnect`.

### 9. Тестирование загрузки файлов и форм

Для multipart-форм нужен установленный `python-multipart`.

```python
# main.py
from typing import Annotated
from fastapi import FastAPI, UploadFile, File, Form

app = FastAPI()


@app.post("/upload")
async def upload(
    file: Annotated[UploadFile, File()],
    description: Annotated[str, Form()],
):
    content = await file.read()
    return {
        "filename": file.filename,
        "size": len(content),
        "description": description,
        "content_type": file.content_type,
    }
```

Тест:

```python
import io
from fastapi.testclient import TestClient
from main import app


def test_upload_file():
    client = TestClient(app)
    # files=... -> multipart; формат: {"имя_поля": (имя_файла, поток, mime)}
    files = {"file": ("report.txt", io.BytesIO(b"hello bytes"), "text/plain")}
    data = {"description": "Тестовый отчёт"}  # обычные поля формы -> data=

    resp = client.post("/upload", files=files, data=data)

    assert resp.status_code == 200
    body = resp.json()
    assert body["filename"] == "report.txt"
    assert body["size"] == len(b"hello bytes")
    assert body["description"] == "Тестовый отчёт"
    assert body["content_type"] == "text/plain"
```

Чисто форма без файла (`application/x-www-form-urlencoded`):

```python
def test_login_form():
    client = TestClient(app)
    # Для формы используем data=, НЕ json=.
    resp = client.post("/login", data={"username": "alice", "password": "secret"})
    assert resp.status_code == 200
```

### 10. Структура тестов и `conftest.py`

Типичная структура:

```
project/
├── app/
│   ├── main.py
│   ├── database.py
│   └── models.py
└── tests/
    ├── conftest.py        # общие фикстуры (client, db_session, auth_headers)
    ├── test_users.py
    ├── test_items.py
    └── test_auth.py
```

`conftest.py` подхватывается автоматически, импортировать его в тестах не нужно. Полезные фикстуры в нём: `client`, `db_session`, `auth_headers` (готовый заголовок авторизации), `seed_data` (начальные данные).

## Полный рабочий пример

`app/main.py`:

```python
from contextlib import asynccontextmanager
from typing import Annotated
from fastapi import FastAPI, Depends, HTTPException, Header
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.orm import sessionmaker, declarative_base, Session

Base = declarative_base()
engine = create_engine("sqlite:///./app.db", connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)


class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    username = Column(String, unique=True, index=True)


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


async def get_current_username(x_token: Annotated[str | None, Header()] = None) -> str:
    if x_token != "secret-token":
        raise HTTPException(status_code=401, detail="Неверный токен")
    return "admin"


@asynccontextmanager
async def lifespan(app: FastAPI):
    Base.metadata.create_all(bind=engine)  # startup
    yield
    # shutdown: тут можно закрыть внешние пулы


app = FastAPI(lifespan=lifespan)


@app.post("/users", status_code=201)
async def create_user(
    username: str,
    db: Annotated[Session, Depends(get_db)],
    _user: Annotated[str, Depends(get_current_username)],
):
    user = User(username=username)
    db.add(user)
    db.commit()
    db.refresh(user)
    return {"id": user.id, "username": user.username}


@app.get("/users/{user_id}")
async def get_user(user_id: int, db: Annotated[Session, Depends(get_db)]):
    user = db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="Не найден")
    return {"id": user.id, "username": user.username}
```

`tests/conftest.py`:

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import StaticPool
from fastapi.testclient import TestClient

from app.main import app, Base, get_db, get_current_username


@pytest.fixture(scope="function")
def db_session():
    engine = create_engine(
        "sqlite://",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    TestingSessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)
    Base.metadata.create_all(bind=engine)
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()
        Base.metadata.drop_all(bind=engine)


@pytest.fixture(scope="function")
def client(db_session):
    def override_get_db():
        yield db_session

    # Подменяем БД и обходим реальную аутентификацию.
    app.dependency_overrides[get_db] = override_get_db
    app.dependency_overrides[get_current_username] = lambda: "test_admin"

    with TestClient(app) as c:
        yield c

    app.dependency_overrides.clear()
```

`tests/test_users.py`:

```python
from app.main import User


def test_create_and_get_user(client, db_session):
    # create
    resp = client.post("/users", params={"username": "neo"})
    assert resp.status_code == 201
    user_id = resp.json()["id"]

    # проверяем в БД напрямую
    assert db_session.query(User).filter_by(username="neo").count() == 1

    # get
    resp2 = client.get(f"/users/{user_id}")
    assert resp2.status_code == 200
    assert resp2.json()["username"] == "neo"


def test_get_missing_user(client):
    assert client.get("/users/999").status_code == 404


def test_isolation_between_tests(client, db_session):
    # Благодаря пересозданию схемы БД пустая -> подтверждаем изоляцию.
    assert db_session.query(User).count() == 0
```

`pyproject.toml` (фрагмент для async-тестов):

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

## Частые вопросы на собеседовании (Q/A)

**1. Что такое `app.dependency_overrides` и зачем он нужен?**
Это словарь подмен зависимостей `{оригинальная_функция: замена}`. FastAPI при резолве каждого `Depends(...)` сначала смотрит в этот словарь. Нужен, чтобы в тестах заменить реальные зависимости (БД, аутентификацию, внешние API) на тестовые без изменения кода приложения. Это основной механизм изоляции тестов в FastAPI.

**2. Почему важно очищать `dependency_overrides` после теста?**
Словарь живёт на уровне объекта `app`, который шарится между тестами. Если не вызвать `app.dependency_overrides.clear()`, подмена «протечёт» в другие тесты и сделает их недетерминированными. Лучше всего чистить в teardown-части фикстуры (после `yield`).

**3. В чём разница между `TestClient` и `httpx.AsyncClient`?**
`TestClient` синхронный (но внутри прокручивает async event loop приложения) — удобен для большинства случаев. `httpx.AsyncClient` + `ASGITransport` делает тест полностью асинхронным; это нужно, когда тест должен сам быть `async` — например, чтобы шарить одну `AsyncSession` и один event loop с приложением.

**4. Почему события `startup`/`lifespan` не срабатывают в тестах?**
Потому что они выполняются только когда `TestClient` используется как контекстный менеджер: `with TestClient(app) as client:`. Если просто создать `TestClient(app)` без `with`, ASGI lifespan-протокол не запускается, и `startup`/`shutdown` (и `lifespan`) не отработают.

**5. Как сделать изолированную тестовую БД?**
Создать отдельный движок (часто SQLite in-memory `sqlite://` со `StaticPool`), пересоздавать схему (`create_all`/`drop_all`) на каждый тест в фикстуре, и через `dependency_overrides[get_db]` отдавать приложению тестовую сессию. Каждый тест получает чистую БД.

**6. Зачем при in-memory SQLite нужен `StaticPool`?**
In-memory SQLite существует только пока живо подключение. Стандартный пул SQLAlchemy создаёт новые соединения, и каждое видит «свою» пустую БД, теряя таблицы. `StaticPool` держит одно соединение на весь движок, поэтому данные/схема сохраняются между запросами в рамках теста.

**7. Как замокать аутентификацию в тестах?**
Вынести проверку в зависимость (`get_current_user`) и подменить её: `app.dependency_overrides[get_current_user] = lambda: fake_user`. Тогда эндпоинты получают фиктивного пользователя, и не нужно гонять реальную проверку токена/похода в БД.

**8. Как тестировать загрузку файлов?**
Через `client.post(url, files={...}, data={...})`. В `files` передаётся кортеж `(имя_файла, поток_байт, mime_type)`, например `{"file": ("a.txt", io.BytesIO(b"..."), "text/plain")}`. Обычные поля формы идут в `data=`. Нужен установленный `python-multipart`.

**9. В чём разница между `json=`, `data=` и `files=` в запросах TestClient?**
`json=` — тело как `application/json`. `data=` — поля формы (`application/x-www-form-urlencoded`) либо текстовые поля multipart. `files=` — файлы для multipart-загрузки. Для эндпоинтов с `Form()`/`File()` используют `data`/`files`, а не `json`.

**10. Как тестировать WebSocket-эндпоинт?**
Через `with client.websocket_connect("/ws") as ws:` и методы `ws.send_text/send_json`, `ws.receive_text/receive_json`. Отключение проверяется ловлей `WebSocketDisconnect`.

**11. Что такое `conftest.py` и зачем он?**
Файл с общими фикстурами, который pytest автоматически подхватывает для всех тестов в каталоге и подкаталогах без импорта. В нём держат `client`, `db_session`, `auth_headers` и т. п., чтобы не дублировать настройку в каждом тестовом модуле.

**12. Чем `pytest-asyncio` отличается от `anyio`-подхода в доках FastAPI?**
Оба позволяют писать `async def test_...`. Официальные доки используют `anyio` (`@pytest.mark.anyio` + фикстура `anyio_backend`). В реальных проектах чаще `pytest-asyncio` (`@pytest.mark.asyncio` или глобальный `asyncio_mode = auto`). Функционально для FastAPI они эквивалентны; важно не смешивать оба плагина бессистемно.

## Подводные камни (gotchas)

- **Забытый `clear()`** — подмены протекают между тестами. Всегда чистить в teardown фикстуры.
- **Неправильный импорт функции-зависимости.** Ключ в `dependency_overrides` должен быть тем же объектом функции, который используется в `Depends`. Импорт «другой» копии (например, через разные пути модуля) сделает подмену молчаливо неработающей.
- **`TestClient` без `with`** — не запускаются `lifespan`/`startup`, ресурсы (пулы, модели) не инициализируются, тесты падают с непонятными ошибками.
- **In-memory SQLite без `StaticPool`** — таблицы «исчезают», тесты падают с `no such table`.
- **`AsyncClient(app=app)`** в новых httpx больше не работает — нужен `ASGITransport(app=app)`.
- **Смешение sync и async** — нельзя из синхронного `TestClient` шарить `AsyncSession`, открытую в другом event loop. Для async-БД используйте `httpx.AsyncClient` и async-фикстуры.
- **Общий scope фикстуры БД** — если сделать `db_session` со `scope="session"`, тесты перестанут быть изолированными (данные накапливаются). Для изоляции — `scope="function"`.
- **`python-multipart` не установлен** — эндпоинты с `Form()`/`File()` выдадут ошибку импорта при старте.
- **Зависший event loop в pytest-asyncio** — конфликт фикстур разного scope с разными loop'ами; держите единый подход к scope event loop.

## Лучшие практики

- Прячьте всё «внешнее» (БД, аутентификацию, платёжки, очереди, S3) за зависимостями `Depends(...)` — это делает их подменяемыми в тестах.
- Очистку `dependency_overrides` делайте в teardown-части фикстуры (`yield`), а не вручную в каждом тесте.
- Используйте `scope="function"` для фикстур БД ради полной изоляции; для тяжёлой неизменяемой подготовки — `scope="session"`.
- Один источник правды для фикстур — `conftest.py`. Не дублируйте настройку клиента/БД в каждом модуле.
- Тестируйте `lifespan`/события явно, через `with TestClient(app) as client`.
- Проверяйте не только HTTP-ответ, но и побочные эффекты (записи в БД, вызовы фейкового сервиса).
- Для async-приложений с `AsyncSession` пишите async-тесты на `httpx.AsyncClient` + `ASGITransport`.
- Держите тестовую БД отдельной от dev/prod (отдельный движок/URL), никогда не запускайте тесты против боевой БД.
- Фабрики данных (фикстуры `seed_user`, `seed_items`) делают тесты читаемыми и DRY.

## Шпаргалка

```python
# --- Подмена зависимости ---
app.dependency_overrides[real_dep] = fake_dep
# ... тест ...
app.dependency_overrides.clear()

# --- Sync клиент + lifespan ---
with TestClient(app) as client:
    client.get("/path")

# --- Async клиент ---
from httpx import AsyncClient, ASGITransport
async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as ac:
    await ac.get("/path")

# --- Тестовая БД (in-memory SQLite) ---
create_engine("sqlite://", connect_args={"check_same_thread": False}, poolclass=StaticPool)
Base.metadata.create_all(bind=engine)   # setup
Base.metadata.drop_all(bind=engine)     # teardown

# --- WebSocket ---
with client.websocket_connect("/ws") as ws:
    ws.send_text("ping"); ws.receive_text()

# --- Файлы и формы ---
client.post("/upload",
            files={"file": ("a.txt", io.BytesIO(b"data"), "text/plain")},
            data={"description": "..."})

# --- Маркеры async-тестов ---
@pytest.mark.asyncio   # pytest-asyncio
@pytest.mark.anyio     # anyio (нужна фикстура anyio_backend -> "asyncio")
```

Краткий вывод: тестируемость FastAPI держится на DI. Подменяйте зависимости через `dependency_overrides`, изолируйте БД отдельным движком, запускайте `lifespan` через `with TestClient(...)`, а для async-кода используйте `httpx.AsyncClient` + `ASGITransport`. Общие фикстуры выносите в `conftest.py`.
