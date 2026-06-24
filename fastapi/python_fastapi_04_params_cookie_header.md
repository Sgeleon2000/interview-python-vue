# FastAPI — Cookie и Header параметры — конспект и вопросы

## О чём раздел

Помимо path-, query- и body-параметров, FastAPI умеет извлекать данные из **HTTP-cookie** и **HTTP-заголовков (headers)**. Для этого есть специальные функции-маркеры:

- **`Cookie()`** — читает значение из cookie запроса;
- **`Header()`** — читает значение из заголовка запроса.

Обе работают так же, как `Query()` и `Path()`: дают валидацию, значения по умолчанию, метаданные для OpenAPI и автодополнение. Главное отличие в источнике данных и в нескольких особенностях `Header` (конвертация подчёркиваний в дефисы и поддержка повторяющихся заголовков).

В разделе разбираем: базовое использование `Cookie()`/`Header()`, автоконвертацию `_` → `-`, дублирующиеся заголовки (списки), модели для cookie- и header-параметров (Pydantic-модели, запрет лишних полей через `extra="forbid"`), а также типичные кейсы — `User-Agent`, заголовки авторизации, кастомные заголовки.

> Все примеры на **Pydantic v2** и в синтаксисе **`Annotated[...]`** — рекомендуемом стиле FastAPI.

## Ключевые концепции

- **`Cookie()`** и **`Header()`** — функции из `fastapi`, аналоги `Query()`/`Path()`; без них скаляр был бы воспринят как query-параметр.
- **Автоконвертация подчёркиваний**: в Python имена параметров с дефисом невозможны (`user-agent` — не валидное имя), поэтому `Header` по умолчанию преобразует `user_agent` → `User-Agent`. Отключается через `Header(convert_underscores=False)`.
- **Регистронезависимость заголовков**: HTTP-заголовки нечувствительны к регистру; FastAPI это учитывает.
- **Дублирующиеся заголовки**: один заголовок может встречаться несколько раз (например, несколько `X-Token`). Тип `list[str]` соберёт все значения.
- **Модели параметров** (FastAPI ≥ 0.115): можно сгруппировать несколько cookie/header в одну Pydantic-модель — `Cookie`/`Header` применяются к самой модели.
- **`extra="forbid"`** в модели запрещает «лишние» cookie/заголовки — полезно для строгого контроля.
- Cookie и Header — обычные параметры обработчика, их можно смешивать с path/query/body.

## Подробный разбор с примерами кода

### 1. Базовый `Cookie()`

```python
from typing import Annotated
from fastapi import Cookie, FastAPI

app = FastAPI()


@app.get("/items/")
async def read_items(
    # читает cookie с именем "session_id"; необязательный
    session_id: Annotated[str | None, Cookie()] = None,
):
    return {"session_id": session_id}
```

Без `Cookie()` параметр `session_id` был бы query-параметром. Имя cookie совпадает с именем параметра.

### 2. Базовый `Header()`

```python
from typing import Annotated
from fastapi import FastAPI, Header

app = FastAPI()


@app.get("/items/")
async def read_items(
    # параметр user_agent -> заголовок User-Agent (автоконвертация _ -> -)
    user_agent: Annotated[str | None, Header()] = None,
):
    return {"User-Agent": user_agent}
```

### 3. Автоконвертация подчёркиваний `_` → `-`

В Python нельзя назвать переменную `user-agent`. Поэтому `Header` по умолчанию:

- читает имя параметра `user_agent`;
- ищет заголовок `User-Agent` (подчёркивания → дефисы);
- игнорирует регистр.

Если нужно читать заголовок с настоящим подчёркиванием в имени (редко, нестандартно), отключаем конвертацию:

```python
from typing import Annotated
from fastapi import FastAPI, Header

app = FastAPI()


@app.get("/items/")
async def read_items(
    # будет искать заголовок ровно "strange_header", без преобразования
    strange_header: Annotated[
        str | None, Header(convert_underscores=False)
    ] = None,
):
    return {"strange_header": strange_header}
```

