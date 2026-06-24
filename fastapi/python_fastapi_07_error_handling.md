# FastAPI — Обработка ошибок — конспект и вопросы

## О чём раздел

Раздел посвящён обработке ошибок в FastAPI: как сообщать клиенту, что запрос
не может быть выполнен (неверные данные, отсутствует ресурс, нет прав и т.д.).
В HTTP это выражается через коды состояния семейства `4xx` (ошибка клиента) и
`5xx` (ошибка сервера).

В FastAPI «нормальный» возврат значения из обработчика превращается в успешный
ответ (`200 OK` по умолчанию). Чтобы вернуть ошибку, нужно либо **поднять
исключение** (`raise HTTPException(...)`), либо настроить **кастомный
обработчик исключений** (`@app.exception_handler(...)`), который перехватит
ваше собственное исключение и сформирует ответ.

Ключевые инструменты раздела:

- `HTTPException` — стандартный способ вернуть HTTP-ошибку с кодом, текстом и
  заголовками.
- `@app.exception_handler(...)` — глобальные обработчики для собственных и
  встроенных исключений.
- `RequestValidationError` — ошибка валидации **входящих** данных (тело,
  query, path), даёт ответ `422 Unprocessable Entity`.
- `ResponseValidationError` — ошибка валидации **исходящих** данных (когда ваш
  код вернул то, что не соответствует `response_model`), даёт `500`.
- Переиспользование обработчиков Starlette (`http_exception_handler`,
  `request_validation_exception_handler`).

## Ключевые концепции

- **`raise`, а не `return`.** Чтобы прервать выполнение и вернуть ошибку, вы
  именно поднимаете исключение `HTTPException`. FastAPI поймает его и сформирует
  корректный HTTP-ответ.
- **`HTTPException` — это обычный Python-exception**, поэтому его можно поднять
  из любого места: из обработчика, из зависимости (`Depends`), из утилитной
  функции. Везде, где он будет поднят в рамках обработки запроса, FastAPI его
  перехватит.
- **`detail`** может быть не только строкой, а любым JSON-сериализуемым
  объектом (`dict`, `list`), что удобно для машинно-читаемых ошибок.
- **`422 Unprocessable Entity`** — стандартный код для ошибок валидации входных
  данных. Тело такого ответа имеет фиксированную структуру со списком ошибок.
- **Глобальные обработчики** позволяют единообразно форматировать ошибки по
  всему приложению (например, оборачивать всё в `{"error": ...}`).
- **Ошибка валидации запроса (`RequestValidationError`)** ≠ **ошибка валидации
  ответа (`ResponseValidationError`)**. Первая — вина клиента (`422`), вторая —
  вина вашего кода (`500`).
- **Starlette `HTTPException` vs FastAPI `HTTPException`.** FastAPI наследует
  свой `HTTPException` от Starlette и добавляет поддержку любого JSON в
  `detail`. Регистрировать обработчик нужно на класс Starlette, если хотите
  ловить ошибки, поднятые внутренними механизмами Starlette.

## Подробный разбор с примерами кода

### 1. `HTTPException`: status_code, detail, headers

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

# Простейшее «хранилище» для примера
items = {"foo": "Кот по имени Фу"}


@app.get("/items/{item_id}")
async def read_item(item_id: str):
    # Если ресурса нет — поднимаем 404, а не возвращаем что-то «пустое»
    if item_id not in items:
        raise HTTPException(
            status_code=404,
            detail="Item не найден",  # уйдёт в тело ответа как {"detail": ...}
        )
    return {"item": items[item_id]}
```

Тело ответа при ошибке:

```json
{
  "detail": "Item не найден"
}
```

`HTTPException` принимает три основных аргумента:

- `status_code: int` — код HTTP-ответа (используйте константы из
  `fastapi.status` или `starlette.status` для читаемости).
- `detail` — содержимое поля `detail` в теле ответа. Может быть строкой,
  `dict`, `list` — любым JSON-сериализуемым значением.
- `headers: dict[str, str] | None` — дополнительные HTTP-заголовки ответа.

### 2. Когда поднимать `HTTPException`

Поднимайте `HTTPException`, когда **клиент** сделал что-то некорректное или
запросил недоступное:

- `400 Bad Request` — некорректный запрос на уровне бизнес-логики.
- `401 Unauthorized` — не аутентифицирован.
- `403 Forbidden` — аутентифицирован, но нет прав.
- `404 Not Found` — ресурс не найден.
- `409 Conflict` — конфликт состояния (например, дубликат).
- `422 Unprocessable Entity` — обычно генерируется автоматически при ошибке
  валидации, вручную поднимать его для «семантических» ошибок тоже допустимо.

Не используйте `HTTPException` для внутренних сбоев — там уместнее дать
исключению «всплыть» (тогда FastAPI вернёт `500`) или поднять собственное
исключение с глобальным обработчиком.

### 3. Использование `fastapi.status`

```python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()


