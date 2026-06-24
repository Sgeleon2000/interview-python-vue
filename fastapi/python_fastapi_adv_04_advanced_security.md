# FastAPI Advanced — Продвинутая безопасность — конспект и вопросы

## О чём раздел

Раздел расширяет базовую тему безопасности FastAPI. Базовый уровень — это OAuth2 с паролем и Bearer-токеном (JWT) плюс простая зависимость `get_current_user`. Продвинутый уровень добавляет:

- **OAuth2 scopes (области доступа)** — тонкая авторизация: токен несёт список разрешений, а каждый эндпоинт требует определённый набор скоупов. Используются `Security()` вместо `Depends()`, специальный объект `SecurityScopes` и заголовок `WWW-Authenticate` с информацией о требуемых правах.
- **HTTP Basic Auth** — простейшая схема "логин:пароль" в заголовке `Authorization: Basic`, с обязательной защитой от timing-атак через `secrets.compare_digest`.
- **Аутентификация через cookies** — когда токен/сессия хранятся в HttpOnly cookie, а не в заголовке (актуально для браузерных приложений и защиты от XSS-кражи токена).
- Общие замечания о безопасности: HTTPS, хеширование паролей, защита от утечки информации через различия в ответах.

## Ключевые концепции

1. **`Security()` vs `Depends()`.** `Security()` — это специализированная версия `Depends()`, которая дополнительно умеет объявлять требуемые **scopes**. Под капотом FastAPI собирает все scopes из дерева зависимостей и отражает их в OpenAPI.

2. **Scopes (области доступа).** Строки-разрешения (`"items:read"`, `"users:me"`, `"admin"`). Токен (JWT) содержит выданные пользователю scopes (обычно в claim `scope` — строка через пробел). Эндпоинт объявляет требуемые scopes; если их нет в токене — 401/403.

3. **`SecurityScopes`.** Специальный объект, который FastAPI инъектирует в зависимость. Содержит `scopes` (список требуемых для данного маршрута) и `scope_str` (та же строка через пробел для заголовка `WWW-Authenticate`). Позволяет в одной зависимости проверять разные наборы прав в зависимости от маршрута.

4. **Иерархия / агрегация скоупов.** Требуемые scopes суммируются по всей цепочке зависимостей: если внешняя зависимость требует `users:me`, а под-зависимость — `items:read`, эндпоинт потребует оба.

5. **Отражение в Swagger UI.** Объявленные scopes показываются в форме авторизации Swagger UI как чекбоксы, и пользователь выбирает, какие запросить.

6. **HTTP Basic.** `HTTPBasic` security scheme. Браузер показывает нативное окно логина. Креды приходят в каждом запросе. Сравнение логина/пароля — только через `secrets.compare_digest` (постоянное время) во избежание timing-атак.

7. **Cookies для аутентификации.** Токен в HttpOnly cookie не доступен JS (защита от XSS), но требует защиты от CSRF. Можно прочитать через `Cookie(...)` или кастомную схему безопасности.

## Подробный разбор с примерами кода

### 1. OAuth2 scopes: объявление схемы

```python
from fastapi.security import OAuth2PasswordBearer

# Объявляем доступные scopes прямо в схеме — они попадут в Swagger UI
oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="token",
    scopes={
        "me": "Чтение информации о текущем пользователе.",
        "items": "Чтение элементов (items).",
        "admin": "Полный административный доступ.",
    },
)
```

### 2. `SecurityScopes` в зависимости проверки пользователя

```python
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, Security, status
from fastapi.security import SecurityScopes
from jose import JWTError, jwt
from pydantic import BaseModel, ValidationError

SECRET_KEY = "CHANGE_ME"
ALGORITHM = "HS256"


class TokenData(BaseModel):
    username: str | None = None
    scopes: list[str] = []


async def get_current_user(
    security_scopes: SecurityScopes,
    token: Annotated[str, Depends(oauth2_scheme)],
) -> "User":
    # Формируем заголовок WWW-Authenticate с учётом требуемых scopes
    if security_scopes.scopes:
        authenticate_value = f'Bearer scope="{security_scopes.scope_str}"'
    else:
        authenticate_value = "Bearer"

    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Не удалось проверить учётные данные",
        headers={"WWW-Authenticate": authenticate_value},
    )

    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str | None = payload.get("sub")
        if username is None:
            raise credentials_exception
        # scope-claim хранится строкой через пробел
        token_scopes = payload.get("scopes", [])
        token_data = TokenData(scopes=token_scopes, username=username)
    except (JWTError, ValidationError):
        raise credentials_exception

    user = get_user(token_data.username)  # ваша функция загрузки пользователя
    if user is None:
        raise credentials_exception

    # Проверяем, что у токена есть ВСЕ требуемые маршрутом scopes
    for scope in security_scopes.scopes:
        if scope not in token_data.scopes:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Недостаточно прав (scope)",
                headers={"WWW-Authenticate": authenticate_value},
            )
    return user
```

