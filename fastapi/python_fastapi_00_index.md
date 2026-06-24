# FastAPI — Полный индекс материалов для подготовки к собеседованию

Сборник конспектов по FastAPI: базовый **Tutorial (User Guide)** и **Advanced User Guide**. Все материалы — на русском, с современным стилем: **Pydantic v2**, `Annotated[...]`, `async/await`. Каждый файл содержит теорию, разбор с примерами, полный рабочий пример, вопросы для собеседования (Q/A), подводные камни, лучшие практики и шпаргалку.

Официальная документация: <https://fastapi.tiangolo.com/>

---

## Tutorial (User Guide)

Базовый курс — то, что нужно знать для уверенного старта и middle-уровня.

| # | Файл | Тема | Краткое описание |
|---|------|------|------------------|
| 01 | [python_fastapi_01_first_steps.md](./python_fastapi_01_first_steps.md) | Первые шаги | Что такое FastAPI, ASGI, uvicorn, path operations, декораторы методов, автодокументация (Swagger/ReDoc), `async def` vs `def`. |
| 02 | [python_fastapi_02_path_query_params.md](./python_fastapi_02_path_query_params.md) | Path и Query параметры | Параметры пути и строки запроса, типизация и валидация, `Path`/`Query` через `Annotated`, ограничения (gt, le, min_length, regex). |
| 03 | [python_fastapi_03_request_body.md](./python_fastapi_03_request_body.md) | Тело запроса | Pydantic v2 модели, тело запроса, вложенные модели, несколько тел, `Body`, валидация и сериализация. |
| 04 | [python_fastapi_04_params_cookie_header.md](./python_fastapi_04_params_cookie_header.md) | Cookie и Header параметры | Чтение cookie и заголовков через `Cookie`/`Header`, конвертация имён, списки заголовков, дубликаты. |
| 05 | [python_fastapi_05_response_model.md](./python_fastapi_05_response_model.md) | Модель ответа | `response_model`, фильтрация полей, `response_model_exclude_unset`, отдельные input/output модели, `status_code`. |
| 06 | [python_fastapi_06_form_files.md](./python_fastapi_06_form_files.md) | Формы и файлы | `Form`, `File`, `UploadFile`, multipart, загрузка файлов, комбинирование форм и файлов. |
| 07 | [python_fastapi_07_error_handling.md](./python_fastapi_07_error_handling.md) | Обработка ошибок | `HTTPException`, кастомные обработчики, `RequestValidationError`, переопределение ответов об ошибках. |
| 08 | [python_fastapi_08_configuration.md](./python_fastapi_08_configuration.md) | Конфигурация path operation | Метаданные, теги, summary/description, deprecated, response description, документирование эндпоинтов. |
| 09 | [python_fastapi_09_dependencies.md](./python_fastapi_09_dependencies.md) | Зависимости (DI) | `Depends`, переиспользуемая логика, под-зависимости, зависимости-классы, `yield`, зависимости на уровне роутера/приложения. |
| 10 | [python_fastapi_10_security.md](./python_fastapi_10_security.md) | Безопасность | OAuth2, JWT, `OAuth2PasswordBearer`, хеширование паролей, scopes, аутентификация и авторизация. |
| 11 | [python_fastapi_11_middleware_cors.md](./python_fastapi_11_middleware_cors.md) | Middleware и CORS | Пользовательские middleware, `CORSMiddleware`, порядок выполнения, before/after обработки запроса. |
| 12 | [python_fastapi_12_databases.md](./python_fastapi_12_databases.md) | Базы данных | Работа с БД (SQLAlchemy/SQLModel), сессии через зависимости, CRUD, асинхронные драйверы. |
| 13 | [python_fastapi_13_bigger_apps.md](./python_fastapi_13_bigger_apps.md) | Большие приложения | Структура проекта, `APIRouter`, разбиение на модули, `include_router`, префиксы и теги. |
| 14 | [python_fastapi_14_background_tasks.md](./python_fastapi_14_background_tasks.md) | Фоновые задачи | `BackgroundTasks`, отложенная работа после ответа, отличия от Celery/очередей. |
| 15 | [python_fastapi_15_metadata_static.md](./python_fastapi_15_metadata_static.md) | Метаданные и статика | Метаданные приложения и тегов, кастомизация OpenAPI/docs URL, `StaticFiles`. |
| 16 | [python_fastapi_16_testing.md](./python_fastapi_16_testing.md) | Тестирование | `TestClient`, pytest, тестирование эндпоинтов, фикстуры, базовый `dependency_overrides`. |