> На практике `convert_underscores=False` нужен редко: многие HTTP-прокси/серверы вообще отбрасывают заголовки с подчёркиваниями. Лучше придерживаться стандартного `Foo-Bar`.

### 4. Дублирующиеся заголовки (списки)

Один и тот же заголовок может присутствовать несколько раз. Чтобы получить **все** значения, объявите тип `list[str]`.

```python
from typing import Annotated
from fastapi import FastAPI, Header

app = FastAPI()


@app.get("/items/")
async def read_items(
    # соберёт ВСЕ значения заголовка X-Token в список
    x_token: Annotated[list[str] | None, Header()] = None,
):
    return {"X-Token values": x_token}
```

Запрос с несколькими заголовками:

```
X-Token: aaa
X-Token: bbb
```

Ответ:

```json
{ "X-Token values": ["aaa", "bbb"] }
```

### 5. Валидация и метаданные для Cookie/Header

`Cookie`/`Header` принимают те же валидаторы, что и `Query`/`Path`.

```python
from typing import Annotated
from fastapi import Cookie, FastAPI, Header

app = FastAPI()


@app.get("/secure/")
async def secure(
    session_id: Annotated[
        str,
        Cookie(min_length=10, max_length=64, description="ID сессии"),
    ],
    x_api_version: Annotated[
        str,
        Header(pattern=r"^v\d+$", description="Версия API, напр. v2"),
    ] = "v1",
):
    return {"session_id": session_id, "api_version": x_api_version}
```

### 6. Модели для cookie-параметров (Cookie param models)

Несколько cookie можно сгруппировать в Pydantic-модель. `Cookie()` применяется к модели целиком.

```python
from typing import Annotated
from fastapi import Cookie, FastAPI
from pydantic import BaseModel

app = FastAPI()


class Cookies(BaseModel):
    # запретить любые cookie, кроме описанных ниже
    model_config = {"extra": "forbid"}

    session_id: str
    fatebook_tracker: str | None = None
    googall_tracker: str | None = None


@app.get("/items/")
async def read_items(cookies: Annotated[Cookies, Cookie()]):
    return cookies
```

Поведение:

- FastAPI извлечёт `session_id`, `fatebook_tracker`, `googall_tracker` из cookie запроса;
- провалидирует их как поля модели;
- благодаря `extra="forbid"` любой лишний cookie вызовет ошибку **422** (полезно для строгого контроля).

> `model_config = {"extra": "forbid"}` — это словарь конфигурации Pydantic v2 (эквивалент `ConfigDict(extra="forbid")`).

### 7. Модели для header-параметров (Header param models)

Аналогично для заголовков. Удобно, когда обработчик ждёт фиксированный набор заголовков.

```python
from typing import Annotated
from fastapi import FastAPI, Header
from pydantic import BaseModel

app = FastAPI()


class CommonHeaders(BaseModel):
    model_config = {"extra": "forbid"}  # запретить лишние заголовки

    host: str
    save_data: bool = False
    if_modified_since: str | None = None
    traceparent: str | None = None
    # повторяющийся заголовок как список
    x_tag: list[str] = []


@app.get("/items/")
async def read_items(headers: Annotated[CommonHeaders, Header()]):
    return headers
```

Особенности:

- автоконвертация подчёркиваний работает и здесь: поле `if_modified_since` → заголовок `If-Modified-Since`, `save_data` → `Save-Data`;
- `x_tag: list[str]` соберёт все повторяющиеся заголовки `X-Tag`;
- `extra="forbid"` запретит «лишние» заголовки. **Осторожно**: браузеры/клиенты шлют много стандартных заголовков (`accept`, `user-agent`, ...), и `forbid` на header-модели может неожиданно ломать запросы. Если нужно — отключите конвертацию подчёркиваний на модели: `Header(convert_underscores=False)`.