@app.get("/users/{user_id}")
async def read_user(user_id: int):
    if user_id != 1:
        # Читается лучше, чем «магическое» число 404
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Пользователь не найден",
        )
    return {"user_id": user_id, "name": "Алиса"}
```

### 4. Добавление заголовков к ошибкам

Иногда стандарт требует добавить заголовки к ответу-ошибке. Классический
пример — заголовок `WWW-Authenticate` при `401`, или `Retry-After` при
`429`/`503`.

```python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()


@app.get("/items-header/{item_id}")
async def read_item_header(item_id: str):
    if item_id != "foo":
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Item не найден",
            headers={"X-Error": "Тут какое-то описание ошибки"},
        )
    return {"item": item_id}


@app.get("/secret")
async def secret():
    # Пример: требуем авторизацию и сообщаем схему через заголовок
    raise HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Требуется авторизация",
        headers={"WWW-Authenticate": "Bearer"},
    )
```

### 5. Машинно-читаемый `detail` (dict/list)

```python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()


@app.post("/orders")
async def create_order(order_id: str):
    if order_id == "exists":
        # detail может быть произвольной JSON-структурой
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail={
                "code": "ORDER_ALREADY_EXISTS",
                "message": "Заказ с таким id уже существует",
                "order_id": order_id,
            },
        )
    return {"order_id": order_id, "status": "created"}
```

### 6. Собственные классы исключений и `@app.exception_handler`

Можно завести своё доменное исключение и зарегистрировать для него обработчик.
Это удобно: бизнес-код поднимает понятное исключение, а превращение его в
HTTP-ответ описано в одном месте.

```python
from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse

app = FastAPI()


# Собственное исключение — не наследник HTTPException
class UnicornException(Exception):
    def __init__(self, name: str):
        self.name = name


# Обработчик: получает запрос и экземпляр исключения, возвращает Response
@app.exception_handler(UnicornException)
async def unicorn_exception_handler(request: Request, exc: UnicornException):
    return JSONResponse(
        status_code=status.HTTP_418_IM_A_TEAPOT,
        content={"message": f"Упс! {exc.name} сделал что-то странное..."},
    )


@app.get("/unicorns/{name}")
async def read_unicorn(name: str):
    if name == "yolo":
        # Поднимаем доменное исключение — обработчик сам сформирует ответ
        raise UnicornException(name=name)
    return {"unicorn_name": name}
```

Обработчик исключений всегда:

- принимает `request: Request` и `exc: <ТипИсключения>`;
- возвращает любой `Response` (обычно `JSONResponse`);
- может быть `async` или обычной функцией.

### 7. Переопределение обработчика `RequestValidationError`

`RequestValidationError` поднимается, когда **входные данные** не прошли
валидацию Pydantic (неверный тип, отсутствует обязательное поле и т.д.).
По умолчанию FastAPI отвечает `422` со структурированным телом. Можно
переопределить формат:

```python
from fastapi import FastAPI, Request, status
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse, PlainTextResponse

app = FastAPI()


@app.exception_handler(RequestValidationError)
async def validation_exception_handler(
    request: Request, exc: RequestValidationError
):
    # exc.errors() — список ошибок; exc.body — тело, которое пытались разобрать
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={
            "detail": exc.errors(),  # подробности об ошибках
            "body": exc.body,        # что прислал клиент
        },
    )


# Альтернатива — вернуть простой текст
# @app.exception_handler(RequestValidationError)
# async def validation_text_handler(request: Request, exc: RequestValidationError):
#     return PlainTextResponse(str(exc), status_code=400)
```

### 8. Переопределение обработчика `HTTPException`

Можно поменять формат ответа и для самих `HTTPException` (например,
возвращать текст вместо JSON или оборачивать в свою структуру). Важно:
регистрировать обработчик нужно на **`StarletteHTTPException`**, потому что
именно его FastAPI/Starlette используют под капотом.

```python
from fastapi import FastAPI, HTTPException
from fastapi.responses import PlainTextResponse
from starlette.exceptions import HTTPException as StarletteHTTPException

app = FastAPI()


