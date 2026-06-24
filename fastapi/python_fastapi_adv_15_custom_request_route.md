# FastAPI Advanced — Кастомные Request и APIRoute — конспект и вопросы

## О чём раздел

Этот раздел про самый «низкоуровневый» механизм расширения FastAPI: подмену классов **`Request`** и **`APIRoute`**, чтобы вклиниться в обработку запроса до и после вызова вашей функции-обработчика. Это уже территория Starlette, на которой стоит FastAPI.

Зачем это нужно:

- **Кастомный `Request`** — переопределить, как читается/кешируется тело запроса. Классика: тело можно прочитать только один раз (это поток); если нужно прочитать его несколько раз (логирование + валидация), нужно его закешировать. Сюда же — распаковка gzip-тела, кастомная десериализация, доступ к «сырому» телу в обработчике исключений.
- **Кастомный `APIRoute`** — обернуть обработчик роута (route handler), чтобы единообразно для всей группы эндпоинтов: замерять время выполнения, логировать, ловить и преобразовывать исключения, менять тело/заголовки запроса, добавлять метрики.

Главные точки расширения: атрибут **`route_class`** у `APIRouter` (или замена дефолтного класса роутов) и метод **`get_route_handler()`**, который возвращает корутину-обработчик запроса.

## Ключевые концепции

- **`Request` (Starlette)** — обёртка над ASGI-запросом: заголовки, query, путь, тело. Методы тела (`await request.body()`, `await request.json()`, `await request.form()`) читают поток; повторное чтение по умолчанию вернёт пустоту, поэтому Starlette кеширует `body()` внутри.
- **`APIRoute` (FastAPI)** — наследник Starlette `Route`, описывает один path operation: путь, методы, зависимости, response_model, и, главное, **route handler** — асинхронную функцию, которая принимает `Request` и возвращает `Response`.
- **`get_route_handler()`** — метод `APIRoute`, возвращающий тот самый обработчик (`async def custom_handler(request) -> Response`). Переопределив его, мы оборачиваем «оригинальный» обработчик своей логикой (before/after).
- **`route_class`** — атрибут `APIRouter`, через который назначается кастомный класс роута для всех эндпоинтов этого роутера. Можно задать на весь `app` через `app.router.route_class`.
- **`Request._receive`** — низкоуровневый ASGI-callable, через который читается тело. Чтобы «вернуть» уже прочитанное тело обработчику, подменяют `request._receive`.
- **Порядок применения** — кастомный route оборачивает выполнение path operation; это **не** то же самое, что middleware (middleware работает на уровне всего приложения и не имеет доступа к распарсенным зависимостям/телу так удобно).
- **Custom request vs middleware vs dependency** — три уровня вклинивания: middleware (всё приложение, до роутинга), кастомный route (группа эндпоинтов, после роутинга, есть доступ к телу/Response), dependency (логика «до» обработчика, удобный DI).

## Подробный разбор с примерами кода

### 1. Почему тело читается только один раз

```python
@app.post("/echo")
async def echo(request: Request):
    body1 = await request.body()   # читает поток
    body2 = await request.body()   # Starlette вернёт КЕШ (тот же body1)
    # А вот ручное чтение request.stream() во второй раз дало бы пустоту.
    return {"len": len(body1)}
```

Starlette кеширует результат `body()`. Проблема возникает, когда **разные участки кода** (например, middleware/обработчик исключений) хотят независимо прочитать тело, или когда нужно предобработать тело (gzip) до парсинга. Тут и нужен кастомный `Request`.

### 2. Кастомный класс Request: кеширование и доступ к «сырому» телу

```python
import gzip
from typing import Callable

from fastapi import FastAPI, APIRouter, Request, Response
from fastapi.routing import APIRoute


class CachedBodyRequest(Request):
    """Request, который кеширует тело и умеет распаковывать gzip."""

    async def body(self) -> bytes:
        # Кешируем в собственном атрибуте, чтобы тело можно было читать многократно
        # И из обработчика, и из обработчика исключений.
        if not hasattr(self, "_cached_body"):
            body = await super().body()
            # Если клиент прислал gzip — распакуем прозрачно.
            if "gzip" in self.headers.get("content-encoding", ""):
                body = gzip.decompress(body)
            self._cached_body = body
        return self._cached_body
```

### 3. Кастомный класс APIRoute: подключаем наш Request

