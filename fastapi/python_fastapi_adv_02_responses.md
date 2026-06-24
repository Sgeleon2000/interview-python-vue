# FastAPI Advanced — Кастомные ответы и управление ответом — конспект и вопросы

## О чём раздел

Раздел про то, как взять под контроль HTTP-ответ в FastAPI: вернуть произвольный `Response`, выбрать другой JSON-энкодер, отдать HTML/текст/файл/поток, задать редирект, выставить cookie и заголовки, поменять статус-код в зависимости от логики и объявить дополнительные ответы в OpenAPI.

По умолчанию FastAPI берёт то, что вы вернули из операции, прогоняет через `jsonable_encoder` и `response_model`, и заворачивает в `JSONResponse`. Но иногда нужно:

- вернуть НЕ JSON (HTML, текст, файл, бинарь, поток);
- ускорить сериализацию JSON (`ORJSONResponse`);
- отдать большой файл/поток, не держа всё в памяти (`StreamingResponse`, `FileResponse`);
- сделать редирект (`RedirectResponse`);
- управлять статус-кодом, cookie и заголовками динамически;
- задокументировать дополнительные варианты ответа в OpenAPI.

Современный контекст: Pydantic v2, `Annotated[...]`, `async def`.

## Ключевые концепции

- **Возврат `Response` напрямую** — всё, что является `Response` (или его наследником), FastAPI отдаёт как есть, без сериализации и без `response_model`. Это даёт полный контроль, но снимает «магию».
- **Классы ответов** (все из `fastapi.responses`, фактически из Starlette):
  - `JSONResponse` — дефолт;
  - `ORJSONResponse` — быстрый JSON на `orjson` (умеет `datetime`, `UUID`, `numpy`); требует пакет `orjson`;
  - `UJSONResponse` — JSON на `ujson` (менее строгий к стандарту); требует `ujson`;
  - `HTMLResponse` — `Content-Type: text/html`;
  - `PlainTextResponse` — `text/plain`;
  - `StreamingResponse` — потоковая отдача из генератора/итератора (sync или async);
  - `FileResponse` — асинхронная отдача файла с диска (правильные заголовки, поддержка range);
  - `RedirectResponse` — редирект (по умолчанию 307).
- **`response_class`** на уровне операции — меняет класс ответа по умолчанию для этой операции (и корректно отражается в OpenAPI `Content-Type`).
- **`default_response_class`** на уровне приложения — глобально меняет класс ответа по умолчанию.
- **`responses={...}`** — словарь дополнительных ответов для OpenAPI: статус → описание/модель/примеры. Не влияет на рантайм, только на документацию.
- **Управление через параметр `Response`** — внедрив `response: Response` в сигнатуру, можно выставить `response.status_code`, `response.set_cookie(...)`, `response.headers[...]`, при этом продолжая возвращать обычные данные (FastAPI смержит их).
- **`status_code` в зависимости (dependency)** — тот же `Response` можно внедрить и в зависимость, чтобы выставлять заголовки/cookie/статус из общего кода.

## Подробный разбор с примерами кода

### 1. Возврат `Response` напрямую (Return a Response Directly)

```python
from datetime import datetime
from fastapi import FastAPI
from fastapi.encoders import jsonable_encoder
from fastapi.responses import JSONResponse

app = FastAPI()


@app.get("/legacy/")
async def get_legacy_data():
    data = {"timestamp": datetime.utcnow(), "value": 42}
    # При прямом возврате Response сериализация FastAPI пропускается,
    # поэтому datetime готовим сами через jsonable_encoder.
    return JSONResponse(content=jsonable_encoder(data))
```

Главное правило: вернули `Response` — `response_model` и автоматическая сериализация **не применяются**. Нужно сериализовать вручную.

### 2. ORJSONResponse / UJSONResponse — быстрый JSON

```python
from fastapi import FastAPI
from fastapi.responses import ORJSONResponse

# Глобально для всего приложения: дефолтный класс ответа — ORJSONResponse.
app = FastAPI(default_response_class=ORJSONResponse)


# Можно и точечно на операцию, это ещё и правильно отражается в OpenAPI.
@app.get("/items/", response_class=ORJSONResponse)
async def read_items() -> list[dict]:
    return [{"item_id": "Foo"}]
```

`ORJSONResponse` быстрее стандартного, умеет сериализовать `datetime`, `UUID`, dataclasses, numpy — часто можно обойтись без `jsonable_encoder`. Требует установленного `orjson`. `UJSONResponse` (на `ujson`) — альтернатива, но менее строго следует JSON-спецификации (например, по краевым случаям), поэтому `ORJSONResponse` обычно предпочтительнее.