Ключевые моменты:

- `security_scopes.scopes` — список требуемых **для конкретного маршрута** scopes (зависит от того, как объявлен `Security(...)` в эндпоинте).
- При нехватке прав по стандарту OAuth2 возвращают `401` с заголовком `WWW-Authenticate`, описывающим нужный scope (некоторые используют `403` для "аутентифицирован, но не авторизован" — оба подхода встречаются; документация FastAPI использует 401).

### 3. Объявление требуемых scopes на эндпоинте через `Security()`

```python
@app.get("/users/me/")
async def read_users_me(
    # Требуем scope "me"
    current_user: Annotated["User", Security(get_current_user, scopes=["me"])],
):
    return current_user


@app.get("/users/me/items/")
async def read_own_items(
    # Требуем сразу два scope
    current_user: Annotated[
        "User", Security(get_current_user, scopes=["me", "items"])
    ],
):
    return [{"item_id": "Foo", "owner": current_user.username}]
```

### 4. Агрегация скоупов через цепочку зависимостей

Scopes суммируются вверх по дереву. Если одна зависимость требует `me`, а зависящая от неё — `items`, итог потребует оба.

```python
async def get_current_active_user(
    current_user: Annotated["User", Security(get_current_user, scopes=["me"])],
) -> "User":
    if current_user.disabled:
        raise HTTPException(status_code=400, detail="Неактивный пользователь")
    return current_user


@app.get("/status/")
async def read_system_status(
    # Здесь активный пользователь уже требует scope "me" (унаследовано)
    current_user: Annotated["User", Depends(get_current_active_user)],
):
    return {"status": "ok", "user": current_user.username}
```

### 5. Выдача scopes при логине

```python
from fastapi.security import OAuth2PasswordRequestForm


@app.post("/token")
async def login(
    form_data: Annotated[OAuth2PasswordRequestForm, Depends()],
):
    user = authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(status_code=400, detail="Неверный логин или пароль")

    # form_data.scopes — это запрошенные клиентом scopes (из формы).
    # Выдаём только те, что реально разрешены пользователю (фильтрация!).
    allowed = [s for s in form_data.scopes if s in user.allowed_scopes]
    access_token = create_access_token(
        data={"sub": user.username, "scopes": allowed}
    )
    return {"access_token": access_token, "token_type": "bearer"}
```

Важно: никогда не выдавайте scopes только потому, что клиент их запросил — всегда фильтруйте по реальным правам пользователя на сервере.

### 6. HTTP Basic Auth с защитой от timing-атак

```python
import secrets
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import HTTPBasic, HTTPBasicCredentials

app = FastAPI()
security = HTTPBasic()


def get_current_username(
    credentials: Annotated[HTTPBasicCredentials, Depends(security)],
) -> str:
    # Кодируем в bytes для сравнения постоянного времени
    current_username_bytes = credentials.username.encode("utf8")
    correct_username_bytes = b"alice"
    # compare_digest сравнивает за постоянное время -> защита от timing-атак
    is_correct_username = secrets.compare_digest(
        current_username_bytes, correct_username_bytes
    )

    current_password_bytes = credentials.password.encode("utf8")
    correct_password_bytes = b"super-secret"
    is_correct_password = secrets.compare_digest(
        current_password_bytes, correct_password_bytes
    )

    # ВАЖНО: проверяем оба условия ПОСЛЕ обоих сравнений,
    # чтобы не было раннего выхода и разницы во времени.
    if not (is_correct_username and is_correct_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Неверный логин или пароль",
            headers={"WWW-Authenticate": "Basic"},
        )
    return credentials.username


@app.get("/basic/")
async def read_current_user(
    username: Annotated[str, Depends(get_current_username)],
):
    return {"username": username}
```

Почему `compare_digest`, а не `==`:

- Обычное сравнение строк может завершаться раньше при первом несовпадающем символе. По времени ответа атакующий способен побайтно подобрать секрет (timing attack).
- `secrets.compare_digest` сравнивает за время, не зависящее от позиции расхождения.
- Дополнительно: сравниваем и логин, и пароль **всегда** (без короткого замыкания `and`), чтобы не утекала информация о том, верен ли логин.