```python
class CachedBodyRoute(APIRoute):
    def get_route_handler(self) -> Callable:
        # Берём «родной» обработчик, который умеет валидировать,
        # вызывать зависимости и саму path operation function.
        original_route_handler = super().get_route_handler()

        async def custom_route_handler(request: Request) -> Response:
            # Подменяем класс запроса на наш кешируемый/gzip-aware.
            request = CachedBodyRequest(request.scope, request.receive)
            return await original_route_handler(request)

        return custom_route_handler
```

### 4. Кастомный route для замера времени и логирования

```python
import time
import logging

logger = logging.getLogger("timing")


class TimedRoute(APIRoute):
    def get_route_handler(self) -> Callable:
        original_route_handler = super().get_route_handler()

        async def custom_route_handler(request: Request) -> Response:
            start = time.perf_counter()
            # Вызываем оригинальный обработчик (зависимости + path operation).
            response: Response = await original_route_handler(request)
            duration = time.perf_counter() - start
            # Кладём время в заголовок и логируем.
            response.headers["X-Response-Time"] = f"{duration:.4f}"
            logger.info("%s %s -> %.4fs", request.method, request.url.path, duration)
            return response

        return custom_route_handler
```

### 5. Кастомный route для обработки исключений на уровне роутера

```python
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse


class ValidationLoggingRoute(APIRoute):
    def get_route_handler(self) -> Callable:
        original_route_handler = super().get_route_handler()

        async def custom_route_handler(request: Request) -> Response:
            try:
                return await original_route_handler(request)
            except RequestValidationError as exc:
                # Имея кастомный Request, можем достать «сырое» тело,
                # которое вызвало ошибку валидации, и залогировать его.
                body = await request.body()
                detail = {"errors": exc.errors(), "body": body.decode("utf-8", "replace")}
                logger.warning("Ошибка валидации: %s", detail)
                return JSONResponse(status_code=422, content=detail)

        return custom_route_handler
```

Чтобы здесь сработало `await request.body()` после неудачной валидации, обычно комбинируют с кешируемым `Request` (как в п.2–3), иначе тело уже «израсходовано».

### 6. Подключение кастомного route: route_class

```python
# Вариант А: на весь роутер.
router = APIRouter(route_class=TimedRoute)


@router.get("/ping")
async def ping():
    return {"ok": True}


# Вариант Б: на всё приложение (после создания app).
app = FastAPI()
app.router.route_class = TimedRoute

app.include_router(router)
```

`route_class` применяется ко всем эндпоинтам этого роутера. Так можно держать разные группы эндпоинтов с разным поведением (один роутер — с таймингом, другой — с gzip-телом).

### 7. Кастомная десериализация тела (например, msgpack)

```python
import msgpack


class MsgpackRequest(Request):
    async def body(self) -> bytes:
        if not hasattr(self, "_cached_body"):
            self._cached_body = await super().body()
        return self._cached_body

    async def json(self):
        # Если тело пришло в msgpack — распакуем как dict,
        # чтобы дальше Pydantic валидировал как обычно.
        if self.headers.get("content-type") == "application/msgpack":
            return msgpack.unpackb(await self.body())
        return await super().json()


class MsgpackRoute(APIRoute):
    def get_route_handler(self) -> Callable:
        original = super().get_route_handler()

        async def handler(request: Request) -> Response:
            request = MsgpackRequest(request.scope, request.receive)
            return await original(request)

        return handler
```

Теперь Pydantic-валидация тела работает поверх данных, распакованных из msgpack, прозрачно для path operation function.

### 8. Подмена `_receive`, чтобы вернуть тело обработчику

Если в кастомном route вы читаете тело **до** оригинального обработчика (например, для логирования), сам обработчик потом получит пустой поток. Решение — «перезалить» уже прочитанное тело обратно:

```python
class LoggingBodyRoute(APIRoute):
    def get_route_handler(self) -> Callable:
        original = super().get_route_handler()

        async def handler(request: Request) -> Response:
            body = await request.body()       # читаем заранее
            logger.info("RAW BODY: %s", body[:500])

            # Возвращаем тело в поток для оригинального обработчика.
            async def receive():
                return {"type": "http.request", "body": body, "more_body": False}

            request._receive = receive
            return await original(request)

        return handler
```

## Полный рабочий пример