### 3. HTMLResponse

```python
from fastapi import FastAPI
from fastapi.responses import HTMLResponse

app = FastAPI()


# Вариант A: response_class — корректно показывает text/html в OpenAPI.
@app.get("/page/", response_class=HTMLResponse)
async def get_page() -> str:
    return "<html><body><h1>Привет</h1></body></html>"


# Вариант B: вернуть HTMLResponse напрямую (если нужен полный контроль).
@app.get("/page-direct/")
async def get_page_direct():
    return HTMLResponse(content="<h1>Прямой ответ</h1>", status_code=200)
```

Нюанс: если указать `response_class=HTMLResponse`, но вернуть `Response` напрямую — приоритет у того, что вы вернули. `response_class` влияет на дефолт и на OpenAPI.

### 4. PlainTextResponse

```python
from fastapi import FastAPI
from fastapi.responses import PlainTextResponse

app = FastAPI()


@app.get("/ping", response_class=PlainTextResponse)
async def ping() -> str:
    return "pong"
```

### 5. StreamingResponse — потоковая отдача

Отдаёт данные кусками из генератора, не загружая всё в память. Поддерживает и sync, и async генераторы.

```python
import asyncio
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()


# Асинхронный генератор: например, server-sent данные или большой расчёт.
async def number_stream():
    for i in range(10):
        yield f"chunk {i}\n"
        await asyncio.sleep(0.1)  # имитация задержки источника данных


@app.get("/stream/")
async def stream():
    return StreamingResponse(number_stream(), media_type="text/plain")
```

Потоковая отдача большого файла без чтения целиком в память:

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()


@app.get("/big-file/")
async def big_file():
    # file-like объект как итератор: отдаём построчно/поблочно.
    def iterfile():
        with open("large_video.mp4", mode="rb") as f:
            yield from f  # генератор -> память не забивается

    return StreamingResponse(iterfile(), media_type="video/mp4")
```

Для генерации «на лету» (CSV-экспорт большой выборки):

```python
import csv
import io
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()


@app.get("/export.csv")
async def export_csv():
    async def generate():
        # Заголовок CSV
        yield "id,name\n"
        # Имитация выборки строк по одной (например, из БД-курсора).
        for i in range(100_000):
            yield f"{i},name_{i}\n"

    headers = {"Content-Disposition": 'attachment; filename="export.csv"'}
    return StreamingResponse(generate(), media_type="text/csv", headers=headers)
```

### 6. FileResponse — отдача файла с диска

```python
from fastapi import FastAPI
from fastapi.responses import FileResponse

app = FastAPI()


@app.get("/download/")
async def download():
    # FileResponse сам асинхронно читает файл, ставит Content-Type,
    # Content-Length и поддерживает Range-запросы (докачку/перемотку).
    return FileResponse(
        path="files/report.pdf",
        media_type="application/pdf",
        filename="report.pdf",  # имя для скачивания (Content-Disposition)
    )


# Можно объявить как response_class (полезно для OpenAPI).
@app.get("/image/", response_class=FileResponse)
async def image():
    return "files/logo.png"
```

`FileResponse` предпочтительнее ручного `StreamingResponse(open(...))` для статических файлов: он эффективнее (отдаёт через ОС, поддерживает range/докачку, корректно выставляет заголовки).

### 7. RedirectResponse

```python
from fastapi import FastAPI
from fastapi.responses import RedirectResponse

app = FastAPI()


# По умолчанию 307 Temporary Redirect (сохраняет метод и тело).
@app.get("/old-path/")
async def old_path():
    return RedirectResponse(url="/new-path/")


# Можно явно задать статус (301 постоянный, 302, 303 see other).
@app.get("/go/")
async def go():
    return RedirectResponse(url="https://fastapi.tiangolo.com", status_code=302)


# Если объявлять как response_class, передавайте URL через status_code/return.
@app.get("/typed-redirect/", response_class=RedirectResponse)
async def typed_redirect():
    return "/target/"
```

Важно: 307/308 сохраняют HTTP-метод и тело, 301/302 исторически могли менять метод на GET, 303 явно превращает в GET. Для POST→GET после обработки формы используйте 303.

### 8. default_response_class

```python
from fastapi import FastAPI
from fastapi.responses import ORJSONResponse