@app.exception_handler(StarletteHTTPException)
async def http_exception_handler(request, exc: StarletteHTTPException):
    # Вернём ошибку как plain text вместо JSON
    return PlainTextResponse(str(exc.detail), status_code=exc.status_code)


@app.get("/items/{item_id}")
async def read_item(item_id: int):
    if item_id == 3:
        raise HTTPException(status_code=418, detail="Нельзя использовать 3")
    return {"item_id": item_id}
```

### 9. Повторное использование обработчиков Starlette

Часто нужно не полностью заменить поведение по умолчанию, а **дополнить** его:
залогировать, добавить заголовок, а затем отдать стандартный ответ. Для этого
вызывают встроенные обработчики Starlette из своих.

```python
from fastapi import FastAPI, HTTPException
from fastapi.exceptions import RequestValidationError
from starlette.exceptions import HTTPException as StarletteHTTPException
# Встроенные обработчики по умолчанию
from fastapi.exception_handlers import (
    http_exception_handler,
    request_validation_exception_handler,
)

app = FastAPI()


@app.exception_handler(StarletteHTTPException)
async def custom_http_exception_handler(request, exc):
    print(f"OMG! Произошла HTTP-ошибка: {exc.detail!r}")  # логируем
    # Делегируем формирование ответа стандартному обработчику
    return await http_exception_handler(request, exc)


@app.exception_handler(RequestValidationError)
async def custom_validation_handler(request, exc):
    print(f"OMG! Ошибка валидации: {exc!r}")
    return await request_validation_exception_handler(request, exc)


@app.get("/items/{item_id}")
async def read_item(item_id: int):
    if item_id == 3:
        raise HTTPException(status_code=418, detail="Nope! 3 нельзя.")
    return {"item_id": item_id}
```

### 10. `RequestValidationError` vs `ResponseValidationError`

```python
from typing import Annotated

from fastapi import FastAPI, Path
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    title: str
    size: int  # обязателен и должен быть int


# RequestValidationError: если клиент пришлёт size="abc" — будет 422
@app.post("/items/")
async def create_item(item: Item):
    return item


# ResponseValidationError: response_model требует size:int,
# а мы возвращаем строку — FastAPI вернёт 500 (ошибка на нашей стороне)
@app.get("/broken/{item_id}", response_model=Item)
async def broken(item_id: Annotated[int, Path()]):
    # Намеренная ошибка: size — строка, не проходит валидацию ответа
    return {"title": "сломано", "size": "не число"}
```

Резюме различий:

| | `RequestValidationError` | `ResponseValidationError` |
|---|---|---|
| Когда | вход не прошёл валидацию | выход не соответствует `response_model` |
| Чья вина | клиента | разработчика |
| HTTP-код | `422` | `500` |
| Где ловить | `@app.exception_handler(RequestValidationError)` | обычно не переопределяют |

### 11. Формат ошибок валидации (`422`)

Тело стандартного `422`-ответа выглядит так (Pydantic v2):

```json
{
  "detail": [
    {
      "type": "int_parsing",
      "loc": ["body", "size"],
      "msg": "Input should be a valid integer, unable to parse string as an integer",
      "input": "abc",
      "url": "https://errors.pydantic.dev/2/v/int_parsing"
    }
  ]
}
```

- `type` — машинный код ошибки Pydantic.
- `loc` — «путь» к проблемному полю: первый элемент — источник (`body`,
  `query`, `path`, `header`, `cookie`), далее — имена полей/индексы.
- `msg` — человекочитаемое сообщение.
- `input` — значение, которое не прошло проверку.
- `url` — ссылка на описание ошибки Pydantic.

## Полный рабочий пример

```python
from typing import Annotated

from fastapi import FastAPI, HTTPException, Path, Request, status
from fastapi.exceptions import RequestValidationError
from fastapi.exception_handlers import (
    http_exception_handler,
    request_validation_exception_handler,
)
from fastapi.responses import JSONResponse
from pydantic import BaseModel, Field
from starlette.exceptions import HTTPException as StarletteHTTPException

app = FastAPI(title="Демо обработки ошибок")


# ---------- Модели ----------
class Item(BaseModel):
    name: str = Field(..., min_length=1)
    price: float = Field(..., gt=0)


# «База данных» в памяти
fake_db: dict[int, Item] = {1: Item(name="Меч", price=99.9)}


# ---------- Собственное доменное исключение ----------
class ItemNotFoundError(Exception):
    def __init__(self, item_id: int):
        self.item_id = item_id


