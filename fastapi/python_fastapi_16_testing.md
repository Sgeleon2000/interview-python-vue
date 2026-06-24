# FastAPI — Тестирование и отладка — конспект и вопросы

## О чём раздел

Как тестировать FastAPI-приложения и как их отлаживать. Основной инструмент —
`TestClient` (синхронный клиент поверх `httpx`/Starlette), который позволяет писать
тесты на `pytest` без запуска реального сервера. Рассмотрим:

- написание тестов: статус, JSON-ответ, тело/заголовки/параметры;
- фикстуры `pytest`;
- **переопределение зависимостей** через `app.dependency_overrides` (мок БД, аутентификации);
- организацию тестов;
- кратко про **async-тесты** (`httpx.AsyncClient` + `ASGITransport`);
- запуск под отладчиком (`if __name__ == "__main__"` + `uvicorn.run`, точки останова).

Хорошее покрытие тестами — обязательный навык middle/senior. FastAPI делает это удобным:
тесты выполняются быстро, без сети, с полным доступом к DI-контейнеру приложения.

## Ключевые концепции

- **`TestClient`** (`fastapi.testclient.TestClient`) — синхронный клиент на базе `httpx`,
  «дёргает» ASGI-приложение напрямую (без поднятия сети). Внутри умеет выполнять и
  async-эндпоинты, и lifespan/startup-shutdown (в контекстном менеджере).
- **`pytest`** — стандартный раннер тестов; функции `test_*`, ассерты на `response.status_code`
  и `response.json()`.
- **Фикстуры** — переиспользуемые ресурсы (клиент, БД, переопределения зависимостей).
- **`app.dependency_overrides`** — словарь `{оригинальная_зависимость: подмена}`; позволяет
  заменить любую `Depends`-зависимость на тестовую (мок БД, мок текущего пользователя).
- **Async-тесты** — для проверки реального async-поведения берут `httpx.AsyncClient`
  с `ASGITransport` (см. advanced-раздел документации FastAPI «Async Tests»).
- **Отладка** — запуск через `uvicorn.run` в `if __name__ == "__main__"` и точки останова
  в IDE / `breakpoint()`.

## Подробный разбор с примерами кода

### Простейший тест с `TestClient`

```python
# main.py
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
async def read_root():
    return {"message": "Hello World"}
```

```python
# test_main.py
from fastapi.testclient import TestClient

from main import app

client = TestClient(app)


def test_read_root():
    response = client.get("/")
    assert response.status_code == 200          # проверяем статус
    assert response.json() == {"message": "Hello World"}  # проверяем JSON
```

`TestClient` имеет httpx-подобный API: `client.get/post/put/delete/...`.

### Тело, заголовки, query-параметры

```python
def test_create_item():
    response = client.post(
        "/items/",
        json={"name": "Книга", "price": 9.99},   # тело запроса (JSON)
        headers={"X-Token": "secret"},            # заголовки
        params={"q": "test"},                     # query-параметры (?q=test)
    )
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Книга"
    assert "id" in data
    assert response.headers["content-type"] == "application/json"
```

Полезно:
- `json=...` — отправка JSON-тела;
- `data=...` / `files=...` — форма и файлы;
- `params=...` — query;
- `headers=...` — заголовки;
- `cookies=...` — куки.

### Проверка ошибок и валидации

```python
def test_item_not_found():
    response = client.get("/items/999")
    assert response.status_code == 404
    assert response.json() == {"detail": "Item not found"}


def test_validation_error():
    # price строкой — Pydantic вернёт 422.
    response = client.post("/items/", json={"name": "X", "price": "not-a-number"})
    assert response.status_code == 422
    body = response.json()
    assert body["detail"][0]["loc"][-1] == "price"
```

### Фикстуры pytest

`conftest.py` хранит общие фикстуры. Клиент как фикстура с `lifespan`:

```python
# conftest.py
import pytest
from fastapi.testclient import TestClient

from main import app


@pytest.fixture
def client():
    # Контекстный менеджер запустит startup/shutdown (lifespan) приложения.
    with TestClient(app) as c:
        yield c
```

```python
# test_items.py
def test_list_items(client):
    response = client.get("/items/")
    assert response.status_code == 200
```

Замечание: без `with` события lifespan/startup не выполнятся. Если приложение
инициализирует ресурсы в lifespan, используйте контекстный менеджер.

### Переопределение зависимостей: мок БД

Главный приём для тестов — заменить зависимость на тестовую через
`app.dependency_overrides`.

