# FastAPI — Безопасность и аутентификация — конспект и вопросы

## О чём раздел

Безопасность в веб-приложениях — это всегда комбинация нескольких связанных, но разных понятий. На собеседовании по FastAPI часто проверяют, понимаете ли вы разницу между ними и умеете ли реализовать рабочий механизм входа.

- **Аутентификация (Authentication)** — кто ты? Проверка личности: пользователь предъявляет логин/пароль, токен, сертификат и т. д.
- **Авторизация (Authorization)** — что тебе можно? Проверка прав: у пользователя есть личность, но есть ли у него доступ к конкретному ресурсу/действию (роли, scopes).
- **OAuth2** — спецификация (фреймворк) того, как происходит выдача и использование токенов доступа. Сам пароль (если он используется) проверяется один раз, дальше клиент ходит с токеном.
- **OpenID Connect (OIDC)** — надстройка над OAuth2, добавляющая стандартизированную аутентификацию (ID-токен). Используется Google, Microsoft и т. д.
- **OpenAPI security schemes** — описание в OpenAPI того, какими способами защищён API (`apiKey`, `http`, `oauth2`, `openIdConnect`). FastAPI генерирует это автоматически, благодаря чему Swagger UI показывает кнопку **Authorize**.

FastAPI предоставляет инструменты в модуле `fastapi.security`, которые:
1. реализуют стандартные схемы (`OAuth2PasswordBearer`, `OAuth2PasswordRequestForm`, `HTTPBearer`, `APIKeyHeader` и т. д.);
2. автоматически интегрируются с OpenAPI/Swagger UI;
3. работают как обычные зависимости (`Depends`/`Security`), поэтому легко комбинируются и переиспользуются.

> Важно: FastAPI даёт **инструменты**, но не навязывает БД, способ хранения пользователей или конкретную реализацию. Хеширование, JWT, refresh-токены вы собираете сами из проверенных библиотек.

---

## Ключевые концепции

| Понятие | Что это | Где в FastAPI |
|---|---|---|
| OAuth2 Password Flow | Поток, где клиент отправляет `username`+`password` и получает токен | `OAuth2PasswordRequestForm`, `OAuth2PasswordBearer` |
| Bearer-токен | Токен в заголовке `Authorization: Bearer <token>` | `OAuth2PasswordBearer` извлекает его |
| `tokenUrl` | URL эндпоинта логина (для документации/Swagger) | параметр `OAuth2PasswordBearer(tokenUrl=...)` |
| JWT | Самодостаточный подписанный токен (header.payload.signature) | библиотека `PyJWT` (или `python-jose`) |
| Хеширование пароля | Необратимое преобразование пароля для хранения | `passlib[bcrypt]` или `pwdlib` |
| `Depends()` | Внедрение зависимости | получение текущего пользователя |
| `Security()` | То же, что `Depends`, но с поддержкой scopes | OAuth2 scopes |
| scopes | Гранулярные права внутри токена | `SecurityScopes`, `Security(dep, scopes=[...])` |

**Поток OAuth2 Password (упрощённо):**

```
1. POST /token  (username, password)  ->  сервер проверяет пароль (по хешу)
2. сервер создаёт JWT (sub=username, exp=...)  ->  отдаёт {access_token, token_type: "bearer"}
3. клиент хранит токен и шлёт его: Authorization: Bearer <token>
4. на защищённом эндпоинте зависимость декодирует токен, достаёт пользователя
5. если токен валиден -> доступ; иначе -> 401
```

---

## Подробный разбор с примерами кода

### 1. First Steps — `OAuth2PasswordBearer` и `tokenUrl`

`OAuth2PasswordBearer` — класс-зависимость, который ожидает токен в заголовке `Authorization: Bearer <token>`. Если токена нет — возвращает `401`.

```python
from typing import Annotated

from fastapi import Depends, FastAPI
from fastapi.security import OAuth2PasswordBearer

app = FastAPI()

# tokenUrl — это ОТНОСИТЕЛЬНЫЙ URL эндпоинта, который выдаёт токен.
# Он НЕ создаёт эндпоинт автоматически — это лишь метаданные для OpenAPI/Swagger UI,
# чтобы кнопка Authorize знала, куда отправлять username/password.
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")


@app.get("/items/")
async def read_items(token: Annotated[str, Depends(oauth2_scheme)]) -> dict:
    # Сюда мы попадаем, только если в запросе был заголовок Authorization: Bearer ...
    # Сейчас token — это просто строка; валидацию добавим ниже.
    return {"token": token}
```

