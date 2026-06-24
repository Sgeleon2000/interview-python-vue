# FastAPI — Метаданные и статика — конспект и вопросы

## О чём раздел

Раздел про две связанные темы:

1. **Метаданные приложения и документации** — как настроить заголовок, описание,
   версию, контакты, лицензию, описания тегов и URL автодокументации (Swagger UI / ReDoc /
   OpenAPI JSON), а также как их отключать.
2. **Раздача статики** — как отдавать статические файлы (CSS, JS, картинки) через
   `StaticFiles` и `app.mount`, и как отдавать отдельные файлы через `FileResponse`.

Это «витрина» вашего API: грамотные метаданные делают авто-документацию читаемой и
профессиональной, а `StaticFiles` позволяет обслуживать ассеты без отдельного веб-сервера
(хотя в проде статику часто отдают через Nginx/CDN).

## Ключевые концепции

- **Метаданные приложения** задаются в конструкторе `FastAPI(...)`: `title`, `description`,
  `summary`, `version`, `terms_of_service`, `contact`, `license_info`.
- **Метаданные тегов** — список `openapi_tags`: описывает каждый тег (имя, описание,
  внешняя документация), задаёт порядок и группировку эндпоинтов в Swagger UI.
- **URL документации** настраиваются параметрами `docs_url`, `redoc_url`, `openapi_url`.
  Любой можно задать кастомным или **отключить**, передав `None`.
- **`StaticFiles`** (`fastapi.staticfiles.StaticFiles`, реэкспорт из Starlette) — ASGI-приложение
  для отдачи файлов из директории.
- **`app.mount(path, app, name=...)`** — монтирует под-приложение (например, `StaticFiles`)
  на под-путь. Смонтированные пути «выпадают» из основной OpenAPI-схемы.
- **`FileResponse`** — ответ, отдающий конкретный файл с диска (для скачивания/одиночных файлов).

## Подробный разбор с примерами кода

### Метаданные приложения

```python
from fastapi import FastAPI

# description поддерживает Markdown — он отрендерится в Swagger UI / ReDoc.
description = """
## ShopAPI

API интернет-магазина. Позволяет:

* Управлять **товарами**.
* Управлять **заказами**.
"""

app = FastAPI(
    title="ShopAPI",                      # Заголовок (виден в доке и схеме)
    description=description,               # Описание (Markdown)
    summary="Краткое API магазина.",      # Короткое summary (OpenAPI 3.1)
    version="2.1.0",                      # Версия вашего API (не OpenAPI!)
    terms_of_service="https://example.com/terms/",
    contact={
        "name": "Команда поддержки",
        "url": "https://example.com/support",
        "email": "support@example.com",
    },
    license_info={
        "name": "Apache 2.0",
        "url": "https://www.apache.org/licenses/LICENSE-2.0.html",
        # Альтернатива url в OpenAPI 3.1: "identifier": "Apache-2.0" (SPDX).
    },
)
```

Замечания:
- `version` — это версия **вашего приложения/API**, а не версия OpenAPI.
- `summary` доступен начиная с OpenAPI 3.1 (FastAPI новых версий использует 3.1).
- `description` рендерится как Markdown.

### Метаданные тегов (`openapi_tags`)

Теги группируют эндпоинты в документации. Через `openapi_tags` можно задать описание
каждого тега и **порядок** их отображения.

```python
tags_metadata = [
    {
        "name": "products",
        "description": "Операции с **товарами**: создание, чтение, обновление.",
    },
    {
        "name": "orders",
        "description": "Управление заказами пользователей.",
        "externalDocs": {
            "description": "Подробная документация по заказам",
            "url": "https://example.com/docs/orders",
        },
    },
]

app = FastAPI(openapi_tags=tags_metadata)


@app.get("/products", tags=["products"])
async def list_products():
    return [{"id": 1, "name": "Книга"}]


@app.post("/orders", tags=["orders"])
async def create_order():
    return {"order_id": 100}
```

Порядок тегов в `openapi_tags` определяет порядок секций в Swagger UI. Имя тега в
`tags=[...]` эндпоинта должно совпадать с `"name"` в метаданных.

### Настройка URL документации

```python
app = FastAPI(
    docs_url="/documentation",   # Swagger UI (по умолчанию /docs)
    redoc_url="/redoc-ui",       # ReDoc (по умолчанию /redoc)
    openapi_url="/openapi-v2.json",  # JSON-схема (по умолчанию /openapi.json)
)
```

Отключение документации (например, на проде):

```python
# Полностью отключаем Swagger UI и ReDoc.
app = FastAPI(docs_url=None, redoc_url=None)

# Отключить всё, включая саму схему OpenAPI:
app = FastAPI(openapi_url=None)   # тогда /docs и /redoc тоже перестанут работать
```

Если `openapi_url=None`, схема не генерируется вовсе, и `/docs`/`/redoc` ломаются
(им неоткуда брать данные). Часто документацию отключают в продакшене по соображениям
безопасности, оставляя её только в dev.