```python
# deps.py
from typing import Annotated
from fastapi import Depends


class Database:
    def get_user(self, user_id: int) -> dict:
        # Реальное обращение к БД...
        ...


def get_db() -> Database:
    return Database()


DBDep = Annotated[Database, Depends(get_db)]
```

```python
# main.py
from fastapi import FastAPI
from deps import DBDep

app = FastAPI()


@app.get("/users/{user_id}")
async def read_user(user_id: int, db: DBDep):
    return db.get_user(user_id)
```

```python
# test_users.py
import pytest
from fastapi.testclient import TestClient

from main import app
from deps import get_db


class FakeDB:
    def get_user(self, user_id: int) -> dict:
        # Детерминированные тестовые данные вместо реальной БД.
        return {"id": user_id, "name": "Тестовый пользователь"}


def override_get_db() -> FakeDB:
    return FakeDB()


@pytest.fixture
def client():
    # Подменяем зависимость get_db на фейковую.
    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as c:
        yield c
    # ВАЖНО: очищаем переопределения после теста, чтобы не «протекли» в другие.
    app.dependency_overrides.clear()


def test_read_user(client):
    response = client.get("/users/1")
    assert response.status_code == 200
    assert response.json() == {"id": 1, "name": "Тестовый пользователь"}
```

Ключ словаря — **оригинальная функция-зависимость** (`get_db`), значение — подмена.
FastAPI при разрешении `Depends(get_db)` подставит `override_get_db`.

### Переопределение аутентификации

```python
# security.py
from typing import Annotated
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

oauth2 = OAuth2PasswordBearer(tokenUrl="token")


async def get_current_user(token: Annotated[str, Depends(oauth2)]) -> dict:
    # Реальная проверка токена...
    if token != "valid":
        raise HTTPException(status.HTTP_401_UNAUTHORIZED)
    return {"username": "real_user"}
```

```python
# test_secure.py
from main import app
from security import get_current_user


async def fake_current_user() -> dict:
    # Подменяем аутентификацию — «всегда залогинен как admin».
    return {"username": "admin", "is_admin": True}


def test_protected(client_factory):
    app.dependency_overrides[get_current_user] = fake_current_user
    with TestClient(app) as client:
        # Токен можно не слать — зависимость замокана.
        resp = client.get("/me")
        assert resp.status_code == 200
        assert resp.json()["username"] == "admin"
    app.dependency_overrides.clear()
```

Это позволяет тестировать защищённые эндпоинты без реального логина.

### Тестирование БД через override + транзакции (паттерн)

Часто `get_db` подменяют так, чтобы он отдавал сессию к тестовой БД (SQLite in-memory
или отдельная схема), с откатом после теста:

```python
# Псевдокод паттерна для SQLAlchemy
def override_get_session():
    # Открываем сессию к тестовой БД, по завершении — откат/закрытие.
    session = TestingSessionLocal()
    try:
        yield session
    finally:
        session.rollback()
        session.close()

app.dependency_overrides[get_session] = override_get_session
```

### Кратко про async-тесты

`TestClient` синхронный и сам гоняет async-эндпоинты — этого достаточно для большинства
тестов. Но если нужно тестировать **именно асинхронно** (например, вызывать внутренние
async-функции в том же event loop), используют `httpx.AsyncClient` с `ASGITransport`:

```python
import pytest
import httpx
from httpx import ASGITransport

from main import app


@pytest.mark.anyio
async def test_root_async():
    transport = ASGITransport(app=app)
    async with httpx.AsyncClient(
        transport=transport, base_url="http://test"
    ) as ac:
        response = await ac.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}


@pytest.fixture
def anyio_backend():
    return "asyncio"
```

Понадобится `anyio` (или `pytest-asyncio`). Подробности — в advanced-разделе документации
FastAPI «Async Tests». Для большинства случаев достаточно обычного `TestClient`.

### Организация тестов

Типовая структура:

```
project/
├── app/
│   ├── main.py
│   ├── deps.py
│   └── routers/
└── tests/
    ├── conftest.py        # общие фикстуры (client, db, overrides)
    ├── test_users.py
    ├── test_items.py
    └── test_auth.py
```

- Общие фикстуры — в `conftest.py` (pytest подхватывает автоматически).
- Группируйте тесты по доменам/роутерам.
- Запуск: `pytest`, `pytest -v`, `pytest tests/test_users.py::test_read_user`.
- Покрытие: `pytest --cov=app`.

### Запуск под отладчиком

Чтобы отлаживать с точками останова прямо в IDE (PyCharm/VS Code), запускают приложение
как обычный Python-скрипт через `uvicorn.run`:

