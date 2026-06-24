# FastAPI — First Steps (первые шаги) — конспект и вопросы

## О чём раздел

Этот раздел знакомит с самыми базовыми вещами FastAPI: что это за фреймворк, как он устроен под капотом (ASGI), как поднять приложение через uvicorn, как объявить эндпоинты (path operations) с помощью декораторов, в чём разница между `async def` и `def`, как FastAPI автоматически генерирует документацию (Swagger UI / ReDoc) на основе стандарта OpenAPI и как он использует подсказки типов (type hints) Python для валидации, сериализации и документации.

FastAPI — это современный, высокопроизводительный веб-фреймворк для построения API на Python 3.8+. Его ключевая идея: вы пишете обычные аннотированные типами функции, а фреймворк бесплатно даёт вам валидацию данных, сериализацию, автодокументацию и автодополнение в IDE.

## Ключевые концепции

- **ASGI (Asynchronous Server Gateway Interface)** — современный стандарт интерфейса между веб-сервером и Python-приложением, асинхронный наследник WSGI. Поддерживает `async/await`, WebSocket, long-polling, HTTP/2. FastAPI — это ASGI-фреймворк, построенный поверх **Starlette** (веб-часть) и **Pydantic** (валидация данных).
- **uvicorn** — ASGI-сервер (на базе uvloop и httptools), который запускает ваше приложение. Альтернативы: Hypercorn, Daphne. В продакшене часто uvicorn запускают под управлением Gunicorn (`gunicorn -k uvicorn.workers.UvicornWorker`) для нескольких воркеров.
- **Path operation** — комбинация HTTP-метода (operation) и пути (path). Например, `GET /items`. Объявляется декоратором (`@app.get`, `@app.post`, ...) над функцией — **path operation function**.
- **Декораторы методов** — `@app.get`, `@app.post`, `@app.put`, `@app.delete`, `@app.patch`, `@app.options`, `@app.head`, `@app.trace`.
- **Type hints (подсказки типов)** — основа FastAPI. По аннотациям параметров фреймворк понимает, откуда брать данные (path/query/body), как их валидировать и как сериализовать ответ.
- **OpenAPI** — стандарт описания REST API (раньше назывался Swagger). FastAPI автоматически генерирует `openapi.json` — схему всего вашего API.
- **JSON Schema** — стандарт описания структуры JSON. Pydantic-модели конвертируются в JSON Schema и встраиваются в OpenAPI.
- **Swagger UI** (`/docs`) и **ReDoc** (`/redoc`) — две интерактивные веб-документации, генерируемые из `openapi.json` автоматически.
- **`async def` vs `def`** — выбор зависит от того, используете ли вы внутри функции асинхронные библиотеки с `await` или нет.

## Подробный разбор с примерами кода

### 1. Минимальное приложение

```python
from fastapi import FastAPI

# Создаём экземпляр приложения. Это главный объект, с которым работает uvicorn.
# Имя переменной (app) важно: команда uvicorn ссылается на него как "модуль:переменная".
app = FastAPI()


# Декоратор @app.get("/") говорит FastAPI:
# функцию ниже нужно вызвать, когда придёт GET-запрос на путь "/".
@app.get("/")
async def root():
    # Возвращаемый dict FastAPI автоматически сериализует в JSON.
    return {"message": "Hello World"}
```

Запуск:

```bash
# main — имя файла main.py без расширения; app — переменная FastAPI()
uvicorn main:app --reload
```

- `--reload` — авто-перезапуск сервера при изменении файлов (только для разработки, не для продакшена; держит процесс watcher и тратит ресурсы).
- По умолчанию сервер слушает `http://127.0.0.1:8000`.
- Документация: `http://127.0.0.1:8000/docs` (Swagger UI) и `http://127.0.0.1:8000/redoc` (ReDoc).
- Схема: `http://127.0.0.1:8000/openapi.json`.

Альтернативно через CLI самого FastAPI (появился в новых версиях):

```bash
fastapi dev main.py      # режим разработки (аналог --reload)
fastapi run main.py      # продакшен-режим
```

### 2. Что происходит по шагам

1. Создаём `app = FastAPI()`.
2. Объявляем path operation декоратором.
3. Пишем функцию-обработчик.
4. Запускаем dev-сервер (uvicorn или `fastapi dev`).