Что происходит в Swagger UI (`/docs`):
- появляется кнопка **Authorize**;
- благодаря `tokenUrl="token"` Swagger знает, что для получения токена надо обратиться на `POST /token` с формой `username`/`password`;
- после ввода данных Swagger сам подставляет `Authorization: Bearer <token>` во все последующие запросы.

> `tokenUrl="token"` указывает на эндпоинт `/token`, который вы должны реализовать сами (см. ниже). Свежий `OAuth2PasswordBearer` не делает магии — он только читает заголовок и описывает схему в OpenAPI.

### 2. Получение текущего пользователя — `get_current_user`

Превращаем строку-токен в объект пользователя. Делаем это через цепочку зависимостей: `oauth2_scheme` -> `get_current_user` -> эндпоинт.

```python
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from pydantic import BaseModel

app = FastAPI()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")


# Pydantic v2 модель пользователя (то, что мы отдаём наружу — без пароля)
class User(BaseModel):
    username: str
    email: str | None = None
    full_name: str | None = None
    disabled: bool | None = None


# Заглушка декодирования токена (в реальности — JWT, см. ниже)
def fake_decode_token(token: str) -> User:
    return User(username=token + "fakedecoded", email="a@b.c")


async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
) -> User:
    user = fake_decode_token(token)
    if not user:
        # Стандарт OAuth2 требует именно заголовок WWW-Authenticate: Bearer
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Невалидные учётные данные",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return user


# Часто делают вторую зависимость поверх первой — проверку "не заблокирован ли"
async def get_current_active_user(
    current_user: Annotated[User, Depends(get_current_user)],
) -> User:
    if current_user.disabled:
        raise HTTPException(status_code=400, detail="Пользователь заблокирован")
    return current_user


@app.get("/users/me")
async def read_users_me(
    current_user: Annotated[User, Depends(get_current_active_user)],
) -> User:
    return current_user
```

Идея «двух зависимостей» (`get_current_user` + `get_current_active_user`) удобна: первая отвечает за разбор токена, вторая — за бизнес-правила (блокировка, роли). Их можно переиспользовать в разных эндпоинтах.

### 3. Хеширование паролей

**Почему нельзя хранить пароли в открытом виде:** при утечке БД злоумышленник получит все пароли. Люди часто переиспользуют пароли, поэтому утечка одного сервиса компрометирует другие. Поэтому пароль никогда не хранится «как есть» — хранится его **хеш** (необратимое преобразование). При логине мы хешируем введённый пароль и сравниваем хеши.

bcrypt дополнительно:
- использует **соль** (salt) — случайную добавку, чтобы одинаковые пароли давали разные хеши (защита от радужных таблиц);
- **намеренно медленный** (cost factor), чтобы перебор был дорогим.

Вариант на `passlib[bcrypt]` (классика):

```python
from passlib.context import CryptContext

# schemes — список алгоритмов; bcrypt — рекомендуемый
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(password: str) -> str:
    return pwd_context.hash(password)  # внутри генерируется соль


def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)
```

Современный вариант на `pwdlib` (рекомендуется в свежей документации FastAPI, нет проблем совместимости со старыми версиями bcrypt):

```python
from pwdlib import PasswordHash

password_hash = PasswordHash.recommended()  # bcrypt/argon2 под капотом


def hash_password(password: str) -> str:
    return password_hash.hash(password)


def verify_password(plain: str, hashed: str) -> bool:
    return password_hash.verify(plain, hashed)
```

> Установка: `pip install "passlib[bcrypt]"` или `pip install "pwdlib[bcrypt]"`.
> bcrypt имеет ограничение в **72 байта** на длину пароля — длинные пароли молча обрезаются.

### 4. Simple OAuth2 с Password и Bearer — `OAuth2PasswordRequestForm`

`OAuth2PasswordRequestForm` — готовая зависимость, читающая **данные формы** (не JSON!) по спецификации OAuth2:
- `username` (обязательно),
- `password` (обязательно),
- `scope` (строка через пробел, опционально),
- `grant_type`, `client_id`, `client_secret` (опционально).