### 7. Аутентификация через cookies

Вариант А — прочитать токен из cookie напрямую:

```python
from typing import Annotated

from fastapi import Cookie, Depends, FastAPI, HTTPException, status

app = FastAPI()


async def get_user_from_cookie(
    session_token: Annotated[str | None, Cookie()] = None,
) -> "User":
    if session_token is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Нет сессии",
        )
    user = lookup_session(session_token)  # ваша проверка сессии
    if user is None:
        raise HTTPException(status_code=401, detail="Сессия недействительна")
    return user


@app.post("/login")
async def login(response: "Response"):
    token = create_session(...)
    # HttpOnly -> недоступно JS (защита от XSS-кражи токена)
    # Secure   -> только по HTTPS
    # SameSite -> снижает риск CSRF
    response.set_cookie(
        key="session_token",
        value=token,
        httponly=True,
        secure=True,
        samesite="lax",
    )
    return {"status": "ok"}
```

Вариант Б — кастомная security-схема, читающая cookie (чтобы это отражалось в OpenAPI), наследуемая от `APIKeyCookie`:

```python
from fastapi.security import APIKeyCookie

cookie_scheme = APIKeyCookie(name="session_token")


async def get_user_via_scheme(
    token: Annotated[str, Depends(cookie_scheme)],
) -> "User":
    user = lookup_session(token)
    if user is None:
        raise HTTPException(status_code=401, detail="Сессия недействительна")
    return user
```

## Полный рабочий пример

```python
import secrets
from datetime import datetime, timedelta, timezone
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, Security, status
from fastapi.security import (
    HTTPBasic,
    HTTPBasicCredentials,
    OAuth2PasswordBearer,
    OAuth2PasswordRequestForm,
    SecurityScopes,
)
from jose import JWTError, jwt
from passlib.context import CryptContext
from pydantic import BaseModel, ValidationError

SECRET_KEY = "CHANGE_ME_TO_A_RANDOM_SECRET"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# --- Фейковая БД -------------------------------------------------------------
FAKE_USERS = {
    "alice": {
        "username": "alice",
        "hashed_password": pwd_context.hash("wonderland"),
        "disabled": False,
        "allowed_scopes": ["me", "items"],
    },
    "admin": {
        "username": "admin",
        "hashed_password": pwd_context.hash("admin"),
        "disabled": False,
        "allowed_scopes": ["me", "items", "admin"],
    },
}


# --- Модели ------------------------------------------------------------------
class User(BaseModel):
    username: str
    disabled: bool = False
    allowed_scopes: list[str] = []


class TokenData(BaseModel):
    username: str | None = None
    scopes: list[str] = []


# --- Схемы безопасности ------------------------------------------------------
oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="token",
    scopes={
        "me": "Доступ к данным своего профиля.",
        "items": "Доступ к своим элементам.",
        "admin": "Административный доступ.",
    },
)
basic_scheme = HTTPBasic()

app = FastAPI(title="Продвинутая безопасность")


# --- Вспомогательное ---------------------------------------------------------
def get_user(username: str) -> User | None:
    data = FAKE_USERS.get(username)
    return User(**data) if data else None


def authenticate_user(username: str, password: str) -> User | None:
    raw = FAKE_USERS.get(username)
    if not raw or not pwd_context.verify(password, raw["hashed_password"]):
        return None
    return User(**raw)


def create_access_token(data: dict) -> str:
    payload = data.copy()
    payload["exp"] = datetime.now(timezone.utc) + timedelta(
        minutes=ACCESS_TOKEN_EXPIRE_MINUTES
    )
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)


# --- OAuth2 scopes: проверка пользователя ------------------------------------
async def get_current_user(
    security_scopes: SecurityScopes,
    token: Annotated[str, Depends(oauth2_scheme)],
) -> User:
    authenticate_value = (
        f'Bearer scope="{security_scopes.scope_str}"'
        if security_scopes.scopes
        else "Bearer"
    )
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Не удалось проверить учётные данные",
        headers={"WWW-Authenticate": authenticate_value},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username = payload.get("sub")
        if username is None:
            raise credentials_exception
        token_data = TokenData(
            username=username, scopes=payload.get("scopes", [])
        )
    except (JWTError, ValidationError):
        raise credentials_exception

    user = get_user(token_data.username)
    if user is None:
        raise credentials_exception

    for scope in security_scopes.scopes:
        if scope not in token_data.scopes:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail=f"Недостаточно прав, требуется scope: {scope}",
                headers={"WWW-Authenticate": authenticate_value},
            )
    return user


async def get_current_active_user(
    current_user: Annotated[User, Security(get_current_user, scopes=["me"])],
) -> User:
    if current_user.disabled:
        raise HTTPException(status_code=400, detail="Неактивный пользователь")
    return current_user


# --- Эндпоинты OAuth2 --------------------------------------------------------
@app.post("/token")
async def login_for_access_token(
    form_data: Annotated[OAuth2PasswordRequestForm, Depends()],
):
    user = authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Неверный логин или пароль",
            headers={"WWW-Authenticate": "Bearer"},
        )
    # Выдаём только разрешённые пользователю scopes
    granted = [s for s in form_data.scopes if s in user.allowed_scopes]
    token = create_access_token({"sub": user.username, "scopes": granted})
    return {"access_token": token, "token_type": "bearer"}


@app.get("/users/me/")
async def read_users_me(
    current_user: Annotated[User, Depends(get_current_active_user)],
):
    return {"username": current_user.username}


@app.get("/users/me/items/")
async def read_own_items(
    current_user: Annotated[
        User, Security(get_current_active_user, scopes=["items"])
    ],
):
    return [{"item_id": "Foo", "owner": current_user.username}]


@app.get("/admin/")
async def admin_panel(
    current_user: Annotated[User, Security(get_current_user, scopes=["admin"])],
):
    return {"message": "Добро пожаловать в админку", "user": current_user.username}


# --- HTTP Basic --------------------------------------------------------------
def get_basic_username(
    credentials: Annotated[HTTPBasicCredentials, Depends(basic_scheme)],
) -> str:
    correct_user = secrets.compare_digest(
        credentials.username.encode("utf8"), b"alice"
    )
    correct_pass = secrets.compare_digest(
        credentials.password.encode("utf8"), b"wonderland"
    )
    if not (correct_user and correct_pass):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Неверный логин или пароль",
            headers={"WWW-Authenticate": "Basic"},
        )
    return credentials.username


@app.get("/basic-protected/")
async def basic_protected(
    username: Annotated[str, Depends(get_basic_username)],
):
    return {"username": username}
```