# Все операции по умолчанию будут отдавать ORJSONResponse.
app = FastAPI(default_response_class=ORJSONResponse)
```

### 9. Дополнительные ответы в OpenAPI (responses={...})

Документирует другие возможные ответы (ошибки, альтернативные модели, не-JSON форматы). На рантайм не влияет.

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    id: str
    value: str


class Message(BaseModel):
    message: str


@app.get(
    "/items/{item_id}",
    response_model=Item,  # основной (200) ответ
    responses={
        404: {"model": Message, "description": "Item не найден"},
        500: {
            "description": "Внутренняя ошибка",
            "content": {
                "application/json": {
                    "example": {"detail": "internal error"}
                }
            },
        },
        200: {
            "description": "Успех. Может также вернуть кастомный заголовок.",
            "content": {
                "application/json": {"example": {"id": "x", "value": "y"}}
            },
        },
    },
)
async def read_item(item_id: str):
    if item_id == "missing":
        return JSONResponse(status_code=404, content={"message": "Не найдено"})
    return {"id": item_id, "value": "ok"}
```

`responses` можно комбинировать: объявить на уровне приложения/роутера (общие ошибки) и дополнить на уровне операции — словари сливаются.

### 10. Cookies в ответе (set_cookie)

Два способа: через внедрённый `Response` или через возвращаемый `Response`.

```python
from fastapi import FastAPI, Response
from fastapi.responses import JSONResponse

app = FastAPI()


# Способ A: внедряем Response, ставим cookie, возвращаем обычные данные.
@app.post("/login-a/")
async def login_a(response: Response):
    response.set_cookie(
        key="session",
        value="abc123",
        httponly=True,   # недоступна из JS
        secure=True,     # только по HTTPS
        samesite="lax",
        max_age=3600,
    )
    return {"ok": True}  # данные смержатся с выставленной cookie


# Способ B: ставим cookie на возвращаемый Response напрямую.
@app.post("/login-b/")
async def login_b():
    resp = JSONResponse(content={"ok": True})
    resp.set_cookie(key="session", value="abc123", httponly=True)
    return resp
```

### 11. Заголовки ответа (Response headers)

```python
from fastapi import FastAPI, Response
from fastapi.responses import JSONResponse

app = FastAPI()


# Способ A: через внедрённый Response.
@app.get("/headers-a/")
async def headers_a(response: Response):
    response.headers["X-Custom-Header"] = "value"
    return {"ok": True}


# Способ B: через возвращаемый Response.
@app.get("/headers-b/")
async def headers_b():
    return JSONResponse(
        content={"ok": True},
        headers={"X-Custom-Header": "value"},
    )
```

### 12. Изменение статус-кода через Response / status_code (в т.ч. в зависимости)

Если объявить `response: Response` в операции, можно выставить статус динамически, продолжая возвращать данные:

```python
from typing import Annotated
from fastapi import FastAPI, Response, status

app = FastAPI()
items: dict[str, str] = {}


@app.put("/items/{item_id}")
async def upsert_item(item_id: str, value: str, response: Response):
    if item_id not in items:
        # Создаём -> 201
        response.status_code = status.HTTP_201_CREATED
    items[item_id] = value
    # Возвращаем обычные данные, FastAPI применит выставленный статус.
    return {"item_id": item_id, "value": value}
```

То же самое — из зависимости (общий код, влияющий на ответ):

```python
from typing import Annotated
from fastapi import Depends, FastAPI, Response, status


# Зависимость может выставлять статус/заголовки на общий Response.
async def maybe_warn(response: Response, x_token: str | None = None):
    if x_token is None:
        response.headers["X-Token-Warning"] = "missing"
        response.status_code = status.HTTP_202_ACCEPTED
    return x_token


@app.get("/with-dep/")
async def with_dep(token: Annotated[str | None, Depends(maybe_warn)] = None):
    return {"token": token}
```

`Response` здесь — один и тот же объект, что и для операции, поэтому изменения из зависимости видны в финальном ответе.

## Полный рабочий пример