### 8. Сочетание с path/query/body

Cookie и Header — обычные параметры; их можно комбинировать с остальными источниками.

```python
from typing import Annotated
from fastapi import Cookie, FastAPI, Header, Path, Query
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    price: float


@app.post("/items/{item_id}")
async def create(
    item_id: Annotated[int, Path(ge=1)],                      # path
    item: Item,                                               # body
    q: Annotated[str | None, Query()] = None,                # query
    session_id: Annotated[str | None, Cookie()] = None,      # cookie
    user_agent: Annotated[str | None, Header()] = None,      # header
):
    return {
        "item_id": item_id,
        "item": item,
        "q": q,
        "session_id": session_id,
        "user_agent": user_agent,
    }
```

## Полный рабочий пример

```python
from typing import Annotated
from fastapi import Cookie, FastAPI, Header, HTTPException
from pydantic import BaseModel

app = FastAPI(title="Cookie & Header демо")


# --- Модель cookie: строго фиксированный набор ---
class SessionCookies(BaseModel):
    model_config = {"extra": "forbid"}  # запретить посторонние cookie

    session_id: str
    ab_test_group: str | None = None    # пример трекинг-cookie


# --- Модель заголовков ---
class RequestHeaders(BaseModel):
    # здесь НЕ ставим extra="forbid", т.к. клиенты шлют много стандартных заголовков
    user_agent: str | None = None             # -> User-Agent
    accept_language: str | None = None        # -> Accept-Language
    x_request_id: str | None = None           # -> X-Request-Id (кастомный)
    x_token: list[str] = []                   # -> повторяющиеся X-Token


def check_auth(authorization: str | None) -> str:
    """Простейшая проверка Bearer-токена из заголовка Authorization."""
    if authorization is None or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Требуется Bearer-токен")
    token = authorization.removeprefix("Bearer ").strip()
    if token != "secret-token":
        raise HTTPException(status_code=403, detail="Неверный токен")
    return token


@app.get("/profile/")
async def read_profile(
    cookies: Annotated[SessionCookies, Cookie()],
    headers: Annotated[RequestHeaders, Header()],
    # отдельный заголовок авторизации (можно и через модель, но часто выносят)
    authorization: Annotated[str | None, Header()] = None,
):
    """Возвращает профиль, проверяя cookie-сессию и Bearer-токен."""
    token = check_auth(authorization)
    return {
        "session_id": cookies.session_id,
        "ab_test_group": cookies.ab_test_group,
        "user_agent": headers.user_agent,
        "accept_language": headers.accept_language,
        "x_request_id": headers.x_request_id,
        "x_token_values": headers.x_token,
        "token_ok": bool(token),
    }
```

Пример запроса (cURL):

```bash
curl http://127.0.0.1:8000/profile/ \
  -H "Authorization: Bearer secret-token" \
  -H "Accept-Language: ru-RU" \
  -H "X-Request-Id: abc-123" \
  -H "X-Token: t1" \
  -H "X-Token: t2" \
  --cookie "session_id=sess-xyz; ab_test_group=B"
```

## Частые вопросы на собеседовании (Q/A)

**1. Зачем нужны `Cookie()` и `Header()`, если можно прочитать `Request`?**
Они дают декларативное извлечение, валидацию, значения по умолчанию, автоконвертацию имён и автодокументирование в OpenAPI/Swagger. Прямой доступ через `request.cookies` / `request.headers` теряет валидацию и не попадает в схему.

**2. Почему `Header` конвертирует подчёркивания в дефисы?**
Имена Python-переменных не могут содержать дефис, а большинство HTTP-заголовков пишутся через дефис (`User-Agent`, `Content-Type`). Поэтому `user_agent` автоматически маппится на `User-Agent`. Регистр заголовков при этом игнорируется.