---

## Advanced User Guide

Продвинутые темы — уровень middle/senior, то, что отличает «умею собирать API» от «понимаю, как оно работает».

| # | Файл | Тема | Краткое описание |
|---|------|------|------------------|
| 01 | [python_fastapi_adv_01_path_operation.md](./python_fastapi_adv_01_path_operation.md) | Продвинутая конфигурация path operation | Дополнительные status codes, кастомизация OpenAPI на уровне операции, `openapi_extra`, расширенные ответы. |
| 02 | [python_fastapi_adv_02_responses.md](./python_fastapi_adv_02_responses.md) | Продвинутые ответы | `Response` напрямую, `JSONResponse`/`HTMLResponse`/`StreamingResponse`/`FileResponse`, кастомные заголовки, cookies, документирование нескольких media types. |
| 03 | [python_fastapi_adv_03_advanced_dependencies.md](./python_fastapi_adv_03_advanced_dependencies.md) | Продвинутые зависимости | Параметризованные зависимости (callable-классы `__call__`), глобальные зависимости, кеширование, продвинутые сценарии DI. |
| 04 | [python_fastapi_adv_04_advanced_security.md](./python_fastapi_adv_04_advanced_security.md) | Продвинутая безопасность | OAuth2 scopes, несколько схем безопасности, HTTP Basic, продвинутая авторизация, `SecurityScopes`. |
| 05 | [python_fastapi_adv_05_request_dataclasses.md](./python_fastapi_adv_05_request_dataclasses.md) | Прямой Request и dataclasses | Доступ к объекту `Request`, использование dataclasses вместо Pydantic-моделей, сырые данные запроса. |
| 06 | [python_fastapi_adv_06_advanced_middleware.md](./python_fastapi_adv_06_advanced_middleware.md) | Продвинутые middleware | Pure ASGI middleware, `HTTPSRedirect`, `TrustedHost`, `GZip`, порядок и стек middleware. |
| 07 | [python_fastapi_adv_07_websockets.md](./python_fastapi_adv_07_websockets.md) | WebSockets | `@app.websocket`, приём/отправка сообщений, зависимости в WS, broadcast, обработка отключений. |
| 08 | [python_fastapi_adv_08_lifespan_events.md](./python_fastapi_adv_08_lifespan_events.md) | События жизненного цикла | `lifespan` (asynccontextmanager), startup/shutdown, инициализация ресурсов (пулы БД, ML-модели). |
| 09 | [python_fastapi_adv_09_testing_advanced.md](./python_fastapi_adv_09_testing_advanced.md) | Продвинутое тестирование | Async-тесты (`httpx.AsyncClient`/ASGITransport), тестирование WebSocket, событий, `dependency_overrides`. |
| 10 | [python_fastapi_adv_10_settings.md](./python_fastapi_adv_10_settings.md) | Настройки и окружение | `pydantic-settings`, `BaseSettings`, переменные окружения, `.env`, настройки как зависимость, кеширование `lru_cache`. |
| 11 | [python_fastapi_adv_11_openapi_callbacks_webhooks.md](./python_fastapi_adv_11_openapi_callbacks_webhooks.md) | OpenAPI, callbacks, webhooks | Кастомизация OpenAPI-схемы, callbacks, webhooks, расширение документации. |
| 12 | [python_fastapi_adv_12_behind_proxy.md](./python_fastapi_adv_12_behind_proxy.md) | За прокси | `root_path`, работа за nginx/Traefik, проксирование, корректные URL в docs, заголовки прокси. |
| 13 | [python_fastapi_adv_13_clients_wsgi.md](./python_fastapi_adv_13_clients_wsgi.md) | Клиенты и WSGI | Генерация клиентов из OpenAPI, монтирование WSGI-приложений (`WSGIMiddleware`), интеграция с Flask/Django. |
| 14 | [python_fastapi_adv_14_templates_graphql.md](./python_fastapi_adv_14_templates_graphql.md) | Шаблоны и GraphQL | Jinja2-шаблоны (`Jinja2Templates`, `TemplateResponse`), статика; GraphQL через Strawberry (`GraphQLRouter`), REST vs GraphQL. |
| 15 | [python_fastapi_adv_15_custom_request_route.md](./python_fastapi_adv_15_custom_request_route.md) | Кастомные Request и APIRoute | Свой класс `Request` (кеш тела, gzip), кастомный `APIRoute`, `route_class`, `get_route_handler`, тайминг/логирование/десериализация. |