Эндпоинт логина должен вернуть JSON `{"access_token": ..., "token_type": "bearer"}`.

```python
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm

app = FastAPI()

# Псевдо-БД: пароль "secret" уже захеширован bcrypt
fake_users_db = {
    "johndoe": {
        "username": "johndoe",
        "full_name": "John Doe",
        "email": "johndoe@example.com",
        "hashed_password": "$2b$12$EixZaYVK1fsbw1ZfbX3OXePaWxn96p36WQoeG6Lruj3vjPGga31lW",
        "disabled": False,
    }
}


@app.post("/token")
async def login(
    form_data: Annotated[OAuth2PasswordRequestForm, Depends()],
) -> dict:
    user = fake_users_db.get(form_data.username)
    if not user or not verify_password(form_data.password, user["hashed_password"]):
        # ВАЖНО: одинаковая ошибка для "нет пользователя" и "неверный пароль",
        # чтобы не раскрывать существование логина (user enumeration).
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Неверный логин или пароль",
        )
    # В простом примере токен = username (так делать в проде НЕЛЬЗЯ — см. JWT ниже)
    return {"access_token": user["username"], "token_type": "bearer"}
```

### 5. JWT (JSON Web Token)

**JWT** — самодостаточный токен из трёх частей, разделённых точками: `header.payload.signature`.
- **header** — алгоритм и тип (`{"alg": "HS256", "typ": "JWT"}`);
- **payload** — данные (claims): `sub` (subject — кого касается токен), `exp` (время истечения), любые свои поля;
- **signature** — подпись секретом. Гарантирует, что payload не подменили.

> JWT **не шифруется**, а только **подписывается**. payload легко декодируется любым (base64). Никогда не кладите в JWT секреты/пароли. Подпись лишь защищает от подделки.

Алгоритмы:
- **HS256** — симметричный (один секрет и подписывает, и проверяет). Простой, подходит для монолита.
- **RS256** — асимметричный (приватный ключ подписывает, публичный проверяет). Удобен, когда несколько сервисов проверяют токен.

Создание и проверка токена через `PyJWT`:

```python
from datetime import datetime, timedelta, timezone

import jwt  # пакет PyJWT:  pip install pyjwt
from jwt.exceptions import InvalidTokenError

# Секрет генерируют так:  openssl rand -hex 32
# В проде он живёт в переменной окружения, НЕ в коде!
SECRET_KEY = "09d25e094faa6ca2556c818166b7a9563b93f7099f6f0f4caa6cf63b88e8d3e7"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30


def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + (
        expires_delta or timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    )
    to_encode.update({"exp": expire})  # claim exp проверяется автоматически при декодировании
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


def decode_access_token(token: str) -> dict:
    # jwt.decode сам проверит подпись и exp; при истечении кинет ExpiredSignatureError
    return jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
```

Обработка истечения и невалидности:

```python
async def get_current_user(token: Annotated[str, Depends(oauth2_scheme)]) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Невалидные учётные данные",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username = payload.get("sub")
        if username is None:
            raise credentials_exception
    except InvalidTokenError:
        # InvalidTokenError — базовый класс, ловит и ExpiredSignatureError, и битую подпись
        raise credentials_exception
    user = get_user(fake_users_db, username=username)
    if user is None:
        raise credentials_exception
    return user
```

> `python-jose` — альтернатива PyJWT, исторически часто встречается в старых туториалах FastAPI. Сейчас официальная документация перешла на **PyJWT**. API похож: `jose.jwt.encode/decode`.

### 6. `Security()` vs `Depends()` и scopes (кратко)

`Security()` — это специализация `Depends()`. Функционально для простого случая они эквивалентны, но `Security()` умеет работать со **scopes** (правами внутри OAuth2-токена) и корректно отражать их в OpenAPI.