## Частые вопросы на собеседовании (Q/A)

**1. Чем `Security()` отличается от `Depends()`?**
`Security()` — подкласс `Depends()`, дополнительно умеющий объявлять `scopes`. Если scopes не нужны, можно использовать `Depends()`; для требования областей доступа — `Security(dep, scopes=[...])`.

**2. Что такое `SecurityScopes` и зачем он?**
Объект, инъектируемый в зависимость, содержащий список scopes, требуемых **конкретным маршрутом** (`.scopes`) и их строку через пробел (`.scope_str`). Позволяет одной зависимости-проверке гибко требовать разные права на разных эндпоинтах и формировать корректный заголовок `WWW-Authenticate`.

**3. Как scopes агрегируются по дереву зависимостей?**
FastAPI собирает scopes из всех `Security(...)` по всей цепочке. Если внешняя зависимость требует `me`, а под-зависимость `items`, итоговый набор в `security_scopes.scopes` будет содержать оба.

**4. Где хранятся scopes в JWT?**
Обычно в claim `scope` (строка через пробел по стандарту OAuth2) или в кастомном `scopes` (список). При логине scopes фильтруются по правам пользователя и кладутся в payload токена.

**5. Почему нельзя выдавать scopes, которые запросил клиент, как есть?**
Клиент может запросить любые scopes (включая `admin`). Сервер обязан выдать только пересечение запрошенных и реально разрешённых пользователю прав. Иначе — эскалация привилегий.

**6. Какой статус-код возвращать при нехватке прав — 401 или 403?**
Документация FastAPI и стандарт OAuth2 Bearer используют `401` с заголовком `WWW-Authenticate`, описывающим требуемый scope. Многие команды отдают `403` для "аутентифицирован, но не авторизован". Главное — последовательность и наличие `WWW-Authenticate` для 401.

**7. Зачем `secrets.compare_digest` в HTTP Basic?**
Для защиты от timing-атак: обычное `==` может завершаться раньше при первом несовпадении, и по времени ответа можно побайтно подобрать секрет. `compare_digest` сравнивает за постоянное время.

**8. Почему в Basic-проверке сравнивают и логин, и пароль всегда, без раннего выхода?**
Чтобы по различиям во времени/поведении нельзя было определить, существует ли пользователь и какая часть кредов неверна. Сначала вычисляем оба сравнения, затем проверяем общий результат.

