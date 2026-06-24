# FastAPI Advanced — Генерация клиентов и интеграция WSGI — конспект и вопросы

## О чём раздел

Две темы, связанные с интеграцией FastAPI во внешний мир:

1. **Генерация типобезопасных клиентов** из OpenAPI-схемы. FastAPI отдаёт полноценную OpenAPI 3.1-схему, по которой генераторы (openapi-ts / `@hey-api/openapi-ts`, openapi-generator и др.) создают готовый клиент с типами. Здесь ключевую роль играет `operationId` и его кастомизация через `generate_unique_id_function`.

2. **Включение WSGI-приложений** (Flask, Django) в FastAPI через `WSGIMiddleware` и `app.mount`. Это позволяет постепенно мигрировать legacy-приложение на FastAPI, оставляя старые куски работающими.

## Ключевые концепции

| Концепция | Что это |
|-----------|---------|
| OpenAPI schema | Машиночитаемое описание API (`/openapi.json`), источник для генераторов клиентов. |
| Генератор клиента | Инструмент (openapi-ts и др.), создающий типизированный клиент по схеме. |
| `operationId` | Уникальный идентификатор операции в OpenAPI; влияет на имя сгенерированного метода клиента. |
| `generate_unique_id_function` | Хук FastAPI для кастомизации `operationId`. |
| WSGI | Синхронный стандарт Python-веб-приложений (Flask, Django). |
| ASGI | Асинхронный стандарт (FastAPI/Starlette). |
| `WSGIMiddleware` | Адаптер, оборачивающий WSGI-приложение в ASGI, чтобы смонтировать его в FastAPI. |
| Постепенная миграция | Стратегия: новые роуты на FastAPI, старые — на смонтированном Flask/Django. |

## Подробный разбор с примерами кода

### 1. Откуда генерируется клиент

FastAPI автоматически отдаёт OpenAPI-схему по `/openapi.json`. Генераторы клиентов читают её и создают код. Для фронтенда (TypeScript) самый популярный путь — **openapi-ts** (`@hey-api/openapi-ts`).

```bash
# Установка генератора (TypeScript)
npm install -D @hey-api/openapi-ts

# Генерация клиента из локально работающего API
npx @hey-api/openapi-ts \
  -i http://localhost:8000/openapi.json \
  -o src/client
```

Либо сохранить схему в файл и генерировать из него:

```bash
# Сохранить схему (например, из CI без поднятия сервера — через python -c)
python -c "import json; from main import app; print(json.dumps(app.openapi()))" > openapi.json
npx @hey-api/openapi-ts -i openapi.json -o src/client
```

### 2. Как `operationId` влияет на имена методов

В сгенерированном клиенте имя метода берётся из `operationId`. По умолчанию FastAPI формирует его как `имя_функции_метод_путь` — длинно и некрасиво (например, `read_items_items__get`).

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    price: float


@app.get("/items/")
async def read_items() -> list[Item]:
    # operationId по умолчанию: read_items_items__get
    # -> метод клиента: readItemsItemsGet() — некрасиво
    return [Item(name="Молоток", price=9.99)]
```

Можно задать `operation_id` вручную на конкретном эндпоинте:

```python
@app.get("/items/", operation_id="list_items")
async def read_items() -> list[Item]:
    # operationId = list_items -> метод клиента listItems()
    return [Item(name="Молоток", price=9.99)]
```

`operation_id` должен быть **уникальным** в рамках всего приложения.

### 3. Глобальная кастомизация через `generate_unique_id_function`

Чтобы не задавать `operation_id` руками на каждом эндпоинте, задают глобальную функцию генерации. Удобный приём — использовать первый тег + имя функции, чтобы получить короткие осмысленные имена.

```python
from fastapi import FastAPI
from fastapi.routing import APIRoute


def custom_generate_unique_id(route: APIRoute) -> str:
    # Берём первый тег и имя функции: items-read_items
    # openapi-ts превратит это в readItems() внутри группы Items
    tag = route.tags[0] if route.tags else "default"
    return f"{tag}-{route.name}"


app = FastAPI(generate_unique_id_function=custom_generate_unique_id)


@app.get("/items/", tags=["items"])
async def read_items():
    # operationId = items-read_items
    return [{"name": "Молоток"}]


@app.post("/items/", tags=["items"])
async def create_item():
    # operationId = items-create_item
    return {"ok": True}