```python
# main.py
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
async def read_root():
    return {"message": "Hello World"}


if __name__ == "__main__":
    import uvicorn

    # Запуск напрямую: можно ставить breakpoint'ы и идти отладчиком.
    # ВНИМАНИЕ: reload=True мешает отладке (перезапускает процесс),
    # для отладки обычно reload=False.
    uvicorn.run(app, host="127.0.0.1", port=8000, reload=False)
```

Запуск файла «под отладкой» в IDE остановится на точках останова в обработчиках.
Альтернатива — `breakpoint()` в коде (встроенный pdb):

```python
@app.get("/debug/{x}")
async def debug(x: int):
    breakpoint()   # выполнение остановится здесь в pdb
    return {"x": x * 2}
```

В самих тестах удобно отлаживать через `pytest --pdb` (падает в pdb на ошибке).

## Полный рабочий пример

```python
# app/main.py
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, status
from pydantic import BaseModel


class Item(BaseModel):
    name: str
    price: float


class ItemOut(Item):
    id: int


# --- "БД" как зависимость ---
class Database:
    def __init__(self) -> None:
        self._items: dict[int, ItemOut] = {}
        self._seq = 0

    def create(self, item: Item) -> ItemOut:
        self._seq += 1
        out = ItemOut(id=self._seq, **item.model_dump())
        self._items[out.id] = out
        return out

    def get(self, item_id: int) -> ItemOut | None:
        return self._items.get(item_id)


_db_singleton = Database()


def get_db() -> Database:
    return _db_singleton


DBDep = Annotated[Database, Depends(get_db)]

app = FastAPI(title="Testing Demo")


@app.post("/items/", response_model=ItemOut, status_code=status.HTTP_201_CREATED)
async def create_item(item: Item, db: DBDep) -> ItemOut:
    return db.create(item)


@app.get("/items/{item_id}", response_model=ItemOut)
async def read_item(item_id: int, db: DBDep) -> ItemOut:
    item = db.get(item_id)
    if item is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, detail="Item not found")
    return item


if __name__ == "__main__":
    import uvicorn

    uvicorn.run(app, host="127.0.0.1", port=8000, reload=False)
```

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient

from app.main import app, get_db, Database


class FakeDB(Database):
    # Чистая БД на каждый тест (наследуем поведение, но изолируем состояние).
    pass


@pytest.fixture
def client():
    fake = FakeDB()
    app.dependency_overrides[get_db] = lambda: fake
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

```python
# tests/test_items.py
def test_create_and_read(client):
    # Создание
    resp = client.post("/items/", json={"name": "Книга", "price": 9.99})
    assert resp.status_code == 201
    created = resp.json()
    assert created["id"] == 1
    assert created["name"] == "Книга"

    # Чтение
    resp2 = client.get(f"/items/{created['id']}")
    assert resp2.status_code == 200
    assert resp2.json() == created


def test_not_found(client):
    resp = client.get("/items/999")
    assert resp.status_code == 404
    assert resp.json() == {"detail": "Item not found"}


def test_validation_error(client):
    resp = client.post("/items/", json={"name": "X", "price": "abc"})
    assert resp.status_code == 422
    assert resp.json()["detail"][0]["loc"][-1] == "price"
```

## Частые вопросы на собеседовании (Q/A)

**1. На чём основан `TestClient`?**
На `httpx` (исторически — `requests`), реализован в Starlette. Он вызывает ASGI-приложение
напрямую, без поднятия реального сетевого сервера.

**2. Можно ли тестировать async-эндпоинты обычным синхронным `TestClient`?**
Да. `TestClient` сам выполняет async-обработчики во внутреннем event loop. Тестовые
функции при этом обычные (`def`).

**3. Когда нужны async-тесты (`httpx.AsyncClient` + `ASGITransport`)?**
Когда нужно вызывать async-код в том же event loop (например, тестируемые async-функции,
работа с async-БД-сессиями), а не только дёргать HTTP. См. advanced «Async Tests».

**4. Как мокнуть БД в тестах?**
Через `app.dependency_overrides[get_db] = override`, где `override` возвращает фейковую
БД или тестовую сессию. Эндпоинты, зависящие от `get_db`, получат подмену.

**5. Что является ключом в `dependency_overrides`?**
Оригинальная функция-зависимость (та, что внутри `Depends(...)`), значение — функция-подмена.

**6. Почему важно очищать `dependency_overrides`?**
Переопределения хранятся на уровне `app` и сохраняются между тестами. Без `clear()`
они «протекут» и сломают изоляцию. Делайте `app.dependency_overrides.clear()` после теста
(или в фикстуре через `yield`).