**9. Какие минусы у HTTP Basic Auth?**
Креды (в base64, не шифрование!) пересылаются в каждом запросе, поэтому обязателен HTTPS. Нет встроенного logout, нет срока действия, плохо подходит для SPA. Подходит для внутренних/служебных API.

**10. В чём плюсы и риски аутентификации через cookies?**
Плюс: HttpOnly-cookie недоступна JS — защита от XSS-кражи токена; браузер шлёт её автоматически. Риск: уязвимость к CSRF — нужны `SameSite`, anti-CSRF токены, `Secure` (только HTTPS).

**11. Как scopes отражаются в Swagger UI?**
Scopes, объявленные в `OAuth2PasswordBearer(scopes={...})`, показываются как чекбоксы в форме авторизации Swagger UI; пользователь выбирает, какие scopes запросить при логине.

**12. Что класть в заголовок `WWW-Authenticate` при ошибке scope?**
`Bearer scope="<требуемые scopes через пробел>"` (берётся из `security_scopes.scope_str`). Это сообщает клиенту, какие права требуются.

## Подводные камни (gotchas)

- **`==` вместо `compare_digest`** в Basic Auth — уязвимость к timing-атакам. Всегда используйте `secrets.compare_digest`.
- **Ранний выход в проверке кредов** (`if user_ok and pass_ok` со short-circuit на отдельных `if`) — вычисляйте оба сравнения, потом проверяйте результат.
- **Выдача всех запрошенных scopes** при логине — эскалация привилегий. Фильтруйте по `user.allowed_scopes`.
- **Забыли `WWW-Authenticate`** в 401 — нарушение стандарта, Swagger UI / клиенты могут вести себя некорректно.
- **`Depends()` вместо `Security()`** там, где нужны scopes — scopes не будут собраны/отражены в OpenAPI.
- **Cookie без `HttpOnly`/`Secure`/`SameSite`** — токен доступен JS (XSS) или уязвим к CSRF / перехвату по HTTP.
- **Хранение паролей в открытом виде** — всегда хешируйте (bcrypt/argon2 через passlib).
- **Отсутствие HTTPS** — Basic и Bearer передают креды/токены, по HTTP они видны в открытом виде (base64 ≠ шифрование).
- **Неверный claim для scopes** — рассинхрон между тем, как scopes кладутся в токен и как читаются.

## Лучшие практики

- Только HTTPS в проде; для cookie — `Secure` + `HttpOnly` + `SameSite`.
- Хешируйте пароли (bcrypt/argon2), никогда не храните в открытом виде.
- В Basic Auth — `secrets.compare_digest`, обе проверки без раннего выхода.
- При логине выдавайте только пересечение запрошенных и разрешённых scopes.
- Используйте `Security(..., scopes=[...])` для авторизации, агрегируйте права через дерево зависимостей.
- Возвращайте корректный `WWW-Authenticate` с указанием требуемого scope.
- Задавайте короткий TTL access-токенам, используйте refresh-токены при необходимости.
- Документируйте scopes в `OAuth2PasswordBearer(scopes={...})` — это и UX в Swagger, и контракт.
- Для cookie-сессий добавляйте CSRF-защиту.

## Шпаргалка

```python
# OAuth2 scopes
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token", scopes={"me": "...", "items": "..."})

async def get_current_user(sec: SecurityScopes, token: Annotated[str, Depends(oauth2_scheme)]):
    for scope in sec.scopes:          # требуемые маршрутом scopes
        if scope not in token_scopes: # проверяем наличие в токене
            raise HTTPException(401, headers={"WWW-Authenticate": f'Bearer scope="{sec.scope_str}"'})

@app.get("/x")
async def x(user: Annotated[User, Security(get_current_user, scopes=["items"])]): ...

# Логин: фильтрация scopes
granted = [s for s in form_data.scopes if s in user.allowed_scopes]

# HTTP Basic + защита от timing-атак
security = HTTPBasic()
ok_u = secrets.compare_digest(cred.username.encode(), b"alice")
ok_p = secrets.compare_digest(cred.password.encode(), b"secret")
if not (ok_u and ok_p): raise HTTPException(401, headers={"WWW-Authenticate": "Basic"})

# Cookie-сессия (безопасные флаги)
response.set_cookie("session_token", token, httponly=True, secure=True, samesite="lax")
# Чтение: Annotated[str | None, Cookie()] или APIKeyCookie(name="session_token")
```