---

## Топ-темы для собеседования

Темы, которые спрашивают чаще всего. Если времени мало — учите в первую очередь это.

| Приоритет | Тема | Где смотреть | Почему важно |
|---|------|--------------|--------------|
| 🔴 must | **Зависимости (Dependency Injection)** | 09, adv_03 | Сердце FastAPI: `Depends`, под-зависимости, `yield`, кеширование, переиспользование. Спрашивают почти всегда. |
| 🔴 must | **Безопасность / JWT / OAuth2** | 10, adv_04 | `OAuth2PasswordBearer`, JWT, хеширование паролей, scopes. Любимая практическая тема. |
| 🔴 must | **`response_model`** | 05 | Контракт ответа, фильтрация полей, разделение input/output моделей, `exclude_unset`. |
| 🔴 must | **`async def` vs `def`** | 01 | Когда что выбрать, event loop, threadpool для sync-функций, блокирующие вызовы. Концептуальный вопрос-ловушка. |
| 🟠 high | **Middleware** | 11, adv_06 | Порядок выполнения, CORS, чистый ASGI-middleware vs `@app.middleware("http")`. |
| 🟠 high | **Lifespan / события** | adv_08 | `lifespan` (asynccontextmanager), инициализация пулов БД/ML-моделей, отличие от startup/shutdown. |
| 🟠 high | **Тестирование с `dependency_overrides`** | 16, adv_09 | Подмена зависимостей (БД, аутентификация) в тестах — ключ к изолированному тестированию. |
| 🟡 mid | Pydantic v2 / валидация | 03 | Модели, валидаторы, сериализация, `model_config`. |
| 🟡 mid | Обработка ошибок | 07 | `HTTPException`, кастомные обработчики, `RequestValidationError`. |
| 🟡 mid | Структура больших приложений | 13 | `APIRouter`, модульность, префиксы, теги. |
| 🟢 plus | WebSockets, BackgroundTasks, GraphQL, root_path | adv_07, 14, adv_14, adv_12 | Узкоспециализированные, но выгодно отличают кандидата. |

### Частые концептуальные вопросы «на засыпку»

- Почему `async def`-обработчик с блокирующим вызовом (например, `time.sleep` или sync-драйвер БД) тормозит весь сервер?
- Что произойдёт, если объявить обработчик как `def` (не async)? (FastAPI выполнит его в threadpool.)
- Как FastAPI использует подсказки типов для валидации, сериализации и документации?
- В чём разница между middleware, зависимостью и кастомным `APIRoute`?
- Как подменить зависимость (например, реальную БД на тестовую) без изменения кода эндпоинтов?
- Чем `lifespan` лучше устаревших `@app.on_event("startup")`?

---

## Рекомендованный порядок изучения

**Этап 1 — основы (обязательно):**
01 → 02 → 03 → 05 → 07

**Этап 2 — рабочее ядро:**
09 (зависимости) → 04 → 06 → 13 → 12

**Этап 3 — продакшен:**
10 (безопасность) → 11 (middleware/CORS) → 14 (фоновые задачи) → 15 (метаданные/статика) → 16 (тестирование)

**Этап 4 — Advanced (middle/senior):**
adv_08 (lifespan) → adv_03 (продвинутые зависимости) → adv_04 (продвинутая безопасность) → adv_02 (ответы) → adv_06 (middleware) → adv_09 (тестирование)

**Этап 5 — специализированное (по необходимости):**
adv_01 → adv_05 → adv_07 (WebSockets) → adv_10 (settings) → adv_11 → adv_12 → adv_13 → adv_14 (шаблоны/GraphQL) → adv_15 (кастомные Request/Route)

> Совет: после каждого файла прорабатывайте раздел «Частые вопросы на собеседовании» вслух — это лучшая проверка готовности.
