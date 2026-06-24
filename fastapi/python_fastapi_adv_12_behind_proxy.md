# FastAPI Advanced — За прокси и Sub Applications — конспект и вопросы

## О чём раздел

Два связанных продакшен-сценария:

1. **Работа за reverse-proxy** (Nginx, Traefik, Kubernetes Ingress) — когда приложение доступно не на корне `/`, а под префиксом пути (например, `https://example.com/api/v1`). Здесь ключевую роль играет **`root_path`** и заголовки `X-Forwarded-*`.
2. **Sub Applications (под-приложения)** — монтирование нескольких независимых ASGI/FastAPI-приложений в одно через `app.mount("/path", subapp)`. У каждого свои `/docs` и `/openapi.json`.

Понимание этого критично, потому что без корректного `root_path` ломаются Swagger UI (`Try it out` бьёт не туда), ссылки и `url_for`.

## Ключевые концепции

| Концепция | Что это |
|-----------|---------|
| reverse-proxy | Прокси (Nginx/Traefik) перед приложением: TLS, балансировка, отдача под префиксом. |
| `root_path` | Префикс пути, под которым приложение видно снаружи, но которого приложение «не видит» в своих маршрутах. |
| `--root-path` | Флаг uvicorn/CLI для задания `root_path` при старте. |
| `X-Forwarded-*` | Заголовки от прокси: `X-Forwarded-For` (IP), `X-Forwarded-Proto` (http/https), `X-Forwarded-Host`, `X-Forwarded-Prefix`. |
| `ProxyHeadersMiddleware` | Middleware uvicorn, доверяющий `X-Forwarded-*` для корректной схемы/клиента. |
| `app.mount()` | Монтирование вложенного ASGI-приложения по префиксу. |
| Sub Application | Самостоятельное FastAPI/ASGI-приложение со своими роутами, docs, middleware, события lifespan. |
| APIRouter | Способ организовать роуты внутри **одного** приложения (общая схема/middleware). |

## Подробный разбор с примерами кода

### 1. Зачем нужен `root_path`

Допустим, прокси отдаёт ваше приложение по адресу `https://example.com/api/v1`, но внутри FastAPI вы пишете роут `@app.get("/items/")`. Снаружи это `https://example.com/api/v1/items/`.

Проблема: Swagger UI грузит `/openapi.json`, а кнопка «Try it out» формирует запросы. Если приложение не знает о префиксе `/api/v1`, оно сгенерирует неправильные URL в схеме (`servers`) и Swagger будет стучаться не туда.

`root_path` сообщает приложению: «снаружи к моим путям добавлен префикс `/api/v1`». FastAPI учтёт это при генерации `servers` в OpenAPI, но **не** будет ожидать этот префикс в самих маршрутах (он «съедается» прокси/ASGI-сервером).

```python
from fastapi import FastAPI, Request

# Вариант А: задать root_path прямо в коде
app = FastAPI(root_path="/api/v1")


@app.get("/app")
async def read_main(request: Request):
    # request.scope["root_path"] содержит "/api/v1"
    return {"message": "Hello", "root_path": request.scope.get("root_path")}
```

### 2. Задание `root_path` через uvicorn

Чаще `root_path` задают при запуске, а не в коде — так конфигурация остаётся в инфраструктуре.

```bash
uvicorn main:app --host 0.0.0.0 --port 8000 --root-path /api/v1
```

Gunicorn + uvicorn worker:

```bash
gunicorn main:app -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000
# root_path можно прокинуть через переменную окружения и FastAPI(root_path=...)
```

### 3. Пример с Nginx

Nginx убирает префикс перед проксированием на приложение:

```nginx
server {
    listen 80;
    server_name example.com;

    location /api/v1/ {
        # /api/v1/items/ -> приложение получит /items/
        proxy_pass http://127.0.0.1:8000/;

        # Пробрасываем заголовки, чтобы приложение знало реальные схему/хост/клиента
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Prefix /api/v1;
    }
}
```

Здесь два варианта:
- **Nginx режет префикс** (`proxy_pass http://...:8000/;` со слешем на конце) → приложение видит `/items/`, и нужно сообщить ему `root_path=/api/v1`.
- **Nginx НЕ режет префикс** → приложение само обрабатывает `/api/v1/items/` (тогда `root_path` не нужен, но роуты содержат префикс).

Чаще выбирают первый вариант + `root_path`.

### 4. `X-Forwarded-*` и `ProxyHeadersMiddleware`

За прокси приложение видит соединение от прокси (`127.0.0.1`), а не от реального клиента, и схему `http`, а не `https`. Прокси сообщает реальные данные через заголовки:

- `X-Forwarded-For` — реальный IP клиента.
- `X-Forwarded-Proto` — `https`/`http` (важно для генерации ссылок).
- `X-Forwarded-Host` — исходный Host.
- `X-Forwarded-Prefix` — префикс пути (некоторые серверы умеют выводить из него `root_path`).

uvicorn по умолчанию доверяет этим заголовкам только от `127.0.0.1`. Управление:

