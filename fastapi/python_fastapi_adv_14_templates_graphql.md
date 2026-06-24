# FastAPI Advanced — Шаблоны (Jinja2) и GraphQL — конспект и вопросы

## О чём раздел

Этот раздел про две «нетипичные» для чистого JSON-API задачи, которые тем не менее встречаются на практике и на собеседованиях:

1. **Серверный рендеринг HTML через шаблоны Jinja2** — когда FastAPI отдаёт не JSON, а готовые HTML-страницы. Используется для админок, лендингов, отчётов, писем, SSR-страниц, страниц логина OAuth и т.п. FastAPI наследует механизм шаблонов от **Starlette** (`Jinja2Templates`, `TemplateResponse`).

2. **GraphQL** — альтернативная REST парадигма построения API. FastAPI **не включает GraphQL из коробки**, но отлично интегрируется со сторонними библиотеками. Официально рекомендуется **Strawberry** (типизированная, на dataclass-подобных декораторах, дружит с подсказками типов и async).

Оба механизма «прикручиваются» поверх обычного ASGI-приложения FastAPI и сосуществуют с обычными REST-эндпоинтами.

## Ключевые концепции

- **Jinja2** — самый популярный шаблонизатор Python (тот же, что в Flask). Синтаксис: `{{ переменная }}` для вывода, `{% for %} / {% if %}` для логики, наследование шаблонов через `{% extends %}` и `{% block %}`.
- **`Jinja2Templates`** — обёртка Starlette над окружением Jinja2. Создаётся один раз с указанием каталога шаблонов: `templates = Jinja2Templates(directory="templates")`.
- **`TemplateResponse`** — специальный `Response`, который рендерит шаблон с контекстом и отдаёт `text/html`. **Требует объект `Request` в контексте** — без него Jinja2-функции вроде `url_for` не работают.
- **Современная сигнатура** (Starlette ≥ 0.29): `templates.TemplateResponse(request, "page.html", {"ctx": ...})` — `request` первым позиционным аргументом. Старая (deprecated): `TemplateResponse("page.html", {"request": request, ...})`.
- **Статика для шаблонов** — CSS/JS/картинки отдаются через `StaticFiles`, смонтированный на отдельный путь: `app.mount("/static", StaticFiles(directory="static"), name="static")`. В шаблоне ссылка строится через `url_for('static', path='...')`.
- **GraphQL** — единый эндпоинт (обычно `POST /graphql`), клиент сам описывает, какие поля ему нужны; сервер возвращает ровно их. Сильная типизация через **schema** (типы, запросы Query, мутации Mutation, подписки Subscription).
- **Strawberry** — code-first GraphQL-библиотека на аннотациях типов. Интеграция с FastAPI через `GraphQLRouter`, который включается как обычный роутер: `app.include_router(graphql_app, prefix="/graphql")`.
- **`GraphQLRouter`** — ASGI-приложение/роутер, который оборачивает схему Strawberry, отдаёт GraphQL-эндпоинт и встроенный IDE (GraphiQL).
- **REST vs GraphQL** — две парадигмы; выбор зависит от характера клиентов, гибкости запросов, кеширования и команды.

## Подробный разбор с примерами кода

### 1. Установка зависимостей

```bash
# Для шаблонов
pip install jinja2
# StaticFiles требует aiofiles (для асинхронной отдачи файлов)
pip install aiofiles
# Для GraphQL
pip install "strawberry-graphql[fastapi]"
```

### 2. Структура проекта под шаблоны

```
project/
├── main.py
├── templates/
│   ├── base.html
│   ├── index.html
│   └── item.html
└── static/
    ├── styles.css
    └── app.js
```

### 3. Настройка Jinja2Templates и базовый рендеринг

```python
from pathlib import Path
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates

app = FastAPI()

# Базовый каталог проекта — чтобы пути не зависели от cwd, из которого запущен uvicorn.
BASE_DIR = Path(__file__).resolve().parent

# Окружение Jinja2 создаём один раз на всё приложение (дорогой объект).
templates = Jinja2Templates(directory=BASE_DIR / "templates")

# Монтируем статику. name="static" нужен, чтобы в шаблоне работал url_for('static', ...).
app.mount("/static", StaticFiles(directory=BASE_DIR / "static"), name="static")


# response_class=HTMLResponse — чтобы в OpenAPI было видно, что отдаётся HTML, а не JSON.
@app.get("/", response_class=HTMLResponse)
async def read_index(request: Request):
    # СОВРЕМЕННАЯ сигнатура: request — первый позиционный аргумент.
    # Дальше — имя шаблона и словарь контекста.
    return templates.TemplateResponse(
        request,
        "index.html",
        {"title": "Главная", "items": ["яблоко", "груша", "слива"]},
    )
```

