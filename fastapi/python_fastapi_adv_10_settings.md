# FastAPI Advanced — Настройки и переменные окружения — конспект и вопросы

## О чём раздел

Раздел про управление конфигурацией FastAPI-приложения: как читать настройки из переменных окружения и `.env`, как их типизировать и валидировать, кешировать и удобно переопределять в тестах. Современный инструмент для этого — **pydantic-settings** (отдельный пакет в Pydantic v2; раньше `BaseSettings` жил в самом pydantic).

Ключевая идея — **12-factor**: конфигурация хранится не в коде, а в окружении. Pydantic-settings даёт удобную, типизированную и валидируемую обёртку над переменными окружения: вы описываете класс `Settings(BaseSettings)`, а значения автоматически подтягиваются из env/`.env`, приводятся к типам и валидируются при старте.

## Ключевые концепции

- **pydantic-settings** — отдельный пакет (`pip install pydantic-settings`). В Pydantic v2 `BaseSettings` вынесен из ядра именно сюда. Импорт: `from pydantic_settings import BaseSettings, SettingsConfigDict`.
- **`BaseSettings`** — специальная Pydantic-модель, которая при инициализации сама читает значения из переменных окружения (и других источников), а не только из переданных аргументов.
- **`SettingsConfigDict`** — конфигурация модели настроек (вместо старого `class Config`). Задаёт `env_file`, `env_prefix`, `case_sensitive`, `extra` и т. д. Назначается в `model_config`.
- **Источники значений и их приоритет** (по умолчанию, сверху вниз — выше = важнее): аргументы при инициализации → переменные окружения → переменные из `.env` → секреты из директории секретов (`secrets_dir`) → значения по умолчанию в полях.
- **Типизация и валидация** — поля настроек аннотируются типами (`str`, `int`, `bool`, `list[str]`, `PostgresDsn`, `SecretStr` и т. д.). Pydantic парсит строки из env в эти типы и валидирует; ошибка валидации — на старте приложения (fail fast).
- **`.env`-файл** — файл `KEY=value` в корне проекта; подключается через `model_config = SettingsConfigDict(env_file=".env")`. Требует установленного `python-dotenv`.
- **Кеширование через `lru_cache`** — настройки создают один раз и переиспользуют через зависимость `get_settings()`, обёрнутую в `functools.lru_cache`. Это избегает повторного чтения `.env`/окружения на каждый запрос.
- **Переопределение в тестах** — через `app.dependency_overrides[get_settings] = ...`, либо очистку кеша `get_settings.cache_clear()`, либо передачу значений в конструктор `Settings(...)`.
- **Секреты** — чувствительные данные (пароли, токены) держат в `SecretStr`/`SecretBytes`, чтобы они не светились в логах и `repr`. Поддерживается чтение из файлов-секретов (Docker/K8s secrets) через `secrets_dir`.

## Подробный разбор с примерами кода

### 1. Базовый класс настроек

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    # Имя поля -> имя переменной окружения (APP_NAME, ADMIN_EMAIL, ...).
    app_name: str = "Моё приложение"
    admin_email: str
    items_per_page: int = 50
    debug: bool = False

    # model_config заменяет старый class Config из Pydantic v1.
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")


settings = Settings()  # читает env + .env, валидирует, fail fast при ошибке
```

`.env`:

```dotenv
ADMIN_EMAIL=admin@example.com
ITEMS_PER_PAGE=20
DEBUG=true
```

Pydantic сам:
- найдёт `ADMIN_EMAIL` (обязательное поле без дефолта) — если его нет, упадёт с `ValidationError` на старте;
- приведёт `ITEMS_PER_PAGE=20` (строка) к `int`;
- распарсит `DEBUG=true` в `bool` (понимает `1/0`, `true/false`, `yes/no`).

### 2. Имена переменных, регистр и префикс

По умолчанию имена полей сопоставляются с env-переменными без учёта регистра (`debug` ↔ `DEBUG`). Можно настроить:

```python
class Settings(BaseSettings):
    api_key: str
    db_host: str = "localhost"

    model_config = SettingsConfigDict(
        env_prefix="MYAPP_",     # читать MYAPP_API_KEY, MYAPP_DB_HOST
        case_sensitive=False,    # регистронезависимо (по умолчанию)
        extra="ignore",          # лишние переменные окружения игнорировать
    )