**3. Как отключить конвертацию подчёркиваний и зачем?**
`Header(convert_underscores=False)`. Нужно в редких случаях, когда заголовок реально содержит подчёркивание. На практике многие серверы/прокси отбрасывают такие заголовки, поэтому это антипаттерн без необходимости.

**4. Как получить несколько значений одного заголовка?**
Объявить тип `list[str]`: `x_token: Annotated[list[str] | None, Header()] = None`. FastAPI соберёт все повторяющиеся заголовки `X-Token` в список.

**5. Как сгруппировать несколько заголовков/cookie в одну модель?**
С FastAPI ≥ 0.115 — объявить Pydantic-модель и пометить параметр `Header()`/`Cookie()`: `headers: Annotated[CommonHeaders, Header()]`. Поля модели маппятся на заголовки/cookie с теми же правилами конвертации.

**6. Как запретить лишние cookie/заголовки?**
В модели задать `model_config = {"extra": "forbid"}` (Pydantic v2). Тогда любой непредусмотренный cookie/заголовок даст 422. Для cookie это безопасно; для заголовков — осторожно, т.к. клиенты шлют много стандартных.

**7. Чем cookie-параметр отличается от query-параметра технически?**
Источником данных: query берётся из строки URL (`?a=1`), cookie — из заголовка `Cookie`, header — из произвольного заголовка. В коде разница только в функции-маркере (`Cookie()` vs `Query()`).

**8. Как читать заголовок `Authorization` и токены?**
Через `authorization: Annotated[str | None, Header()] = None`, затем разобрать схему (`Bearer <token>`). Но для реальной авторизации предпочтительнее `fastapi.security` (`OAuth2PasswordBearer`, `HTTPBearer`, `APIKeyHeader`) — они дают интеграцию с OpenAPI и кнопку Authorize в Swagger.

**9. Чувствительны ли имена заголовков к регистру?**
Нет, по стандарту HTTP заголовки регистронезависимы, и FastAPI это учитывает: `user-agent`, `User-Agent`, `USER-AGENT` равнозначны.

**10. Можно ли смешивать Cookie/Header с path, query и body?**
Да, это обычные параметры обработчика. FastAPI определяет источник каждого по его маркеру/типу и собирает все вместе.

**11. Что вернётся, если обязательный cookie/заголовок отсутствует?**
Если параметр объявлен без значения по умолчанию (обязательный) и его нет — FastAPI вернёт `422` с указанием, какого cookie/header не хватает (в `loc` будет, например, `["cookie", "session_id"]`).

**12. Почему `extra="forbid"` на модели заголовков может быть опасен?**
Браузеры и HTTP-клиенты автоматически добавляют множество заголовков (`accept`, `accept-encoding`, `connection`, `user-agent` и т.д.). С `forbid` любой из них, не описанный в модели, вызовет 422 и сломает запрос. Поэтому `forbid` уместен скорее для cookie или для строго контролируемых server-to-server вызовов.

**13. Как задокументировать кастомный заголовок в Swagger?**
Объявить его как `Header()`-параметр с `description`, `examples`, и (при желании) `alias`. Он автоматически появится в OpenAPI как параметр операции, и Swagger UI позволит его задать.

**14. Можно ли установить cookie/заголовок в ответе?**
Да, но это другая сторона: для ответа используют `Response.set_cookie(...)` / `response.headers[...]` или возвращают `JSONResponse(headers=...)`. `Cookie()`/`Header()` — только для **чтения** входящих данных.

## Подводные камни (gotchas)