Важно: параметр `request: Request` **обязателен** в обработчике — он нужен Jinja2 для генерации URL (`url_for`) и доступен в шаблоне как `request`.

### 4. Шаблоны Jinja2 с наследованием

`templates/base.html`:

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="utf-8">
    <title>{% block title %}Сайт{% endblock %}</title>
    <!-- url_for('static', path=...) строит абсолютный URL к статике.
         Работает только если в контексте есть request. -->
    <link rel="stylesheet" href="{{ url_for('static', path='styles.css') }}">
</head>
<body>
    <header><a href="{{ url_for('read_index') }}">Главная</a></header>
    <main>
        {% block content %}{% endblock %}
    </main>
    <script src="{{ url_for('static', path='app.js') }}"></script>
</body>
</html>
```

`templates/index.html`:

```html
{% extends "base.html" %}

{% block title %}{{ title }}{% endblock %}

{% block content %}
    <h1>{{ title }}</h1>
    <ul>
        {% for item in items %}
            <li>{{ item }}</li>
        {% else %}
            <li>Список пуст</li>
        {% endfor %}
    </ul>
{% endblock %}
```

`url_for('read_index')` ссылается на эндпоинт **по имени функции-обработчика** — это удобно, ссылки не ломаются при смене пути.

### 5. Передача данных и кастомный статус

```python
from fastapi import HTTPException

FAKE_DB = {1: {"name": "Ноутбук", "price": 1200}}


@app.get("/items/{item_id}", response_class=HTMLResponse)
async def read_item(request: Request, item_id: int):
    item = FAKE_DB.get(item_id)
    if item is None:
        # Можно вернуть HTML страницу-ошибку с нужным статусом.
        return templates.TemplateResponse(
            request,
            "item.html",
            {"item": None, "item_id": item_id},
            status_code=404,
        )
    return templates.TemplateResponse(
        request,
        "item.html",
        {"item": item, "item_id": item_id},
    )
```

### 6. Глобальные переменные и фильтры Jinja2

```python
import datetime

templates = Jinja2Templates(directory="templates")

# Глобальные значения/функции, доступные во ВСЕХ шаблонах.
templates.env.globals["site_name"] = "Моя компания"
templates.env.globals["now"] = datetime.datetime.now


# Кастомный фильтр: {{ price | rub }}
def format_rub(value: float) -> str:
    return f"{value:,.2f} ₽".replace(",", " ")


templates.env.filters["rub"] = format_rub
```

В шаблоне: `<footer>{{ site_name }} — {{ now().year }}</footer>` и `<span>{{ item.price | rub }}</span>`.

### 7. Почему FastAPI не включает GraphQL из коробки

- FastAPI/Starlette сфокусированы на REST + OpenAPI. GraphQL — это отдельная объёмная парадигма со своей схемой, исполнением, валидацией.
- Экосистема GraphQL для Python уже зрелая и развивается отдельно (Strawberry, Ariadne, Graphene). Дублировать её в ядре нет смысла.
- GraphQL по сути работает как **одно ASGI-приложение на одном пути**, поэтому его легко «вмонтировать» в FastAPI без поддержки в ядре.
- Раньше FastAPI поставлял хелпер на Graphene, но его убрали в пользу полноценной интеграции сторонних библиотек. Сейчас в документации **рекомендуется Strawberry**.

### 8. Интеграция Strawberry через GraphQLRouter

```python
import strawberry
from fastapi import FastAPI
from strawberry.fastapi import GraphQLRouter


# --- Описываем GraphQL-типы через декораторы Strawberry (code-first) ---
@strawberry.type
class Book:
    title: str
    author: str


# Корневой Query-тип: то, что можно «читать».
@strawberry.type
class Query:
    @strawberry.field
    def books(self) -> list[Book]:
        # Здесь обычно поход в БД/сервис.
        return [Book(title="1984", author="Оруэлл")]

    @strawberry.field
    def book(self, title: str) -> Book | None:
        return Book(title=title, author="Неизвестен")