Условное включение по окружению:

```python
import os

is_prod = os.getenv("ENV") == "production"

app = FastAPI(
    docs_url=None if is_prod else "/docs",
    redoc_url=None if is_prod else "/redoc",
    openapi_url=None if is_prod else "/openapi.json",
)
```

### Раздача статики через `StaticFiles` и `app.mount`

```python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

app = FastAPI()

# Все файлы из папки ./static будут доступны по префиксу /static.
# Например, ./static/css/main.css -> GET /static/css/main.css
app.mount("/static", StaticFiles(directory="static"), name="static")
```

- `directory` — путь к папке с файлами (относительно cwd; лучше строить абсолютный путь).
- `name` — имя для генерации URL через `request.url_for("static", path="...")`.
- Маршруты под `/static` **не попадают** в OpenAPI-схему (это под-приложение).

Полезные опции `StaticFiles`:

```python
# html=True: для запроса каталога отдаёт index.html (режим SPA/сайта).
app.mount("/", StaticFiles(directory="frontend", html=True), name="frontend")
```

Важно: монтирование на `/` (корень) «перехватит» все необъявленные пути — поэтому
такой mount обычно делают **последним**, после всех API-роутов, иначе он может затенить
эндпоинты. Лучше держать API под префиксом (например, `/api`) и монтировать SPA на `/`.

Построение абсолютного пути к директории (надёжнее относительного):

```python
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent
app.mount("/static", StaticFiles(directory=BASE_DIR / "static"), name="static")
```

### Отдача отдельного файла через `FileResponse`

`FileResponse` удобен для скачивания конкретного файла или динамической отдачи:

```python
from fastapi import FastAPI
from fastapi.responses import FileResponse

app = FastAPI()


@app.get("/download/report")
async def download_report():
    # filename задаёт имя файла при скачивании (Content-Disposition).
    return FileResponse(
        path="reports/report.pdf",
        media_type="application/pdf",
        filename="отчёт.pdf",
    )


@app.get("/favicon.ico", include_in_schema=False)
async def favicon():
    # Отдаём один статический файл, скрывая эндпоинт из доки.
    return FileResponse("static/favicon.ico")
```

`FileResponse` (vs `StaticFiles`): `StaticFiles` — целая директория как под-приложение;
`FileResponse` — один файл из обычного эндпоинта (можно навесить авторизацию, логику выбора файла).

## Полный рабочий пример

```python
from pathlib import Path

from fastapi import FastAPI
from fastapi.responses import FileResponse
from fastapi.staticfiles import StaticFiles

BASE_DIR = Path(__file__).resolve().parent

# --- Метаданные тегов ---
tags_metadata = [
    {"name": "products", "description": "Операции с **товарами**."},
    {
        "name": "files",
        "description": "Скачивание файлов и отдача статики.",
        "externalDocs": {
            "description": "Подробнее",
            "url": "https://example.com/docs/files",
        },
    },
]

# --- Метаданные приложения + кастомные URL доки ---
app = FastAPI(
    title="ShopAPI",
    description="Демо-API с метаданными и статикой.",
    summary="Магазин: товары и файлы.",
    version="1.0.0",
    terms_of_service="https://example.com/terms/",
    contact={"name": "Support", "email": "support@example.com"},
    license_info={"name": "MIT", "url": "https://opensource.org/licenses/MIT"},
    openapi_tags=tags_metadata,
    docs_url="/docs",
    redoc_url="/redoc",
    openapi_url="/openapi.json",
)

# --- Статика: ./static -> /static ---
app.mount("/static", StaticFiles(directory=BASE_DIR / "static"), name="static")


@app.get("/products", tags=["products"])
async def list_products():
    return [{"id": 1, "name": "Книга"}]


@app.get("/files/report", tags=["files"])
async def download_report():
    return FileResponse(
        BASE_DIR / "reports" / "report.pdf",
        media_type="application/pdf",
        filename="report.pdf",
    )


@app.get("/favicon.ico", include_in_schema=False)
async def favicon():
    return FileResponse(BASE_DIR / "static" / "favicon.ico")


if __name__ == "__main__":
    import uvicorn

    uvicorn.run("main:app", host="127.0.0.1", port=8000, reload=True)
```

## Частые вопросы на собеседовании (Q/A)

**1. Где задаются метаданные приложения?**
В конструкторе `FastAPI(...)`: `title`, `description`, `summary`, `version`,
`terms_of_service`, `contact`, `license_info`.

**2. Что означает параметр `version`?**
Это версия **вашего API/приложения**, отображаемая в документации и схеме, а не версия
спецификации OpenAPI.

**3. Поддерживает ли `description` форматирование?**
Да, Markdown. Он рендерится в Swagger UI и ReDoc.

**4. Зачем нужны `openapi_tags`?**
Чтобы дать тегам описания (Markdown) и внешние ссылки, а также задать **порядок** и
группировку секций эндпоинтов в документации.