```

Эту же функцию можно задавать на уровне `APIRouter` (`APIRouter(generate_unique_id_function=...)`), чтобы у разных модулей были разные правила именования.

ВАЖНО: имена должны оставаться уникальными. Если две функции называются одинаково в одном теге — будет конфликт `operationId`, и генератор/валидатор схемы выдаст ошибку.

### 4. Почему стоит кастомизировать operationId

- Короткие, читаемые имена методов клиента (`listItems` вместо `readItemsItemsGet`).
- Стабильность: при изменении пути дефолтный `operationId` меняется → ломаются имена в клиенте. Явный/кастомный `operationId` устойчив.
- Группировка по тегам: многие генераторы группируют методы по тегам в «сервисы»/«namespaces».

### 5. Преобразование типов на клиенте

Сгенерированный клиент по Pydantic-моделям получает TS-интерфейсы. Можно дополнительно настроить пост-обработку имён (например, убирать префиксы тегов) — это делается опциями генератора, а не FastAPI.

### 6. Включение WSGI-приложения (Flask)

`WSGIMiddleware` оборачивает синхронное WSGI-приложение в ASGI, после чего его монтируют через `app.mount`.

```python
from fastapi import FastAPI
from fastapi.middleware.wsgi import WSGIMiddleware
from flask import Flask, request

# --- Legacy WSGI-приложение на Flask ---
flask_app = Flask(__name__)


@flask_app.route("/")
def flask_main():
    name = request.args.get("name", "Мир")
    return f"Привет, {name}, из Flask!"


# --- Новое ASGI-приложение на FastAPI ---
app = FastAPI()


@app.get("/v2")
async def read_v2():
    return {"message": "Это новый FastAPI-эндпоинт"}


# Монтируем Flask внутрь FastAPI по префиксу /v1
# Все запросы на /v1/* пойдут в Flask
app.mount("/v1", WSGIMiddleware(flask_app))
```

Теперь:
- `GET /v2` → FastAPI.
- `GET /v1/` → Flask (`Привет, Мир, из Flask!`).
- `GET /v1/?name=Иван` → Flask (`Привет, Иван, из Flask!`).

Аналогично можно смонтировать Django (`get_wsgi_application()`).

### 7. Сценарий постепенной миграции

Стратегия strangler-fig («фиговое дерево-душитель»):

1. Берём legacy Flask/Django-приложение, монтируем его целиком под FastAPI (`app.mount("/", WSGIMiddleware(legacy))` или под префикс).
2. Постепенно переписываем эндпоинты на FastAPI как нативные ASGI-роуты.
3. По мере переноса роуты «отрезаются» от legacy и обслуживаются FastAPI.
4. Когда legacy опустеет — убираем `WSGIMiddleware`.

Преимущества: новые эндпоинты получают async, валидацию Pydantic, авто-документацию, типобезопасные клиенты; старые продолжают работать без переписывания.

ОГРАНИЧЕНИЕ: WSGI-часть остаётся синхронной и работает в пуле потоков — она не получает выгод от async. Тяжёлый трафик лучше переносить на нативный FastAPI в первую очередь.

## Полный рабочий пример

```python
from fastapi import FastAPI
from fastapi.middleware.wsgi import WSGIMiddleware
from fastapi.routing import APIRoute
from flask import Flask, request
from pydantic import BaseModel


# ===== Кастомизация operationId для красивых имён в клиенте =====
def custom_generate_unique_id(route: APIRoute) -> str:
    tag = route.tags[0] if route.tags else "default"
    return f"{tag}-{route.name}"


app = FastAPI(
    title="Migration API",
    generate_unique_id_function=custom_generate_unique_id,
)


# ===== Новые нативные FastAPI-эндпоинты (для генерации TS-клиента) =====
class Item(BaseModel):
    name: str
    price: float


@app.get("/items", tags=["items"])
async def list_items() -> list[Item]:
    # operationId = items-list_items -> метод клиента listItems()
    return [Item(name="Молоток", price=9.99)]


@app.post("/items", tags=["items"])
async def create_item(item: Item) -> Item:
    # operationId = items-create_item -> createItem()
    return item


# ===== Legacy WSGI (Flask), монтируется для постепенной миграции =====
flask_app = Flask(__name__)


@flask_app.route("/report")
def legacy_report():
    fmt = request.args.get("format", "html")
    return f"Старый отчёт в формате {fmt} (Flask)"