# Корневой Mutation-тип: то, что меняет данные.
@strawberry.type
class Mutation:
    @strawberry.mutation
    def add_book(self, title: str, author: str) -> Book:
        # ... сохранение в БД ...
        return Book(title=title, author=author)


# Собираем схему из Query и Mutation.
schema = strawberry.Schema(query=Query, mutation=Mutation)

# GraphQLRouter оборачивает схему в ASGI-роутер с эндпоинтом и GraphiQL.
graphql_app = GraphQLRouter(schema)

app = FastAPI()
# Подключаем как обычный роутер. Теперь GraphQL живёт на /graphql.
app.include_router(graphql_app, prefix="/graphql")
```

Запрос (POST `/graphql`):

```graphql
query {
  books {
    title
    author
  }
}
```

Мутация:

```graphql
mutation {
  addBook(title: "Дюна", author: "Герберт") {
    title
  }
}
```

GraphiQL (встроенная IDE) откроется при GET-запросе на `/graphql` в браузере.

### 9. Async-резолверы и контекст/зависимости в Strawberry

```python
import strawberry
from fastapi import Depends, FastAPI, Request
from strawberry.fastapi import GraphQLRouter


# Зависимость FastAPI, результат которой попадёт в контекст GraphQL.
async def get_db():
    db = {"connected": True}  # представим пул соединений
    yield db


# Кастомный context_getter связывает зависимости FastAPI с GraphQL-резолверами.
async def get_context(request: Request, db=Depends(get_db)):
    return {"request": request, "db": db}


@strawberry.type
class Query:
    # Резолвер может быть АСИНХРОННЫМ — Strawberry это поддерживает.
    @strawberry.field
    async def current_path(self, info: strawberry.Info) -> str:
        # info.context — это то, что вернул get_context.
        request: Request = info.context["request"]
        return request.url.path


schema = strawberry.Schema(query=Query)
graphql_app = GraphQLRouter(schema, context_getter=get_context)

app = FastAPI()
app.include_router(graphql_app, prefix="/graphql")
```

### 10. REST vs GraphQL — сравнение

| Критерий | REST | GraphQL |
|---|---|---|
| Эндпоинты | Много (ресурс = URL) | Один (`/graphql`) |
| Форма ответа | Сервер решает, что вернуть | Клиент запрашивает нужные поля |
| Over/under-fetching | Часто (лишние/недостающие данные) | Минимизируется по дизайну |
| Несколько ресурсов за запрос | Несколько запросов | Один запрос с вложенностью |
| HTTP-кеширование | Простое (GET + URL) | Сложное (обычно POST, нужен слой клиента) |
| Версионирование | `/v1`, `/v2` | Эволюция схемы (deprecation полей) |
| Документация | OpenAPI/Swagger | Интроспекция схемы, GraphiQL |
| Загрузка файлов | Просто (multipart) | Неудобно (нужны расширения спецификации) |
| Кривая входа | Низкая | Выше (схема, резолверы, N+1) |
| Проблема N+1 | Реже | Частая, нужен DataLoader |

### Когда что выбирать

- **REST** — публичные API, простые CRUD-сервисы, важно HTTP-кеширование/CDN, файлы, микросервисы «сервис-сервис», команда без GraphQL-опыта.
- **GraphQL** — богатые клиенты (мобильные/SPA) с разными требованиями к данным, много связанных сущностей, нужно агрегировать несколько источников, хочется одного гибкого контракта вместо десятков эндпоинтов.
- **Гибрид** — частый прод-вариант: REST для интеграций/вебхуков/файлов + GraphQL для UI. FastAPI позволяет держать оба в одном приложении.

## Полный рабочий пример

```python
# main.py — FastAPI с Jinja2-шаблонами, статикой и GraphQL (Strawberry) одновременно.
from pathlib import Path

import strawberry
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
from strawberry.fastapi import GraphQLRouter

BASE_DIR = Path(__file__).resolve().parent

app = FastAPI(title="Templates + GraphQL demo")

# --- Шаблоны и статика ---
templates = Jinja2Templates(directory=BASE_DIR / "templates")
templates.env.globals["site_name"] = "Книжный магазин"
app.mount("/static", StaticFiles(directory=BASE_DIR / "static"), name="static")

# Имитация БД книг.
BOOKS = [
    {"id": 1, "title": "1984", "author": "Оруэлл"},
    {"id": 2, "title": "Дюна", "author": "Герберт"},
]


