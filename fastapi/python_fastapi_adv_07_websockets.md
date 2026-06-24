# FastAPI Advanced — WebSockets — конспект и вопросы

## О чём раздел

**WebSocket** — это протокол двунаправленной (full-duplex) связи поверх одного TCP-соединения. В отличие от обычного HTTP, где клиент шлёт запрос и ждёт ответ, WebSocket держит соединение открытым: и сервер, и клиент могут отправлять сообщения в любой момент.

WebSocket используется там, где нужна **реалтайм-коммуникация**:
- чаты и мессенджеры;
- онлайн-игры;
- биржевые/спортивные тикеры, дашборды с живыми метриками;
- совместное редактирование документов;
- нотификации/пуши с сервера.

FastAPI (через Starlette) даёт декоратор `@app.websocket(path)` и объект `WebSocket` с асинхровыми методами для приёма/отправки данных. WebSocket-эндпоинты, как и обычные, поддерживают зависимости (`Depends`), что позволяет переиспользовать аутентификацию, доступ к БД и т.п.

## Ключевые концепции

- **Handshake (рукопожатие).** Соединение начинается как обычный HTTP-запрос `GET` с заголовком `Upgrade: websocket`. Сервер должен «принять» соединение методом `await websocket.accept()` — только после этого можно обмениваться сообщениями.
- **Объект `WebSocket`** — основной интерфейс:
  - `await websocket.accept()` — принять соединение.
  - `await websocket.receive_text()` / `receive_bytes()` / `receive_json()` — получить сообщение.
  - `await websocket.send_text()` / `send_bytes()` / `send_json()` — отправить сообщение.
  - `await websocket.close(code=1000)` — закрыть соединение.
  - `websocket.query_params`, `websocket.headers`, `websocket.cookies`, `websocket.path_params` — доступ к данным соединения.
- **Цикл обработки.** Обычно эндпоинт — это `while True:` цикл, который читает входящие сообщения и отвечает, пока соединение живо.
- **`WebSocketDisconnect`** — исключение, возникающее при разрыве соединения клиентом. Его нужно перехватывать, чтобы корректно убрать клиента из менеджера.
- **Зависимости в WebSocket.** Работают через `Depends`, но HTTP-only механизмы (`HTTPException`, `OAuth2PasswordBearer` напрямую) ведут себя иначе — для авторизации используют `WebSocketException` / закрытие с кодом.
- **Аутентификация** — обычно через токен в query-параметре, заголовке или cookie; проверяется до или сразу после `accept()`.
- **Менеджер подключений (ConnectionManager)** — паттерн для хранения активных соединений и широковещательной рассылки (broadcast), например в чате.
- **Коды закрытия** — числа по RFC 6455: `1000` — нормально, `1008` — policy violation (часто для отказа в авторизации), `1011` — внутренняя ошибка.

## Подробный разбор с примерами кода

### 1. Минимальный websocket-эндпоинт

```python
from fastapi import FastAPI, WebSocket

app = FastAPI()


@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    # 1. Принимаем соединение (обязательно!)
    await websocket.accept()
    # 2. Цикл обмена сообщениями
    while True:
        # ждём текстовое сообщение от клиента
        data = await websocket.receive_text()
        # отправляем ответ обратно (эхо)
        await websocket.send_text(f"Вы написали: {data}")
```

Без `accept()` соединение не установится. Если не закрыть и не выйти из цикла — корутина «висит» на `receive_text()` до разрыва.

### 2. Обработка разрыва соединения (WebSocketDisconnect)

Клиент может в любой момент закрыть вкладку/потерять сеть. Тогда `receive_*` бросает `WebSocketDisconnect`.

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()


@app.websocket("/ws")
async def ws_endpoint(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"echo: {data}")
    except WebSocketDisconnect:
        # клиент отключился — здесь чистим ресурсы (убираем из менеджера и т.п.)
        print("Клиент отключился")