```python
# main.py — кастомный Request (кеш тела + gzip) и кастомный APIRoute (тайминг + логирование тела).
import gzip
import time
import logging
from typing import Callable

from fastapi import FastAPI, APIRouter, Request, Response
from fastapi.routing import APIRoute
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from pydantic import BaseModel

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("app")


# 1) Кастомный Request: кеш тела + прозрачная распаковка gzip.
class CachedGzipRequest(Request):
    async def body(self) -> bytes:
        if not hasattr(self, "_cached_body"):
            raw = await super().body()
            if "gzip" in self.headers.get("content-encoding", ""):
                raw = gzip.decompress(raw)
                # Вернём распакованное тело в поток, чтобы Pydantic его распарсил.
                async def receive():
                    return {"type": "http.request", "body": raw, "more_body": False}
                self._receive = receive
            self._cached_body = raw
        return self._cached_body


# 2) Кастомный route: подменяет Request, замеряет время, логирует тело при ошибке валидации.
class SmartRoute(APIRoute):
    def get_route_handler(self) -> Callable:
        original_route_handler = super().get_route_handler()

        async def custom_route_handler(request: Request) -> Response:
            request = CachedGzipRequest(request.scope, request.receive)
            start = time.perf_counter()
            try:
                response = await original_route_handler(request)
            except RequestValidationError as exc:
                body = await request.body()
                logger.warning("422 на %s, тело=%r", request.url.path, body[:300])
                return JSONResponse(
                    status_code=422,
                    content={"errors": exc.errors()},
                )
            duration = time.perf_counter() - start
            response.headers["X-Response-Time"] = f"{duration:.4f}"
            logger.info("%s %s -> %.4fs", request.method, request.url.path, duration)
            return response

        return custom_route_handler


app = FastAPI()
# Роутер с нашим поведением.
router = APIRouter(route_class=SmartRoute)


class Item(BaseModel):
    name: str
    price: float


@router.post("/items")
async def create_item(item: Item):
    # Path operation function не знает ни про gzip, ни про тайминг — всё прозрачно.
    return {"received": item}


app.include_router(router)

# Проверка gzip:
#   echo -n '{"name":"x","price":1.5}' | gzip | \
#     curl -s -X POST localhost:8000/items \
#     -H 'Content-Type: application/json' -H 'Content-Encoding: gzip' --data-binary @-
```

## Частые вопросы на собеседовании (Q/A)

**1. Зачем нужен кастомный класс Request?**
Чтобы изменить чтение/кеширование тела: читать тело несколько раз, распаковывать gzip/декодировать кастомный формат, иметь доступ к «сырому» телу в обработчике исключений.

**2. Почему тело запроса нельзя прочитать дважды по умолчанию?**
Тело — это ASGI-поток (`receive`); после прочтения он исчерпан. Starlette кеширует результат `body()`, но другие участки кода (middleware, обработчики ошибок) могут хотеть читать независимо — тогда нужен кастомный кеш.

**3. Что делает `get_route_handler()`?**
Возвращает асинхронный обработчик роута `async def handler(request) -> Response`, который вызывает зависимости и path operation function. Переопределяя его, оборачиваем оригинал своей логикой before/after.

**4. Как назначить кастомный класс роута?**
Через `APIRouter(route_class=MyRoute)` для роутера или `app.router.route_class = MyRoute` для всего приложения.

**5. Чем кастомный route отличается от middleware?**
Middleware работает на уровне всего приложения, до роутинга, и не имеет удобного доступа к распарсенным зависимостям/телу/исключениям конкретного эндпоинта. Кастомный route — после роутинга, для группы эндпоинтов, с прямым доступом к `Request`/`Response` и возможностью ловить ошибки валидации.

**6. Как поймать `RequestValidationError` и залогировать тело, которое его вызвало?**
Обернуть вызов оригинального обработчика в try/except в кастомном route; иметь кешируемый Request, чтобы `await request.body()` отдал тело уже после неудачной валидации.

**7. Зачем подменять `request._receive`?**
Если в route мы прочитали тело заранее, поток исчерпан и оригинальный обработчик получит пустоту. Подмена `_receive` «возвращает» прочитанные байты обратно в поток.

**8. Как сделать кастомную десериализацию (msgpack/cbor)?**
Переопределить `Request.json()` (или `body()`): если `content-type` нестандартный — распаковать своим декодером и вернуть dict; дальше Pydantic валидирует как обычно.