Декоратор `@app.get("/")` регистрирует маршрут. Сама функция (`root`) — это «path operation function»; FastAPI вызовет её для каждого подходящего запроса.

### 3. Порядок HTTP-методов и семантика

```python
@app.get("/items")      # получить (чтение, идемпотентно, без тела)
@app.post("/items")     # создать (есть тело, не идемпотентно)
@app.put("/items/{id}") # заменить целиком (идемпотентно)
@app.patch("/items/{id}")  # частично обновить
@app.delete("/items/{id}") # удалить
```

FastAPI не навязывает семантику, но рекомендует следовать соглашениям REST — это влияет на то, как генерируется документация и как клиенты воспринимают API.

### 4. `async def` vs `def` — когда что использовать

Это один из самых частых вопросов на собеседовании.

```python
# Вариант A: async def — когда внутри есть await к асинхронным библиотекам
@app.get("/async")
async def read_async():
    # await позволяет отдать управление event loop, пока ждём I/O
    result = await some_async_db.fetch(...)
    return result


# Вариант B: обычный def — когда библиотека синхронная (нет await)
@app.get("/sync")
def read_sync():
    # синхронный, возможно блокирующий вызов
    result = some_sync_db.query(...)
    return result
```

Правила:

- Если внутри функции вы вызываете библиотеку с `await` (async DB-драйвер, httpx async, aioredis) — пишите `async def`.
- Если библиотека **синхронная** и блокирующая (обычный psycopg2, requests, тяжёлый CPU-расчёт) — пишите **обычный `def`**. FastAPI сам выполнит такую функцию в отдельном **threadpool** (через `run_in_threadpool` Starlette), чтобы не блокировать event loop.
- **Опасность**: если внутри `async def` сделать блокирующий синхронный вызов (например `time.sleep()` или `requests.get()`), вы **заблокируете весь event loop** и убьёте конкурентность сервера. В обычном `def` тот же блокирующий вызов безопасен, потому что он в отдельном потоке.
- Не знаете — можно безопасно писать обычный `def`. FastAPI разрулит.

### 5. Как FastAPI использует подсказки типов

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/items/{item_id}")
async def read_item(item_id: int):
    # Аннотация item_id: int даёт:
    # 1) конвертацию: "3" из URL -> int 3
    # 2) валидацию: "abc" -> 422 с понятным сообщением об ошибке
    # 3) документацию: в Swagger параметр помечен как integer
    # 4) автодополнение в IDE: item_id известен как int
    return {"item_id": item_id}
```

Type hints — это не просто подсказки для разработчика. FastAPI читает их в рантайме и строит из них валидацию (через Pydantic) и схему OpenAPI.

### 6. Автодокументация: OpenAPI -> Swagger UI / ReDoc

- FastAPI генерирует `openapi.json` — машиночитаемое описание всех путей, параметров, тел запросов и ответов.
- **Swagger UI** (`/docs`) — интерактивная документация: можно прямо в браузере отправлять запросы («Try it out»).
- **ReDoc** (`/redoc`) — более «читательская», документально-ориентированная подача той же схемы.
- Можно отключить или переименовать: `FastAPI(docs_url=None, redoc_url=None)` или задать `openapi_url=None`, чтобы полностью убрать схему.

### 7. Метаданные приложения

```python
app = FastAPI(
    title="My Awesome API",       # заголовок в документации
    description="API для собеседования",
    version="1.0.0",              # версия вашего API (не FastAPI)
    docs_url="/docs",             # путь Swagger UI (None — отключить)
    redoc_url="/redoc",           # путь ReDoc
    openapi_url="/openapi.json",  # путь к схеме (None — отключить целиком)
)
```

## Полный рабочий пример

```python
# main.py
from enum import Enum

from fastapi import FastAPI

# Метаданные попадают в автодокументацию (/docs, /redoc, /openapi.json).
app = FastAPI(
    title="First Steps API",
    description="Учебный API для подготовки к собеседованию по FastAPI",
    version="1.0.0",
)


class Tag(str, Enum):
    """Перечисление допустимых значений тега (попадёт в OpenAPI как enum)."""
    new = "new"
    sale = "sale"


@app.get("/")
async def root():
    """Корневой эндпоинт. Возвращает приветствие.

    Текст докстринги попадает в описание операции в Swagger UI.
    """
    return {"message": "Hello FastAPI"}