```

С `env_prefix="MYAPP_"` поле `api_key` читается из `MYAPP_API_KEY`. Это удобно, чтобы не конфликтовать с чужими переменными окружения.

`extra`:
- `"ignore"` — лишние env-переменные игнорируются (рекомендуется для настроек);
- `"forbid"` — упасть, если есть неизвестная переменная;
- `"allow"` — добавить как динамическое поле.

Явное имя переменной для конкретного поля задаётся через `alias`/`validation_alias`:

```python
from pydantic import Field

class Settings(BaseSettings):
    # Поле называется database_url, но читается из переменной DATABASE_URL.
    database_url: str = Field(validation_alias="DATABASE_URL")
```

### 3. Типизация и валидация сложных типов

```python
from pydantic import Field, PostgresDsn, AnyHttpUrl, field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    database_url: PostgresDsn                 # валидируется как postgres DSN
    redis_url: str = "redis://localhost:6379/0"
    allowed_hosts: list[str] = ["localhost"]  # из env как JSON: ["a","b"]
    max_connections: int = Field(default=10, ge=1, le=100)  # 1..100
    sentry_dsn: AnyHttpUrl | None = None      # опционально

    model_config = SettingsConfigDict(env_file=".env")


# .env:
# DATABASE_URL=postgresql://user:pass@db:5432/app
# ALLOWED_HOSTS=["api.example.com","example.com"]
# MAX_CONNECTIONS=25
```

Важно: для сложных типов (`list`, `dict`, `set`) pydantic-settings ожидает в env **JSON-строку** (`ALLOWED_HOSTS=["a","b"]`). Если хотите CSV-формат (`a,b,c`), нужен кастомный парсер:

```python
class Settings(BaseSettings):
    cors_origins: list[str] = []

    @field_validator("cors_origins", mode="before")
    @classmethod
    def split_csv(cls, v):
        # Превращаем "a,b,c" из env в список.
        if isinstance(v, str) and not v.startswith("["):
            return [item.strip() for item in v.split(",") if item.strip()]
        return v
```

### 4. Секреты — `SecretStr` и `secrets_dir`

```python
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    db_password: SecretStr           # не светится в логах/repr
    jwt_secret: SecretStr

    model_config = SettingsConfigDict(
        env_file=".env",
        secrets_dir="/run/secrets",  # Docker/K8s secrets как файлы
    )


settings = Settings()
print(settings.db_password)               # SecretStr('**********')  -> скрыто
print(settings.db_password.get_secret_value())  # реальное значение -> только явно
```

С `secrets_dir="/run/secrets"` pydantic прочитает поле `db_password` из файла `/run/secrets/db_password` — это стандартный механизм Docker/Kubernetes secrets. `SecretStr` гарантирует, что значение не попадёт в логи случайно (в `repr` показывается `**********`).

### 5. Кеширование настроек через зависимость и `lru_cache`

Создавать `Settings()` на каждый запрос — это лишнее чтение `.env`/окружения и парсинг. Оборачиваем в зависимость с кешем:

```python
from functools import lru_cache
from typing import Annotated
from fastapi import FastAPI, Depends
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    app_name: str = "Приложение"
    admin_email: str
    model_config = SettingsConfigDict(env_file=".env")


@lru_cache  # объект Settings создаётся ОДИН раз и переиспользуется
def get_settings() -> Settings:
    return Settings()


app = FastAPI()


@app.get("/info")
async def info(settings: Annotated[Settings, Depends(get_settings)]):
    return {"app_name": settings.app_name, "admin": settings.admin_email}
```

Почему `lru_cache` + зависимость, а не глобальный `settings = Settings()`:
- **тестируемость**: `get_settings` можно подменить через `app.dependency_overrides`;
- **ленивость**: настройки читаются при первом обращении, а не при импорте модуля (полезно для скорости старта и тестов);
- **кеш**: благодаря `lru_cache` без аргументов объект создаётся ровно один раз.

### 6. Переопределение настроек в тестах

Способ 1 — `dependency_overrides` (рекомендуемый):

```python
from fastapi.testclient import TestClient
from main import app, get_settings, Settings


def get_settings_override():
    # Тестовые настройки, не зависящие от .env окружения.
    return Settings(admin_email="test@example.com", app_name="Тест")