**5. Как изменить адрес Swagger UI / ReDoc / схемы?**
Параметрами `docs_url`, `redoc_url`, `openapi_url`. Например, `docs_url="/documentation"`.

**6. Как отключить документацию в продакшене?**
`docs_url=None, redoc_url=None`. Чтобы убрать и саму схему — `openapi_url=None`
(при этом `/docs` и `/redoc` тоже перестанут работать, т.к. им нечего показывать).

**7. Чем `StaticFiles` отличается от `FileResponse`?**
`StaticFiles` монтируется на путь и отдаёт **целую директорию** как под-приложение.
`FileResponse` отдаёт **один конкретный файл** из обычного эндпоинта (можно добавить
авторизацию и логику).

**8. Почему статические маршруты не видны в OpenAPI?**
`app.mount` подключает отдельное ASGI-приложение; его пути не участвуют в генерации
схемы основного приложения.

**9. Что делает параметр `html=True` у `StaticFiles`?**
При запросе каталога отдаёт `index.html` — удобно для SPA/статических сайтов.

**10. Почему mount на `/` нужно делать последним?**
Монтирование на корень перехватывает все необъявленные пути и может затенить ваши
API-эндпоинты. Делайте его после регистрации роутов или держите API под префиксом `/api`.

**11. Как задать имя файла при скачивании?**
Параметр `filename` у `FileResponse` (устанавливает `Content-Disposition: attachment`).

**12. Можно ли указать SPDX-лицензию вместо URL?**
В OpenAPI 3.1 (FastAPI новых версий) — да: `license_info={"name": "...", "identifier": "Apache-2.0"}`.

**13. Как сгенерировать URL до статического файла в коде?**
Через `request.url_for("static", path="css/main.css")`, где `static` — это `name` из mount.

**14. Стоит ли отдавать статику через FastAPI в продакшене?**
Для небольших проектов — допустимо. В нагруженном проде статику обычно отдают через
Nginx/CDN, чтобы разгрузить приложение.

## Подводные камни (gotchas)

- **`openapi_url=None` ломает `/docs` и `/redoc`** — без схемы документации нечего отображать.
- **Mount на `/`** может затенить API-роуты — монтируйте последним или используйте префикс.
- **Относительный `directory`** зависит от cwd процесса — лучше строить абсолютный путь
  через `Path(__file__).resolve().parent`.
- **Несовпадение имени тега** в `tags=[...]` и `openapi_tags` — описание тега не подтянется.
- **Статика не в OpenAPI** — не ищите смонтированные пути в схеме, их там нет.
- **Безопасность доки** — открытая `/docs` в проде раскрывает структуру API; решайте осознанно.
- **`FileResponse` с несуществующим файлом** вернёт ошибку — проверяйте наличие файла.

## Лучшие практики

- Всегда заполняйте `title`, `description`, `version`, `contact` — это лицо вашего API.
- Описывайте теги через `openapi_tags` и упорядочивайте их логично.
- Стройте абсолютные пути к статике/файлам через `pathlib.Path`.
- Держите API под префиксом (`/api`), если монтируете SPA на `/`.
- В продакшене решите вопрос доступности `/docs` (отключить или защитить авторизацией).
- Тяжёлую статику в проде отдавайте через Nginx/CDN; `StaticFiles` — для простых случаев.
- Скрывайте служебные эндпоинты (favicon и т.п.) из схемы через `include_in_schema=False`.

## Шпаргалка

```python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from fastapi.responses import FileResponse

app = FastAPI(
    title="...", description="...(Markdown)", summary="...",
    version="1.0.0",
    terms_of_service="https://.../terms",
    contact={"name": "...", "email": "..."},
    license_info={"name": "MIT", "url": "..."},
    openapi_tags=[{"name": "x", "description": "..."}],
    docs_url="/docs",        # None -> отключить Swagger UI
    redoc_url="/redoc",      # None -> отключить ReDoc
    openapi_url="/openapi.json",  # None -> отключить схему (и доку)
)

# Статика (директория)
app.mount("/static", StaticFiles(directory="static"), name="static")
# SPA: app.mount("/", StaticFiles(directory="dist", html=True))  # последним!

# Отдельный файл
@app.get("/dl")
async def dl():
    return FileResponse("a.pdf", media_type="application/pdf", filename="a.pdf")
```

| Что                  | Параметр / инструмент              |
|----------------------|------------------------------------|
| Заголовок/версия     | `title`, `version`                 |
| Описание (Markdown)  | `description`, `summary`           |
| Контакты/лицензия    | `contact`, `license_info`          |
| Описания тегов       | `openapi_tags`                     |
| URL Swagger/ReDoc    | `docs_url`, `redoc_url`            |
| URL схемы            | `openapi_url`                      |
| Отключить доку       | передать `None` соответствующему URL |
| Папка статики        | `StaticFiles` + `app.mount`        |
| Один файл            | `FileResponse`                     |