```python
from fastapi import Security
from fastapi.security import SecurityScopes

oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="token",
    scopes={"items": "Чтение items", "users": "Управление пользователями"},
)


async def get_current_user(
    security_scopes: SecurityScopes,
    token: Annotated[str, Depends(oauth2_scheme)],
) -> User:
    # ... декодируем токен, читаем payload["scopes"]
    # для каждого scope из security_scopes.scopes проверяем, что он есть в токене
    ...


@app.get("/users/me/items/")
async def read_own_items(
    # Требуем конкретный scope "items"
    current_user: Annotated[User, Security(get_current_active_user, scopes=["items"])],
):
    ...
```

Полный механизм scopes — отдельная большая тема (см. advanced-раздел документации: <https://fastapi.tiangolo.com/advanced/security/oauth2-scopes/>). На собеседовании достаточно знать: scopes = гранулярные права, кладутся в JWT, проверяются через `SecurityScopes` + `Security(...)`.

---

## Полный рабочий пример (логин с JWT + защищённый эндпоинт)

Самодостаточный файл. Зависимости: `fastapi`, `uvicorn`, `pyjwt`, `passlib[bcrypt]`, `python-multipart` (для форм).

```python
from datetime import datetime, timedelta, timezone
from typing import Annotated

import jwt
from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jwt.exceptions import InvalidTokenError
from passlib.context import CryptContext
from pydantic import BaseModel

# --- Конфигурация (в проде брать из окружения!) ---
SECRET_KEY = "09d25e094faa6ca2556c818166b7a9563b93f7099f6f0f4caa6cf63b88e8d3e7"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# --- Псевдо-БД: пароль "secret" ---
fake_users_db = {
    "johndoe": {
        "username": "johndoe",
        "full_name": "John Doe",
        "email": "johndoe@example.com",
        "hashed_password": "$2b$12$EixZaYVK1fsbw1ZfbX3OXePaWxn96p36WQoeG6Lruj3vjPGga31lW",
        "disabled": False,
    }
}

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")
app = FastAPI()


# --- Pydantic v2 модели ---
class Token(BaseModel):
    access_token: str
    token_type: str


class TokenData(BaseModel):
    username: str | None = None


class User(BaseModel):
    username: str
    email: str | None = None
    full_name: str | None = None
    disabled: bool | None = None


class UserInDB(User):
    hashed_password: str  # хеш храним только во внутренней модели


# --- Утилиты ---
def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)


def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)


def get_user(db: dict, username: str) -> UserInDB | None:
    if username in db:
        return UserInDB(**db[username])
    return None


def authenticate_user(db: dict, username: str, password: str) -> UserInDB | None:
    user = get_user(db, username)
    if not user or not verify_password(password, user.hashed_password):
        return None
    return user


def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + (
        expires_delta or timedelta(minutes=15)
    )
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


# --- Зависимости аутентификации ---
async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
) -> UserInDB:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Невалидные учётные данные",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username = payload.get("sub")
        if username is None:
            raise credentials_exception
        token_data = TokenData(username=username)
    except InvalidTokenError:
        raise credentials_exception
    user = get_user(fake_users_db, username=token_data.username)
    if user is None:
        raise credentials_exception
    return user


async def get_current_active_user(
    current_user: Annotated[UserInDB, Depends(get_current_user)],
) -> UserInDB:
    if current_user.disabled:
        raise HTTPException(status_code=400, detail="Пользователь заблокирован")
    return current_user


# --- Эндпоинт логина ---
@app.post("/token")
async def login_for_access_token(
    form_data: Annotated[OAuth2PasswordRequestForm, Depends()],
) -> Token:
    user = authenticate_user(fake_users_db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Неверный логин или пароль",
            headers={"WWW-Authenticate": "Bearer"},
        )
    access_token = create_access_token(
        data={"sub": user.username},
        expires_delta=timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),
    )
    return Token(access_token=access_token, token_type="bearer")


# --- Защищённые эндпоинты ---
@app.get("/users/me/")
async def read_users_me(
    current_user: Annotated[User, Depends(get_current_active_user)],
) -> User:
    return current_user


@app.get("/users/me/items/")
async def read_own_items(
    current_user: Annotated[User, Depends(get_current_active_user)],
) -> list[dict]:
    return [{"item_id": "Foo", "owner": current_user.username}]
```

Проверка через curl:

```bash
# 1. Получить токен (данные формы, не JSON!)
curl -X POST http://localhost:8000/token \
  -d "username=johndoe&password=secret"
# -> {"access_token":"eyJ...","token_type":"bearer"}

# 2. Запрос к защищённому эндпоинту
curl http://localhost:8000/users/me/ \
  -H "Authorization: Bearer eyJ..."
```

---

## Частые вопросы на собеседовании (Q/A)

**1. В чём разница между аутентификацией и авторизацией?**
Аутентификация — проверка личности («кто ты»: логин/пароль, токен). Авторизация — проверка прав («что тебе можно»: роли, scopes). Сначала аутентифицируем, потом авторизуем.

**2. Что такое OAuth2 и при чём тут пароль?**
OAuth2 — фреймворк выдачи и использования токенов доступа. В **Password flow** клиент один раз отправляет логин/пароль и получает токен, далее ходит только с токеном (`Authorization: Bearer ...`). Пароль больше не передаётся.

**3. Зачем нужен `tokenUrl` в `OAuth2PasswordBearer` и создаёт ли он эндпоинт?**
Нет, он эндпоинт не создаёт. `tokenUrl` — это метаданные для OpenAPI/Swagger UI: по нему кнопка **Authorize** знает, куда слать username/password. Сам эндпоинт `/token` вы реализуете руками.

**4. Как Swagger UI узнаёт про авторизацию и откуда берётся кнопка Authorize?**
FastAPI генерирует OpenAPI security scheme из зависимости `OAuth2PasswordBearer`/`OAuth2PasswordRequestForm`. Swagger читает схему, рисует кнопку Authorize и затем сам добавляет заголовок `Authorization: Bearer <token>` в запросы.

**5. Почему `OAuth2PasswordRequestForm` использует форму, а не JSON?**
Так требует спецификация OAuth2: поля `username`, `password`, `grant_type`, `scope` передаются как `application/x-www-form-urlencoded`. Поэтому для логина нужен `python-multipart`.

**6. Какие поля есть у `OAuth2PasswordRequestForm`?**
`username`, `password` (обязательные), `scope` (строка scope через пробел), `grant_type`, `client_id`, `client_secret` (опциональные).

**7. Почему пароли нельзя хранить в открытом виде и что такое хеширование с солью?**
При утечке БД открытые пароли скомпрометированы напрямую (а люди их переиспользуют). Хранят необратимый **хеш**. **Соль** — случайная добавка к паролю, чтобы одинаковые пароли давали разные хеши (защита от радужных таблиц). bcrypt намеренно медленный — это удорожает перебор.

**8. Чем bcrypt лучше, чем SHA-256, для паролей?**
SHA-256 быстрый — это плохо для паролей (быстрый перебор). bcrypt/argon2 медленные и солёные by design, специально спроектированы под хеширование паролей.

**9. Что такое JWT и из каких частей он состоит?**
JSON Web Token: `header.payload.signature` (base64url). Header — алгоритм/тип; payload — claims (`sub`, `exp`, свои поля); signature — подпись секретом, защищает от подделки.

**10. JWT шифруется или подписывается? Можно ли класть в него пароль?**
По умолчанию JWT **подписывается, но не шифруется**: payload читается любым через base64. Класть пароли/секреты нельзя. Подпись лишь гарантирует целостность.

**11. Что такое claims `sub` и `exp`?**
`sub` (subject) — идентификатор субъекта токена (обычно username/user_id). `exp` (expiration) — время истечения (Unix timestamp); библиотека сама проверяет его при декодировании и кидает ошибку, если токен просрочен.

**12. В чём разница HS256 и RS256?**
HS256 — симметричный: один секрет и для подписи, и для проверки (удобно в монолите). RS256 — асимметричный: приватный ключ подписывает, публичный проверяет (удобно, когда токен проверяют несколько сервисов).

**13. Как обработать истёкший/невалидный токен?**
В `jwt.decode` оборачиваем в `try/except InvalidTokenError` (это базовый класс, ловит и `ExpiredSignatureError`, и битую подпись) и кидаем `HTTPException(401, headers={"WWW-Authenticate": "Bearer"})`.

**14. Зачем заголовок `WWW-Authenticate: Bearer` в ответе 401?**
Этого требует спецификация OAuth2/HTTP для схемы Bearer: он сообщает клиенту, каким способом нужно аутентифицироваться. FastAPI-документация добавляет его во все 401 при Bearer.

**15. Чем `Security()` отличается от `Depends()`?**
`Security()` — это `Depends()` с поддержкой **scopes** и корректным отражением их в OpenAPI. Для простой защиты без scopes они эквивалентны; для OAuth2 scopes нужен `Security(dep, scopes=[...])`.

**16. Что такое scopes и как они реализованы в FastAPI?**
Scopes — гранулярные права (например `items:read`). Их кладут в JWT (`scopes`), объявляют в `OAuth2PasswordBearer(scopes={...})`, проверяют через спец. параметр `SecurityScopes` в зависимости и требуют через `Security(dep, scopes=[...])`.

**17. Почему при неверном логине и при неверном пароле возвращают одинаковую ошибку?**
Чтобы не раскрывать существование пользователя (user enumeration). Разные сообщения позволили бы перебором узнать валидные логины.

**18. Зачем нужны refresh-токены?**
Access-токен делают коротким (минуты) для безопасности. Чтобы пользователь не логинился каждые 30 минут, выдают долгоживущий **refresh-токен**, по которому можно получить новый access-токен. Refresh-токен хранится надёжнее (например, в httpOnly cookie) и может быть отозван.

**19. Можно ли «разлогинить» JWT / отозвать его?**
Сам по себе JWT stateless — отозвать его до истечения нельзя. Решения: короткий `exp`; чёрный список (blacklist) токенов/jti в Redis; версия токена в БД; ротация refresh-токенов. Это компромисс «stateless vs возможность отзыва».

**20. Где хранить секретный ключ и токены?**
Секрет — в переменных окружения / секрет-менеджере, не в коде и не в git. Токен на клиенте: для веба предпочтительно httpOnly + Secure cookie (защита от XSS) либо аккуратное хранение; localStorage уязвим к XSS.

---

## Подводные камни (gotchas)

- **Забыли `python-multipart`.** `OAuth2PasswordRequestForm` читает данные формы. Без `python-multipart` логин упадёт с ошибкой о форме.
- **Шлют логин как JSON.** `/token` ожидает `application/x-www-form-urlencoded`, а не JSON. Частая ошибка в Postman/фронтенде.
- **`tokenUrl` не совпадает с реальным путём.** Если эндпоинт `/api/v1/token`, а в `OAuth2PasswordBearer(tokenUrl="token")`, кнопка Authorize в Swagger будет слать не туда.
- **Секрет в коде/гите.** `SECRET_KEY` обязан жить в окружении. Утечка секрета HS256 = возможность подделать любой токен.
- **Хранят пароль или сам пароль в JWT.** Никогда. В payload только идентификаторы и неконфиденциальные данные.
- **`datetime.utcnow()` устарел / наивные даты.** Используйте `datetime.now(timezone.utc)` (aware). Наивные времена в `exp` приводят к багам со сравнением.
- **Слишком долгий `exp`.** Долгоживущий access-токен невозможно отозвать — компрометация надолго. Делайте короткий access + refresh.
- **Ловят только `ExpiredSignatureError`.** Лучше ловить базовый `InvalidTokenError`, иначе битая подпись «прорвётся» наружу как 500.
- **Ошибка раскрывает существование пользователя.** Разные сообщения для «нет логина» и «неверный пароль» — это user enumeration. Делайте одинаковую ошибку.
- **Путают `Depends` и `Security` при scopes.** Scopes работают только через `Security(...)`.
- **bcrypt 72 байта.** Пароли длиннее 72 байт молча обрезаются — учитывайте при «парольных фразах».
- **Возвращают `hashed_password` наружу.** Используйте отдельные модели `UserInDB` (с хешем) и `User` (ответ без хеша).
- **Нет HTTPS.** Bearer-токен в открытом HTTP перехватывается. Всегда TLS в проде.

---

## Лучшие практики

1. **Только HTTPS/TLS в проде.** Bearer-токены передаются в заголовке открытым текстом — без TLS их перехватят.
2. **Секреты — в переменных окружения / секрет-менеджере.** `SECRET_KEY`, ключи БД, ключи провайдеров. Никогда не в коде/гите. Используйте `pydantic-settings` для типобезопасной загрузки.
3. **Короткий access-токен + refresh-токен.** Access — минуты (5–30), refresh — дни/недели, хранится надёжно (httpOnly+Secure cookie), может быть отозван/ротирован.
4. **Хешируйте пароли bcrypt/argon2** (`passlib[bcrypt]` или `pwdlib`), с солью (она встроена). Никогда не SHA/MD5 без растяжки.
5. **Разделяйте модели:** `UserInDB` (с `hashed_password`) и `User`/ответ (без секретов). `response_model` для гарантии, что хеш не утечёт.
6. **Цепочка зависимостей:** `oauth2_scheme` -> `get_current_user` -> `get_current_active_user`. Бизнес-проверки (блокировка, роли) — в отдельной зависимости.
7. **Одинаковые ошибки** при неверном логине/пароле — против user enumeration. Заголовок `WWW-Authenticate: Bearer` в 401.
8. **Ловите `InvalidTokenError`** (базовый класс), а не только `ExpiredSignatureError`.
9. **`datetime.now(timezone.utc)`** для `exp` — timezone-aware время.
10. **CORS аккуратно.** Настраивайте `CORSMiddleware` явными origins, не `allow_origins=["*"]` вместе с `allow_credentials=True` (браузер это запретит). Указывайте конкретные домены фронтенда.
11. **Rate limiting** на эндпоинте логина — против брутфорса (например, slowapi / на уровне reverse-proxy).
12. **Scopes/роли** для авторизации — гранулярные права через `Security(dep, scopes=[...])`.
13. **Не изобретайте крипту.** Используйте PyJWT/passlib/pwdlib, проверенные библиотеки.

---

## Шпаргалка

```python
# Установка
# pip install fastapi uvicorn pyjwt "passlib[bcrypt]" python-multipart pydantic-settings

# --- Импорты ---
from typing import Annotated
from datetime import datetime, timedelta, timezone
import jwt
from jwt.exceptions import InvalidTokenError
from fastapi import Depends, FastAPI, HTTPException, status, Security
from fastapi.security import (
    OAuth2PasswordBearer,        # читает Authorization: Bearer <token>
    OAuth2PasswordRequestForm,   # форма username/password/scope для /token
    SecurityScopes,              # проверка scopes
)
from passlib.context import CryptContext

# --- Хеширование ---
pwd = CryptContext(schemes=["bcrypt"], deprecated="auto")
pwd.hash("plain")             # -> хеш с солью
pwd.verify("plain", hashed)   # -> bool

# --- JWT ---
SECRET = "..."; ALG = "HS256"           # секрет из окружения!
token = jwt.encode({"sub": "u", "exp": datetime.now(timezone.utc) + timedelta(minutes=30)},
                   SECRET, algorithm=ALG)
payload = jwt.decode(token, SECRET, algorithms=[ALG])  # сам проверит подпись и exp

# --- Схема Bearer + tokenUrl ---
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")  # tokenUrl = метаданные для Swagger

# --- Логин (форма!) ---
# @app.post("/token")
# async def login(form: Annotated[OAuth2PasswordRequestForm, Depends()]) -> Token: ...
# return {"access_token": jwt_token, "token_type": "bearer"}

# --- Защита эндпоинта ---
# async def get_current_user(token: Annotated[str, Depends(oauth2_scheme)]) -> User: ...
# @app.get("/me")
# async def me(user: Annotated[User, Depends(get_current_active_user)]): ...

# --- 401 по стандарту ---
HTTPException(status_code=401, detail="...", headers={"WWW-Authenticate": "Bearer"})

# --- scopes ---
# Security(get_current_active_user, scopes=["items"])  # вместо Depends, когда нужны права
```

Ключевые тезисы одной строкой:
- OAuth2 Password flow: `/token` (форма) -> JWT -> `Authorization: Bearer` -> `Depends(get_current_user)`.
- JWT подписан, но не зашифрован; `sub` + `exp`; HS256 (симметрично) / RS256 (асимметрично).
- Пароли — bcrypt/argon2 с солью; секрет — в окружении; access короткий + refresh.
- `Security()` = `Depends()` + scopes; одинаковые ошибки логина; HTTPS обязателен.

Ссылки: <https://fastapi.tiangolo.com/tutorial/security/> · scopes: <https://fastapi.tiangolo.com/advanced/security/oauth2-scopes/>
```