def test_info():
    app.dependency_overrides[get_settings] = get_settings_override
    client = TestClient(app)
    resp = client.get("/info")
    assert resp.json()["admin"] == "test@example.com"
    app.dependency_overrides.clear()
```

Способ 2 — сброс кеша + переменные окружения (через `monkeypatch`):

```python
def test_with_env(monkeypatch):
    monkeypatch.setenv("ADMIN_EMAIL", "env@example.com")
    get_settings.cache_clear()  # иначе вернётся старый закешированный объект
    settings = get_settings()
    assert settings.admin_email == "env@example.com"
```

Способ 3 — прямая передача в конструктор (init имеет высший приоритет):

```python
def test_direct():
    s = Settings(admin_email="x@y.z", debug=True)
    assert s.debug is True
```

### 7. Иерархия источников и её настройка

Порядок приоритета (по умолчанию, выше — важнее):

1. Аргументы конструктора `Settings(...)`.
2. Переменные окружения (`os.environ`).
3. Переменные из `.env`-файла.
4. Файлы-секреты (`secrets_dir`).
5. Значения по умолчанию в полях модели.

То есть переменная окружения перебьёт значение из `.env`, а аргумент при создании объекта перебьёт всё. Этот порядок можно переопределить, переопределив classmethod `settings_customise_sources`:

```python
from pydantic_settings import BaseSettings, PydanticBaseSettingsSource


class Settings(BaseSettings):
    foo: str = "default"

    @classmethod
    def settings_customise_sources(
        cls, settings_cls,
        init_settings, env_settings, dotenv_settings, file_secret_settings,
    ):
        # Здесь можно поменять порядок/добавить свои источники (YAML, БД и т. д.).
        # Возвращается кортеж источников по убыванию приоритета.
        return (init_settings, env_settings, dotenv_settings, file_secret_settings)
```

### 8. Несколько окружений (dev/staging/prod)

Частый паттерн — выбирать `.env`-файл по переменной `ENV`:

```python
import os
from pydantic_settings import BaseSettings, SettingsConfigDict

ENV = os.getenv("ENV", "dev")


class Settings(BaseSettings):
    database_url: str
    debug: bool = False

    model_config = SettingsConfigDict(
        env_file=(".env", f".env.{ENV}"),  # можно несколько файлов
        env_file_encoding="utf-8",
        extra="ignore",
    )
```

При списке файлов более поздние перекрывают более ранние. Так базовый `.env` дополняется/переопределяется `.env.prod`.

## Полный рабочий пример

`config.py`:

```python
from functools import lru_cache
from pydantic import SecretStr, PostgresDsn, Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    # --- Общие ---
    app_name: str = "Каталог товаров"
    environment: str = "dev"          # dev | staging | prod
    debug: bool = False

    # --- БД ---
    database_url: PostgresDsn
    db_pool_size: int = Field(default=5, ge=1, le=50)

    # --- Безопасность ---
    jwt_secret: SecretStr
    access_token_expire_minutes: int = 30

    # --- Внешние сервисы ---
    redis_url: str = "redis://localhost:6379/0"
    cors_origins: list[str] = ["http://localhost:3000"]

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        env_prefix="",          # без префикса
        case_sensitive=False,
        extra="ignore",         # лишние env-переменные игнорируем
    )


@lru_cache
def get_settings() -> Settings:
    # Кеш: объект создаётся один раз на всё приложение.
    return Settings()
```

`.env`:

```dotenv
ENVIRONMENT=dev
DEBUG=true
DATABASE_URL=postgresql://user:pass@localhost:5432/catalog
JWT_SECRET=super-secret-change-me
CORS_ORIGINS=["http://localhost:3000","https://app.example.com"]
```

`main.py`:

```python
from typing import Annotated
from fastapi import FastAPI, Depends
from fastapi.middleware.cors import CORSMiddleware
from config import Settings, get_settings


def create_app() -> FastAPI:
    settings = get_settings()
    app = FastAPI(title=settings.app_name, debug=settings.debug)

    # Используем настройки при конфигурации middleware.
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    return app


app = create_app()


@app.get("/health")
async def health(settings: Annotated[Settings, Depends(get_settings)]):
    # SecretStr не раскрываем — только факт наличия.
    return {
        "app": settings.app_name,
        "env": settings.environment,
        "debug": settings.debug,
        "has_jwt_secret": bool(settings.jwt_secret.get_secret_value()),
    }