**9. Можно ли иметь разное поведение для разных групп эндпоинтов?**
Да: создать несколько `APIRouter` с разными `route_class` и подключить их в одно приложение.

**10. Кастомный route или зависимость для измерения времени?**
Зависимость работает «до» обработчика и не видит время формирования ответа целиком; кастомный route оборачивает весь вызов и может писать `X-Response-Time` в ответ. Для end-to-end тайминга предпочтителен route (или middleware).

**11. Влияет ли кастомный Request на валидацию Pydantic?**
Косвенно: Pydantic парсит то, что отдаст `request.json()/body()`. Подменив их, вы меняете вход валидации, не трогая саму модель.

**12. Какие риски у чтения/распаковки тела вручную?**
Память (распаковка больших gzip-«бомб»), двойное чтение потока, неверная синхронизация `_receive`. Нужны лимиты на размер тела и аккуратное кеширование.

## Подводные камни (gotchas)

- **Исчерпанный поток** — прочитали тело в route, забыли вернуть через `_receive` — обработчик получает пустое тело и падает на валидации.
- **Кеш в неправильном месте** — кешируйте в собственном атрибуте (`_cached_body`), не полагайтесь только на кеш Starlette, если читаете тело из нескольких слоёв.
- **gzip-бомбы / большие тела** — `gzip.decompress` распаковывает в память целиком; без лимита размера это DoS-вектор.
- **Порядок: route vs middleware** — route оборачивает только path operation; глобальные вещи (CORS, аутентификация для всего приложения) лучше в middleware.
- **`route_class` после include** — назначайте `route_class` до подключения роутера, иначе уже зарегистрированные маршруты не подхватят класс.
- **Исключения внутри route** — если ловите всё подряд, можно «проглотить» нужные HTTP-исключения; ловите конкретные типы.
- **Дублирование логики** — таймингом/логированием часто проще закрыть middleware, а кастомный route оставить там, где нужен доступ к телу/ошибкам.
- **Создание Request заново** — `MyRequest(request.scope, request.receive)` должно использовать актуальные `scope`/`receive`, иначе потеряете данные запроса.

## Лучшие практики

- Используйте кастомный `Request` только когда реально нужен доступ к телу несколько раз / кастомный формат / распаковка. Иначе — middleware или зависимость.
- Кешируйте тело в явном атрибуте и всегда корректно восстанавливайте поток через `_receive`.
- Ставьте **лимиты на размер тела** и осторожно относитесь к распаковке gzip.
- Разделяйте поведение по роутерам с разными `route_class`, а не плодите `if` в одном обработчике.
- В кастомном route ловите **конкретные** исключения (`RequestValidationError`, доменные), не `except Exception` без необходимости.
- Для end-to-end тайминга предпочитайте middleware/route, а не зависимость.
- Документируйте нестандартное поведение (gzip/msgpack) — оно невидимо в коде path operation и сбивает с толку.
- Покрывайте кастомные route/Request тестами через `TestClient` (включая ветки с ошибками валидации и нестандартными телами).

## Шпаргалка

```python
from fastapi import APIRouter, Request, Response
from fastapi.routing import APIRoute
from typing import Callable

# 1) Кастомный Request: кеш тела
class MyRequest(Request):
    async def body(self) -> bytes:
        if not hasattr(self, "_cached_body"):
            self._cached_body = await super().body()
        return self._cached_body

# 2) Кастомный APIRoute: обернуть обработчик
class MyRoute(APIRoute):
    def get_route_handler(self) -> Callable:
        original = super().get_route_handler()
        async def handler(request: Request) -> Response:
            request = MyRequest(request.scope, request.receive)  # подмена Request
            # ... before ...
            response = await original(request)
            # ... after ...
            return response
        return handler

# 3) Подключение
router = APIRouter(route_class=MyRoute)   # на роутер
app.router.route_class = MyRoute          # на всё приложение
```

| Задача | Где переопределять |
|---|---|
| Многократное чтение тела | `Request.body()` (кеш) |
| Распаковка gzip / кастомный формат | `Request.body()/json()` + `_receive` |
| Замер времени, заголовок X-Response-Time | `APIRoute.get_route_handler()` |
| Лог/обработка ошибок на уровне роутера | try/except в кастомном handler |
| Разное поведение по группам | разные `route_class` на разных `APIRouter` |
| Глобально на всё приложение | `app.router.route_class` |