```bash
# Доверять X-Forwarded-* от любого источника (только если прокси — единственный вход!)
uvicorn main:app --proxy-headers --forwarded-allow-ips '*'
```

```python
from fastapi import FastAPI, Request

app = FastAPI()


@app.get("/info")
async def info(request: Request):
    # request.url учитывает X-Forwarded-Proto/Host, если включён proxy-headers
    return {
        "client_host": request.client.host,    # реальный IP при включённых proxy-headers
        "scheme": request.url.scheme,           # https за корректно настроенным прокси
        "url": str(request.url),
    }
```

ВНИМАНИЕ по безопасности: `--forwarded-allow-ips '*'` безопасно только когда внешний трафик гарантированно идёт через прокси. Иначе клиент сможет подделать свой IP/схему.

### 5. Traefik и автоматический `root_path`

Traefik с `StripPrefix`-middleware режет префикс и пробрасывает `X-Forwarded-Prefix`. Если запускать uvicorn с поддержкой вывода `root_path` из заголовка, префикс подтянется автоматически. Но самый надёжный способ — явный `--root-path`.

### 6. Sub Applications: `app.mount`

`app.mount("/subpath", subapp)` встраивает **независимое** ASGI-приложение. У под-приложения собственные роуты, документация, middleware, lifespan.

```python
from fastapi import FastAPI

# Главное приложение
app = FastAPI()


@app.get("/app")
async def read_main():
    return {"message": "Hello from main app"}


# Независимое под-приложение
subapi = FastAPI()


@subapi.get("/sub")
async def read_sub():
    return {"message": "Hello from sub app"}


# Монтируем по префиксу /subapi
app.mount("/subapi", subapi)
```

Теперь:
- `GET /app` → главное приложение.
- `GET /subapi/sub` → под-приложение.
- `GET /docs` → docs главного.
- `GET /subapi/docs` → **отдельные** docs под-приложения.

FastAPI автоматически выставляет `root_path` для под-приложения (`/subapi`), поэтому его Swagger UI работает корректно из коробки.

### 7. `mount` vs `APIRouter`

| | `app.mount(subapp)` | `app.include_router(router)` |
|-|---------------------|------------------------------|
| Что монтируется | Целое ASGI/FastAPI-приложение | Группа роутов одного приложения |
| OpenAPI-схема | Отдельная (свой `/openapi.json`, `/docs`) | Общая со всем приложением |
| Middleware | Своё, изолированное | Общее приложения |
| Lifespan / события | Свои | Общие приложения |
| Зависимости | Изолированы | Общие, можно навешивать на роутер |
| Можно ли смонтировать не-FastAPI ASGI/WSGI | Да (Starlette, Flask через WSGIMiddleware, статика) | Нет |
| Типичное применение | Версионирование как отдельные приложения, встраивание сторонних ASGI/WSGI, статика | Модульная организация роутов в одном API |

Правило: нужна **общая** документация и middleware → `APIRouter`. Нужны **изолированные** приложения (своя схема, свой жизненный цикл, сторонний ASGI/WSGI) → `mount`.

### 8. Монтирование статики и сторонних ASGI

```python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

app = FastAPI()
# Отдача статических файлов по /static
app.mount("/static", StaticFiles(directory="static"), name="static")
```

## Полный рабочий пример

```python
from fastapi import FastAPI, Request
from fastapi.staticfiles import StaticFiles

# --- Под-приложение v2 (изолированное, свои docs) ---
api_v2 = FastAPI(title="API v2")


@api_v2.get("/items")
async def items_v2():
    return [{"name": "Молоток", "version": 2}]


# --- Главное приложение, работает за прокси под /api ---
app = FastAPI(
    title="Main API",
    root_path="/api",          # снаружи приложение видно как https://host/api/...
)


@app.get("/")
async def root(request: Request):
    return {
        "message": "main",
        # root_path виден в scope; учитывается при генерации servers в OpenAPI
        "root_path": request.scope.get("root_path"),
        "scheme": request.url.scheme,   # https при корректных proxy-headers
        "client": request.client.host if request.client else None,
    }


# Монтируем под-приложение: доступно как /api/v2/... снаружи (/v2/... внутри app)
app.mount("/v2", api_v2)

# Статика
app.mount("/static", StaticFiles(directory="static"), name="static")
```

Запуск за прокси:

```bash
uvicorn main:app --host 0.0.0.0 --port 8000 \
    --proxy-headers --forwarded-allow-ips '127.0.0.1' \
    --root-path /api
```

Nginx:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8000/;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Проверка:
- `https://host/api/` → главное.
- `https://host/api/v2/items` → под-приложение v2.
- `https://host/api/docs` → docs главного (с правильным `servers`).
- `https://host/api/v2/docs` → docs под-приложения v2.

## Частые вопросы на собеседовании (Q/A)

**1. Что такое `root_path` и зачем он нужен?**
Это префикс пути, под которым приложение доступно снаружи через прокси, но которого нет в его собственных маршрутах. Нужен, чтобы OpenAPI-схема (`servers`) и Swagger UI «Try it out» формировали корректные URL.