```python
import asyncio
from datetime import datetime
from typing import Annotated

from fastapi import Depends, FastAPI, Response, status
from fastapi.encoders import jsonable_encoder
from fastapi.responses import (
    FileResponse,
    HTMLResponse,
    JSONResponse,
    ORJSONResponse,
    PlainTextResponse,
    RedirectResponse,
    StreamingResponse,
)
from pydantic import BaseModel

# Глобально используем быстрый ORJSONResponse.
app = FastAPI(default_response_class=ORJSONResponse, title="Custom Responses")


class Item(BaseModel):
    id: str
    value: str


class Message(BaseModel):
    message: str


db: dict[str, str] = {}


# --- Зависимость, выставляющая заголовок/статус на общий Response ---
async def audit(response: Response):
    response.headers["X-Audit"] = "logged"
    return None


# --- Обычный JSON с управлением статусом и cookie ---
@app.put("/items/{item_id}")
async def upsert_item(
    item_id: str,
    value: str,
    response: Response,
    _: Annotated[None, Depends(audit)],
):
    if item_id not in db:
        response.status_code = status.HTTP_201_CREATED
    db[item_id] = value
    response.set_cookie("last_item", item_id, httponly=True, samesite="lax")
    return {"id": item_id, "value": value}


# --- Эндпоинт с документированными доп. ответами ---
@app.get(
    "/items/{item_id}",
    response_model=Item,
    responses={404: {"model": Message, "description": "Не найдено"}},
)
async def read_item(item_id: str):
    if item_id not in db:
        return JSONResponse(
            status_code=404,
            content=jsonable_encoder(Message(message="Не найдено")),
        )
    return {"id": item_id, "value": db[item_id]}


# --- HTML ---
@app.get("/page", response_class=HTMLResponse)
async def page() -> str:
    return "<html><body><h1>Главная</h1></body></html>"


# --- Plain text ---
@app.get("/ping", response_class=PlainTextResponse)
async def ping() -> str:
    return "pong"


# --- Streaming (генерация на лету) ---
@app.get("/stream")
async def stream():
    async def gen():
        for i in range(5):
            yield f"chunk {i} @ {datetime.utcnow().isoformat()}\n"
            await asyncio.sleep(0.05)

    return StreamingResponse(gen(), media_type="text/plain")


# --- Streaming файла с диска ---
@app.get("/video")
async def video():
    def iterfile():
        with open("media/sample.mp4", "rb") as f:
            yield from f

    return StreamingResponse(iterfile(), media_type="video/mp4")


# --- FileResponse ---
@app.get("/download", response_class=FileResponse)
async def download():
    return FileResponse(
        "files/report.pdf",
        media_type="application/pdf",
        filename="report.pdf",
    )


# --- Redirect ---
@app.get("/old")
async def old():
    return RedirectResponse(url="/page", status_code=307)
```

## Частые вопросы на собеседовании (Q/A)

**1. Что происходит, когда вы возвращаете `Response` напрямую вместо данных?**
FastAPI отдаёт этот `Response` клиенту «как есть»: пропускает `response_model`, автоматическую сериализацию и фильтрацию полей. Контент нужно сериализовать самостоятельно (например, `jsonable_encoder`), кроме случаев, когда сам класс ответа умеет это (как `ORJSONResponse`).

**2. Чем `ORJSONResponse` отличается от стандартного `JSONResponse`?**
Использует библиотеку `orjson` — заметно быстрее и умеет сериализовать `datetime`, `UUID`, dataclasses, numpy «из коробки». Требует установленного `orjson`. Часто позволяет обойтись без `jsonable_encoder`.

**3. Когда выбрать `UJSONResponse`, а когда `ORJSONResponse`?**
`UJSONResponse` (на `ujson`) тоже быстрый, но менее строго следует спецификации JSON в краевых случаях. В большинстве проектов предпочитают `ORJSONResponse` как более корректный и обычно более быстрый.

**4. В чём разница между `response_class` (на операции) и `default_response_class` (на приложении)?**
`response_class` меняет класс ответа по умолчанию для конкретной операции и корректно отражает `Content-Type` в OpenAPI. `default_response_class` делает то же глобально для всех операций.

**5. Когда использовать `StreamingResponse`, а когда `FileResponse`?**
`StreamingResponse` — для данных, генерируемых на лету или читаемых кусками (большой CSV, прокси-поток, SSE). `FileResponse` — для готовых файлов на диске: он эффективнее, ставит правильные заголовки и поддерживает Range-запросы (докачка/перемотка).

**6. Почему `StreamingResponse` экономит память?**
Потому что отдаёт данные по кускам из генератора/итератора, не материализуя весь объём в памяти. Сервер шлёт чанки по мере их готовности (chunked transfer / поток).

**7. Какой статус-код у `RedirectResponse` по умолчанию и почему это важно?**
По умолчанию 307 (Temporary Redirect), который сохраняет HTTP-метод и тело запроса. 301/302 исторически могут менять метод на GET, 303 явно превращает в GET. Для POST→GET после формы используют 303.

**8. Как поменять статус-код, продолжая возвращать обычные данные (не `Response`)?**
Внедрить `response: Response` в сигнатуру и выставить `response.status_code = ...`. FastAPI применит этот статус к финальному ответу, при этом данные всё ещё пройдут обычную сериализацию.

**9. Как выставить cookie? Какие флаги безопасности важны?**
Через `response.set_cookie(...)` (на внедрённом или возвращаемом `Response`). Важные флаги: `httponly=True` (недоступна из JS, защита от XSS-кражи), `secure=True` (только HTTPS), `samesite="lax"/"strict"` (защита от CSRF).