# ---------- Обработчики исключений ----------
@app.exception_handler(ItemNotFoundError)
async def item_not_found_handler(request: Request, exc: ItemNotFoundError):
    # Единое место форматирования ошибки «не найдено»
    return JSONResponse(
        status_code=status.HTTP_404_NOT_FOUND,
        content={
            "error": "ITEM_NOT_FOUND",
            "message": f"Item с id={exc.item_id} не найден",
        },
    )


@app.exception_handler(StarletteHTTPException)
async def custom_http_exception_handler(request: Request, exc: StarletteHTTPException):
    # Логируем и делегируем стандартному обработчику
    print(f"[HTTP {exc.status_code}] {exc.detail!r}")
    return await http_exception_handler(request, exc)


@app.exception_handler(RequestValidationError)
async def custom_validation_handler(request: Request, exc: RequestValidationError):
    # Оборачиваем стандартный 422 в свою структуру
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={"error": "VALIDATION_ERROR", "detail": exc.errors()},
    )


# ---------- Эндпоинты ----------
@app.get("/items/{item_id}")
async def get_item(item_id: Annotated[int, Path(ge=1)]):
    # Поднимаем доменное исключение — обработчик сформирует 404
    if item_id not in fake_db:
        raise ItemNotFoundError(item_id=item_id)
    return fake_db[item_id]


@app.post("/items/{item_id}", status_code=status.HTTP_201_CREATED)
async def create_item(item_id: Annotated[int, Path(ge=1)], item: Item):
    if item_id in fake_db:
        # Конфликт состояния — поднимаем HTTPException с заголовком
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="Item с таким id уже существует",
            headers={"X-Conflict-Id": str(item_id)},
        )
    fake_db[item_id] = item
    return item