@app.get("/", response_class=HTMLResponse)
async def index(request: Request):
    # Рендерим список книг как HTML-страницу.
    return templates.TemplateResponse(
        request, "index.html", {"title": "Каталог", "books": BOOKS}
    )


@app.get("/books/{book_id}", response_class=HTMLResponse)
async def book_page(request: Request, book_id: int):
    book = next((b for b in BOOKS if b["id"] == book_id), None)
    if book is None:
        raise HTTPException(status_code=404, detail="Книга не найдена")
    return templates.TemplateResponse(request, "book.html", {"book": book})


# --- GraphQL ---
@strawberry.type
class Book:
    id: int
    title: str
    author: str


@strawberry.type
class Query:
    @strawberry.field
    async def books(self) -> list[Book]:
        return [Book(**b) for b in BOOKS]

    @strawberry.field
    async def book(self, id: int) -> Book | None:
        found = next((b for b in BOOKS if b["id"] == id), None)
        return Book(**found) if found else None


@strawberry.type
class Mutation:
    @strawberry.mutation
    async def add_book(self, title: str, author: str) -> Book:
        new = {"id": len(BOOKS) + 1, "title": title, "author": author}
        BOOKS.append(new)
        return Book(**new)


schema = strawberry.Schema(query=Query, mutation=Mutation)
graphql_app = GraphQLRouter(schema)
app.include_router(graphql_app, prefix="/graphql")
```

`templates/base.html`, `templates/index.html`, `templates/book.html` — как в примерах выше (наследование от base, вывод списка/деталей). Запуск: `uvicorn main:app --reload`. HTML — на `/` и `/books/1`, GraphiQL — на `/graphql`.

## Частые вопросы на собеседовании (Q/A)

**1. Откуда у FastAPI поддержка шаблонов?**
От Starlette. `Jinja2Templates` и `TemplateResponse` — это реэкспорт из Starlette через `fastapi.templating` / `fastapi.responses`.

**2. Почему `request` обязателен при рендере шаблона?**
`TemplateResponse` кладёт `request` в контекст Jinja2; он нужен для `url_for`, доступа к заголовкам/сессии и т.д. Без него рендер `url_for` упадёт. В современной сигнатуре `request` передаётся первым позиционным аргументом.

**3. Чем отличается современная сигнатура `TemplateResponse` от старой?**
Новая: `TemplateResponse(request, "name.html", {ctx})`. Старая (deprecated): `TemplateResponse("name.html", {"request": request, ...})`. Старая вызывает DeprecationWarning и может быть удалена.

**4. Как подключить CSS/JS к шаблонам?**
Смонтировать `StaticFiles` на путь (`app.mount("/static", StaticFiles(directory="static"), name="static")`) и в шаблоне строить ссылки через `url_for('static', path='styles.css')`.

**5. Зачем `response_class=HTMLResponse` в декораторе, если `TemplateResponse` сам HTML?**
Чтобы OpenAPI/Swagger корректно показывали, что эндпоинт отдаёт `text/html`, а не `application/json`. На фактический ответ это не влияет, но улучшает документацию.

**6. Почему FastAPI не имеет встроенного GraphQL?**
Это отдельная парадигма с собственной зрелой экосистемой; GraphQL встраивается как одно ASGI-приложение на одном пути. Дублировать его в ядре нецелесообразно. Рекомендуется Strawberry.

**7. Как Strawberry подключается к FastAPI?**
Через `GraphQLRouter(schema)` из `strawberry.fastapi`, который включается как обычный роутер: `app.include_router(graphql_app, prefix="/graphql")`.

**8. Можно ли использовать зависимости FastAPI (Depends) внутри GraphQL?**
Да, через `context_getter`: функция с `Depends(...)` формирует контекст, который доступен в резолверах как `info.context`.

**9. Поддерживает ли Strawberry async-резолверы?**
Да. Резолверы можно объявлять `async def` и обращаться к async-БД/сервисам через `await`.

**10. Что такое проблема N+1 в GraphQL и как её решать?**
При обходе связанных сущностей наивные резолверы делают отдельный запрос на каждый элемент. Решается батчингом через **DataLoader** (загружает связанные данные пачкой за один запрос).

**11. Когда выбрать GraphQL, а когда REST?**
GraphQL — для гибких богатых клиентов, агрегации многих источников, минимизации over/under-fetching. REST — для публичных API, простого HTTP-кеширования/CDN, файлов, интеграций «сервис-сервис».

**12. Как кешировать GraphQL по сравнению с REST?**
REST легко кешируется по URL+GET (CDN/браузер). GraphQL обычно POST на один URL — HTTP-кеш не работает «из коробки», кеширование делают на уровне клиента (Apollo/Relay) или persisted queries.

## Подводные камни (gotchas)

- **Забыли `request` в обработчике шаблона** — рендер падает или `url_for` не работает. Всегда добавляйте `request: Request`.
- **Относительные пути к `templates`/`static`** зависят от cwd запуска uvicorn. Используйте `Path(__file__).resolve().parent`, иначе «работает у меня, падает в проде».
- **Нет `name="static"`** в `app.mount(...)` — `url_for('static', ...)` не найдёт маршрут.
- **`StaticFiles` без `aiofiles`** — может ругаться/работать неэффективно; ставьте `aiofiles`.
- **Старая сигнатура `TemplateResponse`** даёт DeprecationWarning — переходите на `request`-first.
- **Авто-экранирование Jinja2** включено для HTML — не отключайте `autoescape` бездумно, иначе XSS. Для доверенного HTML используйте `| safe` точечно.
- **GraphQL и проблема N+1** — без DataLoader большой граф убивает БД количеством запросов.
- **GraphQL ломает HTTP-кеширование** — не ждите CDN-кеша «бесплатно», как у REST GET.
- **Интроспекция схемы в проде** — может раскрывать структуру API; иногда её отключают по соображениям безопасности.
- **Глубокие/дорогие запросы в GraphQL** — клиент может запросить чудовищно вложенный граф; нужны ограничения глубины/сложности (query depth/cost limiting).

## Лучшие практики

- Создавайте `Jinja2Templates` и `StaticFiles` **один раз** на приложение, пути стройте от `BASE_DIR`.
- Используйте **наследование шаблонов** (`base.html` + блоки), глобалы и фильтры для повторного кода.
- Указывайте `response_class=HTMLResponse` для HTML-эндпоинтов ради корректного OpenAPI.
- Не отключайте автоэкранирование; пользовательские данные не вставляйте через `| safe`.
- Для GraphQL используйте **Strawberry** (типобезопасность, async, дружба с type hints).
- Связывайте GraphQL с инфраструктурой FastAPI через `context_getter` (БД, аутентификация, request).
- Решайте N+1 через **DataLoader**, ограничивайте глубину/сложность запросов.
- Не пытайтесь заменить REST на GraphQL везде — комбинируйте: REST для интеграций/файлов, GraphQL для UI.
- Отдавайте статику в проде через nginx/CDN, а не через `StaticFiles`, под высокой нагрузкой.

## Шпаргалка

```python
# --- Шаблоны ---
from fastapi.templating import Jinja2Templates
from fastapi.staticfiles import StaticFiles
from fastapi.responses import HTMLResponse