```

### 3. Работа с JSON

```python
@app.websocket("/ws/json")
async def ws_json(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            payload = await websocket.receive_json()  # dict
            # эхо обратно с добавленным полем
            await websocket.send_json({"received": payload, "ok": True})
    except WebSocketDisconnect:
        pass
```

`receive_json()` сам парсит текстовый фрейм в Python-объект; `send_json()` сериализует и отправляет.

### 4. Зависимости в websocket-эндпоинтах

Зависимости работают так же, как в HTTP-эндпоинтах. Можно получать query-параметры, cookie, заголовки и общие зависимости (например, сессию БД).

```python
from typing import Annotated
from fastapi import (
    FastAPI, WebSocket, WebSocketException, Depends, Query, Cookie, status
)

app = FastAPI()


# Зависимость: достаём токен из query или cookie, при отсутствии — закрываем
async def get_token(
    websocket: WebSocket,
    token: Annotated[str | None, Query()] = None,
    session: Annotated[str | None, Cookie()] = None,
) -> str:
    value = token or session
    if value is None:
        # закрываем соединение с кодом policy violation
        raise WebSocketException(code=status.WS_1008_POLICY_VIOLATION)
    return value


@app.websocket("/ws/secure")
async def ws_secure(
    websocket: WebSocket,
    token: Annotated[str, Depends(get_token)],
):
    await websocket.accept()
    await websocket.send_text(f"Авторизован с токеном: {token}")
    try:
        while True:
            msg = await websocket.receive_text()
            await websocket.send_text(f"echo: {msg}")
    except WebSocketDisconnect:
        pass
```

Особенности:
- В websocket нельзя использовать `HTTPException` — вместо неё `WebSocketException(code=...)` или ручное `await websocket.close(code=...)`.
- Зависимость может объявить параметр `websocket: WebSocket`, чтобы иметь доступ к соединению.

### 5. Аутентификация по токену / cookie

Браузерный `WebSocket` API не позволяет легко слать кастомные заголовки, поэтому токен обычно передают:
- в query: `wss://host/ws?token=...` (просто, но токен светится в логах/URL);
- в cookie (если фронт и API на одном домене) — браузер отправит cookie автоматически при handshake;
- субпротоколом (`Sec-WebSocket-Protocol`) — более «чистый» способ.

```python
import jwt
from fastapi import WebSocket, WebSocketException, status

SECRET = "secret-key"


async def authenticate(websocket: WebSocket) -> dict:
    # 1) пробуем токен из query
    token = websocket.query_params.get("token")
    # 2) или из cookie
    if token is None:
        token = websocket.cookies.get("access_token")
    if token is None:
        raise WebSocketException(code=status.WS_1008_POLICY_VIOLATION)
    try:
        payload = jwt.decode(token, SECRET, algorithms=["HS256"])
    except jwt.PyJWTError:
        raise WebSocketException(code=status.WS_1008_POLICY_VIOLATION)
    return payload  # данные пользователя


@app.websocket("/ws/chat")
async def chat(websocket: WebSocket):
    user = await authenticate(websocket)  # до accept можно отклонить
    await websocket.accept()
    await websocket.send_json({"hello": user.get("sub")})
```

Замечание: при отклонении до `accept()` некоторые клиенты увидят просто закрытое соединение. Иногда удобнее сначала `accept()`, затем проверить и `close(code=1008)` с понятной причиной.

### 6. Менеджер подключений (broadcast, чат)

Чтобы рассылать сообщения нескольким клиентам, нужно хранить активные соединения. Классический паттерн — `ConnectionManager`.

```python
from fastapi import WebSocket


class ConnectionManager:
    def __init__(self) -> None:
        # список активных соединений
        self.active_connections: list[WebSocket] = []

    async def connect(self, websocket: WebSocket) -> None:
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket) -> None:
        # убираем соединение из списка
        if websocket in self.active_connections:
            self.active_connections.remove(websocket)

    async def send_personal(self, message: str, websocket: WebSocket) -> None:
        await websocket.send_text(message)

    async def broadcast(self, message: str) -> None:
        # рассылаем всем активным; «мёртвые» соединения убираем
        dead: list[WebSocket] = []
        for connection in self.active_connections:
            try:
                await connection.send_text(message)
            except Exception:
                dead.append(connection)
        for conn in dead:
            self.disconnect(conn)


manager = ConnectionManager()


@app.websocket("/ws/chat/{room}/{username}")
async def chat_room(websocket: WebSocket, room: str, username: str):
    await manager.connect(websocket)
    await manager.broadcast(f"[{room}] {username} присоединился")
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"[{room}] {username}: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast(f"[{room}] {username} вышел")
```

ВАЖНО: такой `ConnectionManager` хранит соединения **в памяти одного процесса**. При нескольких воркерах/инстансах broadcast не дойдёт до клиентов на других процессах — нужен внешний брокер (Redis Pub/Sub, NATS, Kafka).

### 7. Тестирование вебсокетов через TestClient

Starlette `TestClient` (он же `fastapi.testclient.TestClient`) даёт контекстный менеджер `websocket_connect`.

```python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)


def test_echo():
    with client.websocket_connect("/ws") as websocket:
        websocket.send_text("привет")
        data = websocket.receive_text()
        assert data == "Вы написали: привет"


def test_json():
    with client.websocket_connect("/ws/json") as websocket:
        websocket.send_json({"a": 1})
        resp = websocket.receive_json()
        assert resp["ok"] is True


def test_auth_required():
    from starlette.websockets import WebSocketDisconnect
    import pytest
    # без токена соединение должно закрыться
    with pytest.raises(WebSocketDisconnect):
        with client.websocket_connect("/ws/secure") as ws:
            ws.receive_text()


def test_with_token():
    with client.websocket_connect("/ws/secure?token=abc") as ws:
        msg = ws.receive_text()
        assert "abc" in msg
```

`TestClient` синхронный (методы без `await`), он сам крутит event loop. Для асинхронных тестов можно использовать `httpx`/`async` клиенты, но для websocket чаще достаточно `TestClient`.

## Полный рабочий пример

Чат-сервер с авторизацией по токену, менеджером подключений и комнатами.

```python
from contextlib import asynccontextmanager
from typing import Annotated

from fastapi import (
    FastAPI, WebSocket, WebSocketDisconnect, WebSocketException,
    Depends, Query, status,
)
from fastapi.responses import HTMLResponse


# --- Менеджер подключений с разбивкой по комнатам ---
class ConnectionManager:
    def __init__(self) -> None:
        # room -> список соединений
        self.rooms: dict[str, list[WebSocket]] = {}

    async def connect(self, room: str, websocket: WebSocket) -> None:
        await websocket.accept()
        self.rooms.setdefault(room, []).append(websocket)

    def disconnect(self, room: str, websocket: WebSocket) -> None:
        conns = self.rooms.get(room, [])
        if websocket in conns:
            conns.remove(websocket)
        if not conns:
            self.rooms.pop(room, None)

    async def broadcast(self, room: str, message: dict) -> None:
        dead: list[WebSocket] = []
        for conn in self.rooms.get(room, []):
            try:
                await conn.send_json(message)
            except Exception:
                dead.append(conn)
        for conn in dead:
            self.disconnect(room, conn)


manager = ConnectionManager()


# --- Современный lifespan вместо on_event ---
@asynccontextmanager
async def lifespan(app: FastAPI):
    # здесь могли бы подключить Redis Pub/Sub для multi-worker broadcast
    print("Чат-сервер запущен")
    yield
    print("Чат-сервер остановлен")


app = FastAPI(lifespan=lifespan, title="WebSocket chat")


# --- Зависимость авторизации (токен из query) ---
async def get_current_user(
    websocket: WebSocket,
    token: Annotated[str | None, Query()] = None,
) -> str:
    # В реале здесь декодируем JWT и достаём пользователя
    fake_users = {"t-alice": "alice", "t-bob": "bob"}
    username = fake_users.get(token or "")
    if username is None:
        raise WebSocketException(code=status.WS_1008_POLICY_VIOLATION)
    return username


@app.websocket("/ws/{room}")
async def chat_endpoint(
    websocket: WebSocket,
    room: str,
    user: Annotated[str, Depends(get_current_user)],
):
    await manager.connect(room, websocket)
    await manager.broadcast(room, {"system": f"{user} вошёл в комнату {room}"})
    try:
        while True:
            data = await websocket.receive_json()
            text = data.get("text", "")
            await manager.broadcast(room, {"user": user, "text": text})
    except WebSocketDisconnect:
        manager.disconnect(room, websocket)
        await manager.broadcast(room, {"system": f"{user} покинул комнату"})


# --- Простая HTML-страница для ручной проверки ---
@app.get("/")
async def index() -> HTMLResponse:
    html = """
    <!DOCTYPE html>
    <html><body>
      <h3>Чат (комната general, токен t-alice)</h3>
      <input id="msg"/><button onclick="send()">Отправить</button>
      <ul id="log"></ul>
      <script>
        const ws = new WebSocket("ws://localhost:8000/ws/general?token=t-alice");
        ws.onmessage = (e) => {
          const li = document.createElement("li");
          li.textContent = e.data;
          document.getElementById("log").appendChild(li);
        };
        function send() {
          const inp = document.getElementById("msg");
          ws.send(JSON.stringify({text: inp.value}));
          inp.value = "";
        }
      </script>
    </body></html>
    """
    return HTMLResponse(html)
```

Тест к этому примеру:

```python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)


def test_chat_broadcast():
    with client.websocket_connect("/ws/general?token=t-alice") as alice:
        # alice получает системное сообщение о своём входе
        msg = alice.receive_json()
        assert "alice" in msg["system"]
        with client.websocket_connect("/ws/general?token=t-bob") as bob:
            # alice видит, что bob вошёл
            join = alice.receive_json()
            assert "bob" in join["system"]
            bob.receive_json()  # bob видит свой вход
            # bob пишет — alice получает
            bob.send_json({"text": "привет"})
            received = alice.receive_json()
            assert received == {"user": "bob", "text": "привет"}
```

## Частые вопросы на собеседовании (Q/A)

**1. Чем WebSocket отличается от HTTP?**
HTTP — запрос/ответ, соединение обычно закрывается после ответа, сервер не может сам инициировать передачу. WebSocket — постоянное двунаправленное соединение поверх одного TCP, и сервер, и клиент шлют сообщения в любой момент. Идеален для реалтайма.

**2. Зачем нужен `await websocket.accept()`?**
Соединение начинается как HTTP-handshake с `Upgrade`. Пока сервер не вызовет `accept()`, обмена сообщениями нет. До `accept()` можно отклонить соединение (например, при отказе в авторизации).

**3. Как обрабатывать отключение клиента?**
Перехватывать `WebSocketDisconnect`, которое бросают `receive_*` при разрыве. В обработчике убирать соединение из менеджера и освобождать ресурсы.

**4. Как реализовать broadcast нескольким клиентам?**
Хранить активные соединения (паттерн ConnectionManager) и в цикле вызывать `send_*` для каждого. Не забывать убирать «мёртвые» соединения и обрабатывать ошибки отправки.

**5. Работают ли зависимости (`Depends`) в websocket-эндпоинтах?**
Да. Можно внедрять query/cookie/header-параметры и общие зависимости (БД, сервисы). Отличие: вместо `HTTPException` используют `WebSocketException(code=...)` или `await websocket.close(code=...)`.

**6. Как аутентифицировать websocket-соединение?**
Передать токен в query-параметре, cookie или субпротоколе (заголовки из браузера слать неудобно). Проверить токен до или сразу после `accept()`, при ошибке — закрыть с кодом `1008` (policy violation).

**7. Почему нельзя использовать `HTTPException` в websocket?**
WebSocket после handshake не работает по HTTP-семантике статус-кодов в теле ответа. Для ошибок используются коды закрытия WebSocket (RFC 6455). FastAPI предоставляет `WebSocketException`.

**8. Как тестировать websocket?**
Через `TestClient.websocket_connect(...)` как контекстный менеджер: `send_text/send_json` и `receive_text/receive_json`. Для проверки отказа в подключении ловят `WebSocketDisconnect`.

**9. Что произойдёт с in-memory ConnectionManager при нескольких воркерах?**
Каждый процесс хранит свой список соединений, broadcast не дойдёт до клиентов на других воркерах. Решение — внешний брокер (Redis Pub/Sub, NATS, Kafka) или sticky-сессии + общий канал.

**10. Что такое коды закрытия WebSocket?**
Числа по RFC 6455: `1000` — нормальное закрытие, `1001` — уход (going away), `1008` — нарушение политики (часто для авторизации), `1011` — внутренняя ошибка сервера.

**11. В чём разница `receive_text`, `receive_bytes`, `receive_json`?**
`receive_text` — текстовый фрейм (str), `receive_bytes` — бинарный (bytes), `receive_json` — текстовый фрейм, распарсенный в Python-объект. Несоответствие типа фрейма ожиданию приведёт к ошибке.

**12. Как масштабировать WebSocket-приложение?**
Вынести состояние подключений из процесса (брокер сообщений), использовать sticky-сессии на балансировщике, ограничивать число соединений, обрабатывать heartbeat/ping-pong и таймауты для «мёртвых» соединений.

## Подводные камни (gotchas)

- **Забыли `accept()`** — соединение не установится, клиент получит ошибку.
- **Не перехватили `WebSocketDisconnect`** — в логах сыплются необработанные исключения, соединения «протекают» в менеджере.
- **In-memory менеджер не работает с несколькими воркерами/инстансами** — нужен брокер.
- **`HTTPException` в websocket не сработает как ожидается** — используйте `WebSocketException`/`close`.
- **Блокирующий код в эндпоинте** (CPU-bound, sync IO) блокирует event loop и все соединения — выносите в пул/воркеры.
- **Неубранные «мёртвые» соединения** при broadcast приводят к ошибкам и утечкам — оборачивайте `send` в try/except и чистите список.
- **Токен в query** попадает в логи прокси/сервера — для чувствительных систем предпочтительнее cookie или субпротокол.
- **Отсутствие ping/pong и таймаутов** — «полумёртвые» соединения копятся; добавляйте heartbeat.
- **Гонки при изменении общего списка соединений** из разных корутин — для одного event loop это безопасно при синхронных операциях со списком, но при async-итерации копируйте список перед рассылкой.

## Лучшие практики

- Всегда вызывайте `accept()` и оборачивайте основной цикл в `try/except WebSocketDisconnect`.
- Выносите управление соединениями в `ConnectionManager`; чистите соединения при отключении и ошибках отправки.
- Для многопроцессного/многосерверного развёртывания используйте внешний брокер (Redis Pub/Sub и т.п.).
- Аутентифицируйте соединение до начала обмена; при отказе закрывайте с осмысленным кодом (`1008`).
- Не выполняйте блокирующие операции в обработчике; используйте async-библиотеки.
- Реализуйте heartbeat/ping и таймауты для отлова мёртвых соединений.
- Валидируйте входящие сообщения (например, через Pydantic-модель) перед обработкой.
- Покрывайте websocket-логику тестами через `TestClient.websocket_connect`.
- Ограничивайте число соединений и размер сообщений для защиты от перегрузки.

## Шпаргалка

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, WebSocketException, status

app = FastAPI()

@app.websocket("/ws")
async def ws(websocket: WebSocket):
    await websocket.accept()                    # принять соединение
    try:
        while True:
            data = await websocket.receive_text()   # receive_bytes / receive_json
            await websocket.send_text(data)          # send_bytes / send_json
    except WebSocketDisconnect:
        ...                                          # клиент отключился

# Доступ к данным соединения
# websocket.query_params, websocket.cookies, websocket.headers, websocket.path_params

# Закрытие / отказ
# await websocket.close(code=1000)
# raise WebSocketException(code=status.WS_1008_POLICY_VIOLATION)

# Менеджер подключений (broadcast)
class ConnectionManager:
    def __init__(self): self.active = []
    async def connect(self, ws): await ws.accept(); self.active.append(ws)
    def disconnect(self, ws): self.active.remove(ws)
    async def broadcast(self, msg):
        for ws in list(self.active):
            await ws.send_text(msg)

# Тест
from fastapi.testclient import TestClient
client = TestClient(app)
with client.websocket_connect("/ws") as ws:
    ws.send_text("hi")
    assert ws.receive_text() == "hi"
```