**10. Зачем нужен параметр `responses={...}` и влияет ли он на рантайм?**
Он описывает дополнительные возможные ответы (коды ошибок, альтернативные модели, форматы) в OpenAPI. На рантайм НЕ влияет — это только документация. Можно объявлять на уровне приложения/роутера и дополнять на операции (словари сливаются).

**11. Можно ли менять заголовки/статус из зависимости?**
Да. Если внедрить `response: Response` в зависимость, это тот же объект ответа, что и у операции. Изменения заголовков/cookie/статуса из зависимости попадут в финальный ответ. Удобно для общей логики (аудит, предупреждения).

**12. Откуда импортируются классы ответов и где они физически реализованы?**
Импортируются из `fastapi.responses`, но фактически это классы Starlette (`starlette.responses`). FastAPI реэкспортирует их для удобства.

**13. Если указать `response_class=HTMLResponse`, но вернуть `JSONResponse` напрямую — что победит?**
Победит то, что вы вернули напрямую (`JSONResponse`). `response_class` задаёт лишь класс по умолчанию и влияет на OpenAPI; явный возврат `Response` имеет приоритет.

## Подводные камни (gotchas)

- **Прямой `Response` отключает `response_model`.** Никакой фильтрации полей и автосериализации — готовьте контент сами (`jsonable_encoder` для `JSONResponse`).
- **`StreamingResponse(open(...))` без генератора держит файл/память.** Для статических файлов используйте `FileResponse` — он эффективнее и умеет Range.
- **Генератор `StreamingResponse` не должен бросать исключения после старта отдачи.** Заголовки уже отправлены, корректный статус ошибки выставить нельзя — клиент получит оборванный поток.
- **Cookie без `secure`/`httponly`/`samesite` небезопасны.** Для сессий всегда задавайте флаги; `secure` требует HTTPS (на localhost может мешать тестированию).
- **`responses={...}` не валидирует ответ.** Это только документация: реальный возврат может не совпасть с объявленной моделью — FastAPI не проверит.
- **307/308 vs 303 при редиректе.** Неправильный код после POST приводит к повторной отправке тела. Для PRG-паттерна (Post/Redirect/Get) нужен 303.
- **`ORJSONResponse`/`UJSONResponse` требуют установленных пакетов** (`orjson`, `ujson`). Без них — ImportError.
- **Менять `response.status_code` имеет смысл только при обычном возврате данных.** Если вы возвращаете `Response` напрямую, статус берётся из самого `Response`, а не из внедрённого.

## Лучшие практики

- Для не-JSON ответов указывайте `response_class` — это правильно документирует `Content-Type` в OpenAPI.
- Большие файлы отдавайте `FileResponse`, генерируемые потоки — `StreamingResponse` с генератором.
- Для производительного JSON ставьте `default_response_class=ORJSONResponse` глобально.
- Документируйте ошибки и альтернативные ответы через `responses={...}`, в т.ч. общие — на уровне роутера.
- Cookie сессий — всегда `httponly`, `secure` (на проде), `samesite`.
- Динамический статус/заголовки делайте через внедрённый `Response`, чтобы не терять автосериализацию данных.
- Общие заголовки/аудит выносите в зависимость, выставляющую их на общий `Response`.
- Для редиректов после POST используйте 303 (PRG-паттерн), для временных — 307.

## Шпаргалка

```python
from fastapi.responses import (
    JSONResponse, ORJSONResponse, UJSONResponse, HTMLResponse,
    PlainTextResponse, StreamingResponse, FileResponse, RedirectResponse,
)

# Прямой возврат + ручная сериализация
return JSONResponse(content=jsonable_encoder(data), status_code=201)

# Быстрый JSON
app = FastAPI(default_response_class=ORJSONResponse)
@app.get("/x", response_class=ORJSONResponse)

# HTML / текст
@app.get("/p", response_class=HTMLResponse)
@app.get("/t", response_class=PlainTextResponse)

# Поток / файл
StreamingResponse(gen(), media_type="text/csv")
FileResponse("f.pdf", filename="f.pdf", media_type="application/pdf")

# Редирект (307 по умолчанию)
RedirectResponse(url="/new", status_code=303)

# Доп. ответы в OpenAPI
@app.get("/x", responses={404: {"model": Message}})

# Cookie / заголовки / статус через внедрённый Response
async def h(response: Response):
    response.status_code = status.HTTP_201_CREATED
    response.headers["X-H"] = "v"
    response.set_cookie("s", "abc", httponly=True, secure=True, samesite="lax")
    return {"ok": True}
```