@app.get("/items/{item_id}")
async def read_item(item_id: int):
    """Чтение элемента по числовому id.

    item_id берётся из пути и валидируется как int.
    """
    return {"item_id": item_id}


@app.get("/tags/{tag}")
async def read_by_tag(tag: Tag):
    """Демонстрация Enum в path-параметре: допустимы только new / sale."""
    return {"tag": tag, "value": tag.value}


@app.post("/items")
async def create_item(name: str):
    """Создание элемента. POST с не-идемпотентной семантикой."""
    return {"created": name}


# Опционально: запуск из самого файла (python main.py).
# В реальной разработке обычно используют команду uvicorn / fastapi dev.
if __name__ == "__main__":
    import uvicorn

    uvicorn.run("main:app", host="127.0.0.1", port=8000, reload=True)
```

Запуск любым из способов:

```bash
uvicorn main:app --reload
# или
fastapi dev main.py
# или
python main.py
```

Проверка:

```bash
curl http://127.0.0.1:8000/                 # {"message":"Hello FastAPI"}
curl http://127.0.0.1:8000/items/42         # {"item_id":42}
curl http://127.0.0.1:8000/items/abc        # 422 Unprocessable Entity
```

## Частые вопросы на собеседовании (Q/A)

**1. Что такое FastAPI и на чём он построен?**
Современный async-веб-фреймворк для API на Python. Построен на Starlette (веб/ASGI слой: маршрутизация, middleware, WebSocket) и Pydantic (валидация и сериализация данных через type hints). Даёт автодокументацию OpenAPI.

**2. Что такое ASGI и чем отличается от WSGI?**
WSGI — синхронный интерфейс «сервер ↔ Python-приложение» (Flask, Django до 3.0). ASGI — асинхронный преемник: поддерживает `async/await`, WebSocket, HTTP/2, long-polling. FastAPI — ASGI-фреймворк, поэтому может обрабатывать множество соединений конкурентно в одном процессе.

**3. Зачем нужен uvicorn? Можно ли без него?**
uvicorn — ASGI-сервер, который непосредственно принимает HTTP-запросы и вызывает ваше приложение. Без ASGI-сервера приложение не запустить (сам FastAPI не слушает сокеты). Альтернативы — Hypercorn, Daphne. В проде часто Gunicorn с uvicorn-воркерами.

**4. В чём разница между `async def` и `def` для обработчика?**
`async def` — если внутри есть `await` к асинхронным библиотекам; функция выполняется в event loop. Обычный `def` — для синхронного/блокирующего кода; FastAPI выполняет его в threadpool, чтобы не блокировать loop. Главная ошибка — блокирующий вызов внутри `async def`: он замораживает весь event loop.

**5. Что произойдёт, если внутри `async def` вызвать `time.sleep(5)`?**
Заблокируется весь event loop: ни один другой запрос не будет обрабатываться эти 5 секунд. Нужно либо `await asyncio.sleep(5)`, либо вынести в обычный `def`/threadpool.

**6. Как FastAPI использует подсказки типов?**
Читает аннотации в рантайме и строит из них: конвертацию входных данных, валидацию (через Pydantic), сериализацию ответа и схему OpenAPI. Плюс — автодополнение в IDE и проверка mypy.

**7. Что такое path operation?**
Сочетание HTTP-метода (operation) и пути (path), например `GET /items/{id}`. Функция под декоратором — path operation function.

**8. Откуда берётся документация `/docs` и `/redoc`?**
Из автоматически сгенерированного `openapi.json`. Swagger UI и ReDoc — это два разных фронтенда, рендерящие одну и ту же OpenAPI-схему. Можно отключить через `docs_url=None`, `redoc_url=None`, `openapi_url=None`.

**9. Что такое OpenAPI и JSON Schema, как связаны?**
OpenAPI — стандарт описания REST API (пути, методы, параметры, ответы). JSON Schema — стандарт описания структуры JSON-данных; используется внутри OpenAPI для описания тел запросов/ответов (Pydantic-модели конвертируются в JSON Schema).

**10. Зачем флаг `--reload` и почему его нельзя в проде?**
Авто-перезапуск при изменении исходников — удобно в разработке. В проде он лишний (тратит ресурсы на watcher, перезапускает воркеры) и не нужен; там используют несколько воркеров без reload.

**11. Как запустить несколько воркеров?**
`uvicorn main:app --workers 4` или Gunicorn: `gunicorn -k uvicorn.workers.UvicornWorker -w 4 main:app`. Несколько процессов = параллелизм по CPU-ядрам (в дополнение к конкурентности event loop внутри каждого).

**12. Можно ли вернуть из обработчика обычный dict?**
Да. FastAPI автоматически сериализует dict/list/Pydantic-модель в JSON через `jsonable_encoder` и оборачивает в `JSONResponse`. Можно вернуть и явный `Response`.

**13. Что значит `main:app` в команде uvicorn?**
`main` — модуль (файл `main.py`), `app` — имя переменной с экземпляром `FastAPI()` внутри модуля. uvicorn импортирует модуль и берёт этот объект как ASGI-приложение.

**14. Зачем нужен Starlette, если есть FastAPI?**
Starlette даёт низкоуровневый ASGI-функционал: маршрутизацию, middleware, фоновые задачи, WebSocket, тестовый клиент. FastAPI добавляет поверх него валидацию (Pydantic), dependency injection и автодокументацию. FastAPI — подкласс Starlette.

## Подводные камни (gotchas)

- **Блокирующий код в `async def`** — самая частая и опасная ошибка: `time.sleep`, `requests`, синхронный psycopg2, тяжёлый CPU — всё это замораживает event loop. Решение: `async`-аналоги, `await asyncio.sleep`, либо вынос в обычный `def` (threadpool) или в `run_in_executor`.
- **`--reload` в продакшене** — никогда. Только dev.
- **Неправильное имя в `uvicorn main:app`** — частая опечатка; должно совпадать с именем файла и переменной.
- **Отдача `Response`/`StreamingResponse` напрямую** — тогда автоматическая сериализация и `response_model` не применяются.
- **Думать, что type hints — только подсказки** — в FastAPI они влияют на рантайм-поведение (валидация, схема). Неверная аннотация = неверная валидация.
- **Конкурентность ≠ параллелизм**: один воркер с event loop конкурентно обрабатывает I/O, но CPU-bound задачи всё равно нужно масштабировать воркерами/процессами.

## Лучшие практики

- В разработке используйте `fastapi dev main.py` или `uvicorn main:app --reload`; в проде — без reload, с несколькими воркерами за reverse-proxy (nginx).
- Пишите `async def` для async-библиотек, обычный `def` — для синхронных. Не смешивайте блокирующий код в async-функциях.
- Всегда аннотируйте типы параметров и возвращаемых значений — это даёт валидацию, документацию и автодополнение бесплатно.
- Задавайте метаданные приложения (`title`, `version`, `description`) — они улучшают автодокументацию.
- Используйте докстринги функций — они попадают в Swagger UI как описание операций.
- Следуйте REST-семантике методов (GET — чтение, POST — создание и т. д.).
- Для CPU-bound задач используйте отдельные воркеры/процессы (или вынос в очередь, Celery/RQ), не блокируйте event loop.

## Шпаргалка

```python
from fastapi import FastAPI

app = FastAPI(title="API", version="1.0.0")

@app.get("/")            # GET
async def root(): ...

@app.post("/x")          # POST
@app.put("/x/{id}")      # PUT
@app.patch("/x/{id}")    # PATCH
@app.delete("/x/{id}")   # DELETE
```

| Что | Значение |
|---|---|
| ASGI | Async-интерфейс сервер↔приложение (преемник WSGI) |
| uvicorn | ASGI-сервер запуска приложения |
| Starlette | Веб-слой под FastAPI |
| Pydantic | Валидация/сериализация по type hints |
| `/docs` | Swagger UI (интерактивная документация) |
| `/redoc` | ReDoc (документальная подача) |
| `/openapi.json` | Машинная схема OpenAPI |
| `async def` | Есть `await` к async-библиотекам |
| `def` | Синхронный код → threadpool |

```bash
uvicorn main:app --reload          # dev
fastapi dev main.py                # dev (CLI FastAPI)
fastapi run main.py                # prod
uvicorn main:app --workers 4       # несколько воркеров
gunicorn -k uvicorn.workers.UvicornWorker -w 4 main:app  # prod через gunicorn
```

Правило `async`/`def`: блокирующий вызов — только в обычном `def`; в `async def` — только `await`.