**2. Где задать `root_path`?**
В коде (`FastAPI(root_path=...)`) или при запуске (`uvicorn --root-path /api`). Предпочтительно в инфраструктуре/CLI, чтобы код не зависел от деплоя.

**3. Что делает прокси с префиксом и как это связано с `root_path`?**
Если прокси режет префикс (`proxy_pass .../`), приложение видит «короткие» пути и должно знать `root_path`. Если не режет — приложение само обрабатывает полные пути, `root_path` не нужен.

**4. Зачем `X-Forwarded-*` заголовки?**
Чтобы приложение за прокси узнало реальный IP клиента (`X-Forwarded-For`), исходную схему (`X-Forwarded-Proto`, важно для https-ссылок) и хост.

**5. Почему `request.client.host` показывает 127.0.0.1?**
Потому что соединение приходит от прокси. Нужно включить `--proxy-headers` и доверять источнику через `--forwarded-allow-ips`, тогда uvicorn подставит реальный IP из `X-Forwarded-For`.

**6. Чем опасен `--forwarded-allow-ips '*'`?**
Любой клиент сможет подделать `X-Forwarded-For`/`X-Forwarded-Proto`. Допустимо только если весь трафик гарантированно проходит через доверенный прокси.

**7. В чём разница между `app.mount` и `app.include_router`?**
`mount` встраивает независимое ASGI-приложение со своей схемой, docs, middleware и lifespan. `include_router` добавляет роуты в то же приложение с общей схемой и middleware.

**8. У под-приложения отдельная документация?**
Да. У смонтированного FastAPI-приложения свои `/docs` и `/openapi.json` по его префиксу.

**9. Можно ли смонтировать Flask/Django или статику?**
Да, через `app.mount`: WSGI-приложения оборачивают в `WSGIMiddleware`, статику — `StaticFiles`. `include_router` так не умеет.

**10. Нужно ли вручную задавать `root_path` под-приложению?**
Нет, FastAPI сам выставляет `root_path` для под-приложения по префиксу `mount`, поэтому его docs работают корректно.

**11. Как версионировать API через под-приложения?**
Смонтировать `app.mount("/v1", api_v1)` и `app.mount("/v2", api_v2)` — у каждой версии своя изолированная схема и docs.

**12. Что приоритетнее при выборе: `mount` или `APIRouter`?**
Если нужна единая документация и сквозные зависимости/middleware — `APIRouter`. Если нужна изоляция (отдельные схемы, lifespan, сторонние ASGI/WSGI) — `mount`.

## Подводные камни (gotchas)

- **Swagger «Try it out» бьёт не туда** за прокси → почти всегда забыт `root_path`.
- **Двойной префикс**: задали `root_path=/api` И прокси НЕ режет префикс → пути удваиваются (`/api/api/...`). Согласуйте поведение прокси и `root_path`.
- **https-ссылки становятся http** → не включён `--proxy-headers` или прокси не шлёт `X-Forwarded-Proto`.
- **Реальный IP не виден** → не настроен `--forwarded-allow-ips`.
- **Под-приложение не наследует middleware/зависимости главного** — это фича, но часто неожиданно. Для общих — используйте `APIRouter` или дублируйте.
- **Lifespan под-приложения**: события startup/shutdown под-приложения выполняются независимо; не рассчитывайте, что главный lifespan их подхватит.
- **CORS/аутентификация** настраиваются на каждом приложении отдельно при `mount`.
- **Слеш в `proxy_pass`** в Nginx критичен: со слешем (`.../`) префикс режется, без — нет.

## Лучшие практики

- Задавайте `root_path` через `--root-path`/переменные окружения, а не хардкодом в коде.
- Всегда запускайте за прокси с `--proxy-headers` и явным списком доверенных IP.
- Согласуйте поведение прокси (режет ли префикс) с наличием `root_path`.
- Для модульной организации одного API используйте `APIRouter`; для изоляции и сторонних приложений — `mount`.
- Версионируйте через под-приложения, когда версии должны быть полностью независимы (своя схема/lifespan).
- Проверяйте корректность через `request.scope["root_path"]` и реальный `request.url`/`request.client`.
- Не доверяйте `X-Forwarded-*` без ограничения источников — иначе подделка IP/схемы.

## Шпаргалка

```python
# root_path в коде
app = FastAPI(root_path="/api/v1")

# доступ к нему
request.scope.get("root_path")
```

```bash
# root_path и proxy-headers при запуске
uvicorn main:app --root-path /api/v1 \
    --proxy-headers --forwarded-allow-ips '127.0.0.1'
```

```python
# Sub application (своя схема и /docs)
sub = FastAPI()
app.mount("/sub", sub)        # снаружи /sub/..., свои /sub/docs

# Статика и WSGI через mount
app.mount("/static", StaticFiles(directory="static"))
# app.mount("/legacy", WSGIMiddleware(flask_app))

# Router (общая схема и middleware)
app.include_router(router, prefix="/items", tags=["items"])
```

```nginx
# Nginx: режем префикс + проброс заголовков
location /api/v1/ {
    proxy_pass http://127.0.0.1:8000/;     # слеш в конце => режет префикс
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Prefix /api/v1;
}
```