```

## Частые вопросы на собеседовании (Q/A)

**1. В чём разница между `return` и `raise HTTPException` в обработчике?**
`return` формирует **успешный** ответ (по умолчанию `200`). Чтобы вернуть
ошибку и прервать выполнение, нужно поднять исключение `raise
HTTPException(...)`. FastAPI поймает его и построит ответ с нужным кодом.

**2. Какие аргументы принимает `HTTPException`?**
`status_code` (обязательный код HTTP), `detail` (содержимое тела, может быть
строкой, dict или list) и `headers` (доп. заголовки ответа).

**3. Может ли `detail` быть не строкой?**
Да. `detail` принимает любое JSON-сериализуемое значение, поэтому можно отдавать
структурированные ошибки: `detail={"code": "...", "message": "..."}`.

**4. Откуда можно поднимать `HTTPException`?**
Из любого места в рамках обработки запроса: из обработчика, из зависимости
(`Depends`), из вложенной функции. Везде FastAPI его перехватит. Это удобно для
зависимостей-проверок (например, проверка прав).

**5. Что такое `RequestValidationError` и когда он возникает?**
Это исключение, которое FastAPI поднимает при неуспешной валидации **входящих**
данных (тело, query, path, headers, cookies). По умолчанию приводит к ответу
`422` со списком ошибок в поле `detail`.

**6. Чем `RequestValidationError` отличается от `ResponseValidationError`?**
Первый — ошибка валидации **входных** данных (вина клиента, `422`). Второй —
ошибка валидации **исходящих** данных относительно `response_model` (вина
разработчика, `500`). `ResponseValidationError` сигнализирует о баге в коде.

**7. Как зарегистрировать кастомный обработчик исключений?**
Декоратором `@app.exception_handler(ТипИсключения)` над функцией, которая
принимает `(request, exc)` и возвращает `Response` (обычно `JSONResponse`).

**8. Почему обработчик `HTTPException` регистрируют на `StarletteHTTPException`,
а не на `fastapi.HTTPException`?**
FastAPI наследует свой `HTTPException` от Starlette, и многие внутренние
механизмы поднимают именно Starlette-версию. Регистрация на
`StarletteHTTPException` гарантирует, что обработчик поймает все такие ошибки.

**9. Как переопределить формат `422`, но сохранить стандартную информацию?**
Зарегистрировать `@app.exception_handler(RequestValidationError)` и
использовать `exc.errors()` (список ошибок) и `exc.body` (присланное тело),
обернув их в свою структуру.

**10. Как переиспользовать стандартные обработчики Starlette?**
Импортировать `http_exception_handler` и
`request_validation_exception_handler` из `fastapi.exception_handlers`,
выполнить свою логику (логирование и т.п.) и вернуть `await
http_exception_handler(request, exc)`.

**11. Как добавить заголовки к ответу-ошибке?**
Передать `headers={...}` в `HTTPException`, либо в кастомном обработчике
создать `JSONResponse(..., headers={...})`. Типичный пример —
`WWW-Authenticate` при `401`.

**12. Какая структура у стандартного тела ошибки валидации?**
`{"detail": [ {"type", "loc", "msg", "input", "url"} , ... ]}`. `loc` —
путь к полю, начинается с источника (`body`/`query`/`path`/...).

**13. Стоит ли использовать собственные классы исключений?**
Да, для доменной логики это чисто: код поднимает понятное исключение
(`ItemNotFoundError`), а маппинг в HTTP описан в одном обработчике. Это
уменьшает дублирование `HTTPException` по коду.

**14. Что произойдёт, если в обработчике возникнет необработанное (не HTTP)
исключение?**
FastAPI/Starlette вернёт `500 Internal Server Error`. В production детали не
показываются клиенту; в режиме отладки можно увидеть traceback.

## Подводные камни (gotchas)

- **`return HTTPException(...)` не работает.** Нужно именно `raise`. Возврат
  объекта исключения отдаст его как обычный JSON со статусом `200`.
- **Регистрация обработчика на `fastapi.HTTPException` вместо
  `StarletteHTTPException`** может не поймать ошибки, поднятые внутри Starlette.
  Для надёжности используйте `StarletteHTTPException`.
- **`detail` должен быть JSON-сериализуемым.** Если положить туда несериализуемый
  объект, получите ошибку при формировании ответа.
- **Кастомный обработчик `RequestValidationError` без поля `detail`** в нужном
  формате может сломать ожидания клиентов и инструменты, которые парсят
  стандартный `422`. Меняйте формат осознанно.
- **`ResponseValidationError` маскирует баг.** Видя `500` из-за валидации
  ответа, не «затыкайте» его обработчиком — чините несоответствие данных
  модели.
- **Заголовки из `HTTPException` теряются**, если вы полностью переопределите
  обработчик и не пробросите `exc.headers` в свой `Response`.
- **Порядок не важен, важна специфичность.** Для одного типа исключения может
  быть только один обработчик; более конкретный класс перехватывается своим
  обработчиком раньше базового.

## Лучшие практики

- Используйте константы `fastapi.status` (`status.HTTP_404_NOT_FOUND`) вместо
  «магических» чисел.
- Для доменных ошибок заводите собственные исключения и централизуйте маппинг
  в HTTP через `@app.exception_handler`.
- Делайте `detail` машинно-читаемым (`{"code": ..., "message": ...}`), если
  клиенты должны программно реагировать на ошибки.
- Логируйте ошибки в кастомных обработчиках, но возвращайте клиенту только
  безопасную информацию (не отдавайте стектрейсы и внутренние детали).
- Переиспользуйте стандартные обработчики Starlette, чтобы не потерять
  совместимость со схемой `422`/`detail`.
- Не превращайте `5xx` в `4xx`: ошибки клиента — `4xx`, сбои сервера — `5xx`.
- Единый формат ошибок по всему API упрощает работу фронтенду и клиентам.

## Шпаргалка

```python
# Поднять HTTP-ошибку
from fastapi import HTTPException, status
raise HTTPException(
    status_code=status.HTTP_404_NOT_FOUND,
    detail="Не найдено",                 # str | dict | list
    headers={"X-Error": "msg"},          # необязательно
)

# Кастомный обработчик своего исключения
@app.exception_handler(MyError)
async def handler(request, exc):
    return JSONResponse(status_code=400, content={"msg": str(exc)})

# Переопределить обработчик HTTPException (на Starlette-классе!)
from starlette.exceptions import HTTPException as StarletteHTTPException
@app.exception_handler(StarletteHTTPException)
async def http_handler(request, exc):
    return await http_exception_handler(request, exc)  # делегируем стандартному

# Переопределить обработчик валидации запроса
from fastapi.exceptions import RequestValidationError
@app.exception_handler(RequestValidationError)
async def val_handler(request, exc):
    return JSONResponse(status_code=422, content={"detail": exc.errors()})

# Переиспользовать стандартные обработчики
from fastapi.exception_handlers import (
    http_exception_handler,
    request_validation_exception_handler,
)
```

| Сущность | Назначение | HTTP-код |
|---|---|---|
| `HTTPException` | вернуть ошибку клиенту вручную | любой `4xx`/`5xx` |
| `RequestValidationError` | вход не прошёл валидацию | `422` |
| `ResponseValidationError` | выход не соответствует `response_model` | `500` |
| `@app.exception_handler` | глобальный маппинг исключения в ответ | — |
```