```

`tests/test_config.py`:

```python
from fastapi.testclient import TestClient
from config import Settings, get_settings
from main import app


def get_settings_override():
    # Полностью контролируемые тестовые настройки.
    return Settings(
        environment="test",
        debug=True,
        database_url="postgresql://u:p@localhost:5432/test",
        jwt_secret="test-secret",
        cors_origins=["http://testserver"],
    )


def test_health_uses_overridden_settings():
    app.dependency_overrides[get_settings] = get_settings_override
    client = TestClient(app)

    resp = client.get("/health")
    assert resp.status_code == 200
    body = resp.json()
    assert body["env"] == "test"
    assert body["debug"] is True
    assert body["has_jwt_secret"] is True

    app.dependency_overrides.clear()


def test_secret_is_hidden_in_repr():
    s = get_settings_override()
    # SecretStr не светит значение в repr.
    assert "test-secret" not in repr(s)
    assert s.jwt_secret.get_secret_value() == "test-secret"
```

## Частые вопросы на собеседовании (Q/A)

**1. Где живёт `BaseSettings` в Pydantic v2?**
В отдельном пакете `pydantic-settings` (`pip install pydantic-settings`), импорт `from pydantic_settings import BaseSettings, SettingsConfigDict`. В Pydantic v1 он был частью самого `pydantic`. Это частая ошибка миграции.

**2. Чем `SettingsConfigDict` отличается от старого `class Config`?**
Это Pydantic v2-способ задавать конфигурацию модели — через атрибут `model_config = SettingsConfigDict(...)` вместо вложенного `class Config`. Туда кладут `env_file`, `env_prefix`, `case_sensitive`, `extra`, `secrets_dir` и т. д.

**3. Как подключить `.env`-файл?**
`model_config = SettingsConfigDict(env_file=".env")`. Нужен установленный `python-dotenv`. Можно передать список файлов — поздние перекрывают ранние.

**4. Каков приоритет источников настроек?**
Сверху вниз (выше — важнее): аргументы конструктора → переменные окружения → `.env` → файлы-секреты (`secrets_dir`) → значения по умолчанию. То есть env перебивает `.env`, а явный аргумент перебивает env.

**5. Зачем оборачивать `get_settings` в `lru_cache`?**
Чтобы объект `Settings` создавался один раз, а не на каждый запрос (иначе на каждый вызов перечитывается `.env` и заново парсятся/валидируются env-переменные). `lru_cache` без аргументов превращает функцию в кешированный синглтон.

**6. Почему предпочитают зависимость `get_settings()` глобальной переменной `settings`?**
Зависимость можно подменить в тестах через `app.dependency_overrides`, она ленивая (читает настройки при первом обращении, а не на импорте) и кешируется. Глобальная переменная читается на импорте и плохо переопределяется в тестах.

**7. Как переопределить настройки в тестах?**
Три способа: (1) `app.dependency_overrides[get_settings] = lambda: Settings(...)` — основной; (2) `monkeypatch.setenv(...)` + `get_settings.cache_clear()`; (3) передать значения прямо в конструктор `Settings(field=value)` (init — высший приоритет).

**8. Как хранить и защищать секреты?**
Использовать тип `SecretStr`/`SecretBytes` — значение скрыто в `repr`/логах, достаётся только через `.get_secret_value()`. Для Docker/K8s secrets — `secrets_dir="/run/secrets"`, тогда поле читается из одноимённого файла. В git хранят `.env.example` без значений, реальный `.env` — в `.gitignore`.

**9. Как pydantic-settings парсит списки и словари из env?**
Ожидает JSON-строку: `ALLOWED_HOSTS=["a","b"]`. Для CSV-формата (`a,b,c`) нужен кастомный `@field_validator(mode="before")`, который сам разобьёт строку.

**10. Что произойдёт, если обязательное поле без дефолта не задано в окружении?**
При создании `Settings()` Pydantic выбросит `ValidationError` — приложение упадёт на старте (fail fast). Это хорошо: лучше упасть сразу, чем словить `None` где-то в рантайме.

**11. Как реализовать настройки для разных окружений (dev/prod)?**
Выбирать `.env`-файл по переменной `ENV`: `env_file=(".env", f".env.{ENV}")` либо отдельные классы настроек/фабрика. Поздние файлы в списке перекрывают ранние.

**12. Как изменить порядок или добавить новый источник (YAML, Vault, БД)?**
Переопределить classmethod `settings_customise_sources(...)` в классе настроек и вернуть кортеж источников в нужном порядке (можно подключить кастомный `PydanticBaseSettingsSource`).

## Подводные камни (gotchas)

- **Импорт `BaseSettings` из `pydantic`** в v2 — ошибка; он переехал в `pydantic-settings`.
- **`env_file` без `python-dotenv`** — `.env` молча не подхватится. Поставьте `python-dotenv`.
- **Забыли `cache_clear()`** при тестах через env — `lru_cache` вернёт старый объект, изменения переменных не применятся.
- **Списки в env как CSV** — по умолчанию не работают; нужен JSON или кастомный валидатор.
- **`SecretStr` в логах** — печать `settings` покажет `**********`, но `print(secret.get_secret_value())` раскроет значение; не логируйте реальные значения.
- **Реальный `.env` в git** — утечка секретов. Коммитьте только `.env.example`, реальный — в `.gitignore`.
- **Глобальный `settings = Settings()` на импорте** — затрудняет тесты и читает окружение слишком рано (на этапе импорта модуля).
- **`extra="forbid"` + чужие env-переменные** — приложение упадёт из-за неизвестной переменной окружения; для настроек обычно ставят `extra="ignore"`.
- **`case_sensitive=True`** — тогда `debug` и `DEBUG` — разные имена; легко получить незаполненные поля.

## Лучшие практики (12-factor)

- Храните конфигурацию в окружении, а не в коде (принцип III «Config» 12-factor).
- Один класс `Settings(BaseSettings)` как единый типизированный источник конфигурации.
- Отдавайте настройки через зависимость `get_settings()` с `@lru_cache`, а не через глобальную переменную.
- Обязательные поля без дефолтов — пусть приложение падает на старте, если их нет (fail fast).
- Секреты — только `SecretStr`/`SecretBytes`; для контейнеров — `secrets_dir` (Docker/K8s secrets).
- Коммитьте `.env.example` (без значений), реальный `.env` — в `.gitignore`.
- Валидируйте значения (`ge/le`, `PostgresDsn`, `AnyHttpUrl`) — конфигурация проверяется при старте.
- В тестах переопределяйте `get_settings` через `dependency_overrides` или передавайте значения в конструктор.
- Разделяйте окружения через `.env.<env>` и переменную `ENV`, не плодите ветвлений в коде.
- Не логируйте объект настроек целиком, если в нём есть секреты.

## Шпаргалка

```python
# --- Установка ---
# pip install pydantic-settings python-dotenv