templates = Jinja2Templates(directory=BASE_DIR / "templates")
app.mount("/static", StaticFiles(directory=BASE_DIR / "static"), name="static")

@app.get("/", response_class=HTMLResponse)
async def page(request: Request):
    return templates.TemplateResponse(request, "index.html", {"x": 1})

# В шаблоне:
#   {{ url_for('static', path='styles.css') }}
#   {% extends "base.html" %} / {% block content %}{% endblock %}

# --- GraphQL (Strawberry) ---
import strawberry
from strawberry.fastapi import GraphQLRouter

@strawberry.type
class Query:
    @strawberry.field
    async def hello(self) -> str: return "привет"

schema = strawberry.Schema(query=Query)        # +mutation=, +subscription=
graphql_app = GraphQLRouter(schema)            # +context_getter=
app.include_router(graphql_app, prefix="/graphql")
```

| Что | Чем |
|---|---|
| Шаблонизатор | Jinja2 (`pip install jinja2`) |
| Класс шаблонов | `Jinja2Templates(directory=...)` |
| Ответ HTML | `TemplateResponse(request, name, ctx)` |
| Статика | `StaticFiles` + `url_for('static', path=...)` |
| GraphQL-библиотека | Strawberry (рекомендуется) |
| Подключение GraphQL | `GraphQLRouter` + `include_router` |
| Контекст/DI в GraphQL | `context_getter` с `Depends` |
| N+1 в GraphQL | DataLoader |