- **Без `Cookie()`/`Header()` параметр становится query.** Легко забыть маркер и получить данные не оттуда.
- **`convert_underscores` включён по умолчанию.** `user_agent` ищет `User-Agent`, а не `user_agent`. Если нужен буквальный заголовок — отключайте конвертацию (но это редкость).
- **Заголовки с подчёркиваниями часто режутся прокси/веб-серверами** (nginx по умолчанию). Избегайте их.
- **`extra="forbid"` на header-модели ломает реальные запросы** из-за стандартных заголовков браузера.
- **Один заголовок → только первое значение**, если тип не `list`. Для всех значений нужен `list[str]`.
- **Cookie/Header читают только запрос.** Для установки в ответе — `Response.set_cookie` / `response.headers`.
- **Для авторизации лучше `fastapi.security`**, а не ручной разбор `Authorization` — интеграция с OpenAPI и схемами безопасности.
- **Регистр заголовков игнорируется**, но регистр **значений** — нет (например, `Bearer` vs `bearer`).
- **`model_config = {"extra": "forbid"}` — это Pydantic v2** способ; в v1 был `class Config: extra = "forbid"`.
- **Cookie чувствительны к безопасности.** Не доверяйте им слепо: используйте подписанные/HttpOnly/Secure cookie, валидируйте сессии на сервере.

## Лучшие практики

- Используйте **`Annotated[...]`** и явные `Cookie()`/`Header()` для читаемости и корректного источника.
- Давайте заголовкам **`description`/`examples`** — попадёт в OpenAPI.
- Для повторяющихся заголовков объявляйте **`list[str]`**.
- Группируйте связанные cookie/headers в **Pydantic-модель** для чистоты обработчика.
- **`extra="forbid"`** применяйте к cookie-моделям; для header-моделей — с большой осторожностью.
- Придерживайтесь **стандартного именования заголовков** (`X-Foo-Bar`), не используйте подчёркивания.
- Для аутентификации предпочитайте **`fastapi.security`** (`HTTPBearer`, `APIKeyHeader`, `OAuth2`...).
- Не храните чувствительные данные в обычных cookie; используйте **HttpOnly/Secure/SameSite** и серверные сессии.
- Логируйте и пробрасывайте корреляционные заголовки (`X-Request-Id`, `traceparent`) для трассировки.
- Валидируйте формат токенов/версий через `pattern`/`min_length`/`max_length`.

## Шпаргалка

```python
from typing import Annotated
from fastapi import Cookie, FastAPI, Header
from pydantic import BaseModel

app = FastAPI()

# --- Один cookie ---
@app.get("/c")
async def c(session_id: Annotated[str | None, Cookie()] = None): ...

# --- Один header (user_agent -> User-Agent) ---
@app.get("/h")
async def h(user_agent: Annotated[str | None, Header()] = None): ...

# --- Без конвертации подчёркиваний ---
Header(convert_underscores=False)

# --- Повторяющийся заголовок -> список всех значений ---
x_token: Annotated[list[str] | None, Header()] = None   # X-Token: a / X-Token: b -> ["a","b"]

# --- Валидация ---
Cookie(min_length=10, max_length=64)
Header(pattern=r"^v\d+$", description="Версия API")

# --- Модель cookie (строгая) ---
class Cookies(BaseModel):
    model_config = {"extra": "forbid"}      # запретить лишние cookie -> 422
    session_id: str
    tracker: str | None = None

@app.get("/cm")
async def cm(cookies: Annotated[Cookies, Cookie()]): ...

# --- Модель заголовков (БЕЗ forbid, т.к. браузер шлёт много заголовков) ---
class Headers(BaseModel):
    user_agent: str | None = None           # -> User-Agent
    if_modified_since: str | None = None     # -> If-Modified-Since
    x_tag: list[str] = []                    # -> повторяющийся X-Tag

@app.get("/hm")
async def hm(headers: Annotated[Headers, Header()]): ...

# --- Смешивание источников ---
# Path(...) | Query(...) | Body / модель | Cookie(...) | Header(...)

# Правила Header:
#   _ -> -          (user_agent -> User-Agent)
#   регистр имени   -> игнорируется
#   list[str]       -> все значения повторяющегося заголовка
# Cookie/Header только ЧИТАЮТ запрос; для ответа: Response.set_cookie / response.headers
# Для auth: предпочесть fastapi.security (HTTPBearer, APIKeyHeader, OAuth2...)
```