# Всё под /legacy обслуживает Flask, пока не перепишем на FastAPI
app.mount("/legacy", WSGIMiddleware(flask_app))
```

Генерация TypeScript-клиента из этой схемы:

```bash
# Поднять API: uvicorn main:app
npx @hey-api/openapi-ts -i http://localhost:8000/openapi.json -o src/client
# В клиенте появятся методы listItems() и createItem(), сгруппированные по тегу items.
# Эндпоинты Flask (/legacy/*) в OpenAPI-схему FastAPI НЕ попадают.
```

## Частые вопросы на собеседовании (Q/A)

**1. Откуда генератор берёт данные для клиента?**
Из OpenAPI-схемы FastAPI (`/openapi.json`). Это машиночитаемое описание всех эндпоинтов, моделей и типов.

**2. Что такое `operationId` и на что он влияет?**
Уникальный идентификатор операции в OpenAPI. Генераторы клиентов берут из него имя метода. Дефолтный `operationId` длинный (`read_items_items__get`).

**3. Как сделать имена методов клиента красивыми?**
Задать `operation_id="list_items"` на эндпоинте или глобально через `generate_unique_id_function`, например `f"{tag}-{route.name}"`.

**4. Чем плох дефолтный `operationId`?**
Он длинный и зависит от пути: при изменении пути меняется `operationId` → ломаются имена в сгенерированном клиенте. Кастомный id стабильнее и читабельнее.

**5. Что требуется от `operationId`?**
Уникальность в рамках всего приложения. Дубли вызовут ошибку валидации схемы и поломку генератора.

**6. Можно ли задать функцию генерации id на уровне роутера?**
Да, `APIRouter(generate_unique_id_function=...)` — у разных модулей могут быть свои правила.

**7. Как смонтировать Flask/Django в FastAPI?**
Обернуть WSGI-приложение в `WSGIMiddleware` и смонтировать: `app.mount("/legacy", WSGIMiddleware(flask_app))`.

**8. Чем WSGI отличается от ASGI?**
WSGI — синхронный стандарт (Flask, классический Django). ASGI — асинхронный (FastAPI/Starlette), поддерживает async, WebSocket, long-polling.

**9. Получает ли смонтированное WSGI-приложение преимущества async?**
Нет. Оно остаётся синхронным и выполняется в пуле потоков. Async-выгоды только у нативных FastAPI-роутов.

**10. Попадают ли эндпоинты Flask в OpenAPI-схему FastAPI?**
Нет. WSGI-приложение для FastAPI — «чёрный ящик»; его роуты в `/openapi.json` не включаются и в клиент не генерируются.

**11. Опишите стратегию постепенной миграции с Flask на FastAPI.**
Смонтировать legacy через `WSGIMiddleware`, постепенно переписывать эндпоинты на нативный FastAPI, по мере переноса убирать из legacy, в конце удалить `WSGIMiddleware`. Это паттерн strangler-fig.

**12. Можно ли генерировать клиент в CI без запущенного сервера?**
Да, выгрузить схему программно (`app.openapi()` → JSON-файл) и подать файл генератору. Это надёжнее, чем дёргать живой сервер.

## Подводные камни (gotchas)

- **Меняющийся дефолтный `operationId`**: переименовали путь/функцию — и имена методов в клиенте поехали. Лечится кастомным id.
- **Дубли `operationId`**: две функции с одинаковым именем в одном теге при `f"{tag}-{route.name}"` → конфликт. Имена должны быть уникальны.
- **WSGI-приложение не в схеме**: не ждите, что роуты Flask появятся в Swagger/клиенте FastAPI.
- **Синхронность WSGI**: тяжёлая нагрузка на Flask-часть упирается в пул потоков, async не помогает.
- **Статика/большие тела в WSGI**: стриминг и крупные загрузки через `WSGIMiddleware` менее эффективны, чем нативный ASGI.
- **Разные системы зависимостей**: DI FastAPI и расширения Flask живут раздельно; общий код/сессии БД нужно аккуратно шарить.
- **Префиксы при mount**: путь внутри Flask отсчитывается от точки монтирования; не дублируйте префикс в роутах Flask.
- **Генерация по живому серверу в CI** хрупка (порядок старта, порт). Лучше выгружать схему в файл.

## Лучшие практики

- Задавайте `generate_unique_id_function` глобально (`tag-route.name`) для чистых имён клиента и стабильности.
- Группируйте эндпоинты тегами — генераторы создают по ним удобные «сервисы».
- Версионируйте/коммитьте сгенерированный клиент или генерируйте его в CI из выгруженной `openapi.json`.
- При миграции монтируйте legacy через `WSGIMiddleware`, переносите эндпоинты постепенно (strangler-fig), приоритизируя горячие пути.
- Новые эндпоинты пишите нативно на FastAPI (async, Pydantic, типизированные ответы) — это даёт документацию и качественный клиент.
- Следите за уникальностью `operationId`, особенно при автоматическом именовании.
- Для CI-пайплайна генерируйте клиент из файла схемы, а не из живого сервера.

## Шпаргалка

```python
# Кастомный operationId на эндпоинте
@app.get("/items/", operation_id="list_items")
async def read_items(): ...

# Глобальная кастомизация
from fastapi.routing import APIRoute
def gen_id(route: APIRoute) -> str:
    return f"{route.tags[0]}-{route.name}"
app = FastAPI(generate_unique_id_function=gen_id)
# либо на роутере: APIRouter(generate_unique_id_function=gen_id)
```

```bash
# Генерация TS-клиента
npx @hey-api/openapi-ts -i http://localhost:8000/openapi.json -o src/client
# из файла (CI):
python -c "import json,main; print(json.dumps(main.app.openapi()))" > openapi.json
npx @hey-api/openapi-ts -i openapi.json -o src/client
```

```python
# Включение WSGI (Flask/Django) для миграции
from fastapi.middleware.wsgi import WSGIMiddleware
app.mount("/legacy", WSGIMiddleware(flask_app))
# Django:
# from django.core.wsgi import get_wsgi_application
# app.mount("/legacy", WSGIMiddleware(get_wsgi_application()))
```