# --- Класс настроек ---
from functools import lru_cache
from typing import Annotated
from fastapi import Depends
from pydantic import SecretStr, Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    app_name: str = "App"
    admin_email: str                       # обязательное -> упадёт без env
    debug: bool = False
    db_password: SecretStr                 # скрытый секрет
    max_conn: int = Field(default=10, ge=1, le=100)

    model_config = SettingsConfigDict(
        env_file=".env",
        env_prefix="",
        case_sensitive=False,
        extra="ignore",
        secrets_dir="/run/secrets",        # Docker/K8s secrets
    )


@lru_cache
def get_settings() -> Settings:            # кешированный синглтон
    return Settings()


# --- Использование в эндпоинте ---
async def handler(settings: Annotated[Settings, Depends(get_settings)]):
    ...

# --- Приоритет источников (выше = важнее) ---
# init args > env vars > .env > secrets_dir > defaults

# --- Переопределение в тестах ---
app.dependency_overrides[get_settings] = lambda: Settings(admin_email="t@t.io", db_password="x")
# ... тест ...
app.dependency_overrides.clear()

# или
monkeypatch.setenv("ADMIN_EMAIL", "t@t.io"); get_settings.cache_clear()

# --- Секрет ---
settings.db_password                       # SecretStr('**********')
settings.db_password.get_secret_value()    # реальное значение
```

Краткий вывод: используйте `pydantic-settings` (`BaseSettings` + `SettingsConfigDict`) как единый типизированный источник конфигурации, читайте из env/`.env`, отдавайте через `get_settings()` с `@lru_cache`, прячьте секреты в `SecretStr`/`secrets_dir`, переопределяйте в тестах через `dependency_overrides`. Следуйте 12-factor: конфиг — в окружении, не в коде.