**7. Зачем запускать `TestClient` через `with`?**
Контекстный менеджер запускает события `lifespan`/startup/shutdown. Без него ресурсы,
инициализируемые в lifespan, не поднимутся.

**8. Как протестировать защищённый эндпоинт без реального логина?**
Переопределить зависимость аутентификации (`get_current_user`) на функцию, возвращающую
фиктивного пользователя.

**9. Как проверить тело/заголовки/параметры запроса?**
Передавать в методы клиента `json=`, `data=`, `files=`, `headers=`, `params=`, `cookies=`.
Ответ: `response.status_code`, `response.json()`, `response.headers`.

**10. Как организовать общие фикстуры?**
В `conftest.py` (pytest автоматически их подхватывает во всех тестах каталога).

**11. Как отлаживать приложение с точками останова?**
Запустить как скрипт через `if __name__ == "__main__": uvicorn.run(app, ..., reload=False)`
под отладчиком IDE, или вставить `breakpoint()`. В тестах — `pytest --pdb`.

**12. Почему `reload=True` мешает отладке?**
Reload запускает дочерний процесс/перезагрузку при изменениях, из-за чего отладчик
теряет привязку к процессу и точки останова не срабатывают. Для отладки — `reload=False`.

**13. Как проверить ошибку валидации Pydantic?**
Ожидать статус `422` и проверить структуру `response.json()["detail"]` (список ошибок
с `loc`, `msg`, `type`).

**14. Как измерить покрытие тестами?**
`pytest --cov=app` (плагин `pytest-cov`).

## Подводные камни (gotchas)

- **Не очищенные `dependency_overrides`** протекают между тестами — всегда `clear()`.
- **Забытый `with TestClient(app)`** → lifespan/startup не выполнится, ресурсы не поднимутся.
- **Состояние между тестами** (общая БД-синглтон) ломает изоляцию — создавайте чистый
  стейт на каждый тест (фикстура).
- **`reload=True` при отладке** мешает точкам останова — используйте `reload=False`.
- **Подмена не той функции** — ключ override должен быть **точно** той функцией, что в `Depends`.
- **Async-БД без async-тестов** — синхронный `TestClient` гоняет всё в своём loop; для
  тонких async-сценариев берите `AsyncClient`.
- **Ожидание неправильного статуса** — помните про `422` для валидации, `201` для create и т.п.

## Лучшие практики

- Используйте `TestClient` для большинства тестов; `AsyncClient` — только когда реально нужно.
- Выносите клиент и переопределения в фикстуры (`conftest.py`), очищайте overrides после теста.
- Мокайте БД и аутентификацию через `dependency_overrides`, а не патчингом внутренностей.
- Делайте тесты изолированными: чистое состояние БД на каждый тест (in-memory SQLite/откат).
- Проверяйте и успешные сценарии, и ошибки (404/401/422), и контракты ответа (`response_model`).
- Для отладки держите `if __name__ == "__main__": uvicorn.run(app, reload=False)`.
- Меряйте покрытие (`pytest --cov`) и группируйте тесты по доменам.

## Шпаргалка

```python
from fastapi.testclient import TestClient
from main import app, get_db

client = TestClient(app)                  # синхронный клиент

# Запросы
client.get("/x", params={"q": "1"})
client.post("/x", json={...}, headers={...}, cookies={...})

# Проверки
assert r.status_code == 200
assert r.json() == {...}
assert r.headers["content-type"] == "application/json"

# Подмена зависимости (мок БД / auth)
app.dependency_overrides[get_db] = lambda: FakeDB()
# ...тест...
app.dependency_overrides.clear()          # ОБЯЗАТЕЛЬНО очистить

# lifespan в тестах
with TestClient(app) as c:
    c.get("/")

# Async-тест
import httpx
from httpx import ASGITransport
async with httpx.AsyncClient(transport=ASGITransport(app=app),
                             base_url="http://test") as ac:
    await ac.get("/")

# Отладка
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="127.0.0.1", port=8000, reload=False)  # reload=False!
```

| Задача                  | Инструмент                                  |
|-------------------------|---------------------------------------------|
| HTTP-тесты              | `TestClient` (httpx/Starlette)              |
| Раннер                  | `pytest`                                    |
| Общие фикстуры          | `conftest.py`                               |
| Мок БД / auth           | `app.dependency_overrides`                  |
| Async-тесты             | `httpx.AsyncClient` + `ASGITransport`       |
| lifespan в тесте        | `with TestClient(app) as c`                 |
| Отладка                 | `uvicorn.run(reload=False)` / `breakpoint()`|
| Покрытие                | `pytest --cov`                              |
