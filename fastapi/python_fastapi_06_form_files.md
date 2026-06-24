# FastAPI — Формы и файлы — конспект и вопросы

## О чём раздел

Этот раздел про приём данных, которые приходят **не как JSON**, а как HTML-формы и загруженные файлы:

- `Form()` — чтение полей формы (`application/x-www-form-urlencoded`) и почему для этого нужен пакет **python-multipart**;
- чем `form-data` отличается от JSON и когда что используется;
- **Form models** — объявление полей формы через одну Pydantic-модель;
- загрузка файлов: `bytes` vs `UploadFile` и их различия по памяти;
- `File()`, несколько файлов сразу, метаданные файла (имя, content-type);
- комбинирование форм и файлов в одном запросе;
- ключевое ограничение: **нельзя в одном запросе смешивать `Form`-поля и JSON-тело (`Body`)**.

Главная мысль: формы и файлы передаются как `multipart/form-data` (или `urlencoded`), а это совсем другой формат тела, чем JSON. FastAPI даёт для него отдельные инструменты — `Form` и `File`.

## Ключевые концепции

| Концепция | Назначение |
|-----------|------------|
| `Form()` | Объявить параметр как поле HTML-формы (а не JSON-тело). |
| python-multipart | Сторонний пакет, обязательный для парсинга форм и файлов. |
| `File()` | Объявить параметр как загружаемый файл. |
| `bytes` + `File()` | Файл целиком читается в память как байты. |
| `UploadFile` | Объект файла с буфером/диском, метаданными и async-методами. |
| Form models | Pydantic-модель, поля которой берутся из формы. |
| Несколько файлов | `list[UploadFile]` / `list[bytes]`. |
| Form + File вместе | Можно смешивать формы и файлы в одном запросе. |
| Form + Body(JSON) | **Нельзя** одновременно — это разные форматы тела. |

## Подробный разбор с примерами кода

### 1. `Form()` и зачем нужен python-multipart

HTML-формы по умолчанию шлют данные как `application/x-www-form-urlencoded` (или `multipart/form-data`, если есть файлы). Это **не JSON**, поэтому обычные параметры тела (`Body`) не подойдут — нужен `Form`.

```python
from typing import Annotated

from fastapi import FastAPI, Form

app = FastAPI()


@app.post("/login/")
async def login(
    username: Annotated[str, Form()],
    password: Annotated[str, Form()],
):
    # username и password будут прочитаны из тела формы,
    # а не из query-параметров и не из JSON.
    return {"username": username}
```

Чтобы это заработало, нужно установить **python-multipart** — именно он парсит тело формы:

```bash
pip install python-multipart
```

Без него при запросе FastAPI выдаст ошибку про отсутствующую зависимость.

### 2. Чем `form-data` отличается от JSON

| | JSON (`application/json`) | Form (`urlencoded` / `multipart/form-data`) |
|--|---------------------------|---------------------------------------------|
| Формат | Структурированный JSON-объект | Пары `ключ=значение`, плоские поля |
| Вложенность | Полноценная (объекты, массивы) | Плохо передаёт вложенность |
| Файлы | Нельзя (только base64-обходы) | Да, `multipart/form-data` несёт бинарь |
| Декларация в FastAPI | `Body()` / модель | `Form()` / `File()` |
| Типичный сценарий | REST API, SPA | HTML-формы, OAuth2 (`password` flow), загрузка файлов |

Важно: спецификация OAuth2 (`password` grant) требует, чтобы `username` и `password` передавались именно как поля формы — поэтому в auth-эндпоинтах используется `Form`.

### 3. Form models — поля формы через Pydantic-модель

Начиная с современных версий FastAPI (Pydantic v2) можно объявить целую модель, поля которой берутся из формы:

```python
from typing import Annotated

from fastapi import FastAPI, Form
from pydantic import BaseModel

app = FastAPI()


class LoginForm(BaseModel):
    username: str
    password: str
    remember_me: bool = False


@app.post("/login-model/")
async def login_model(data: Annotated[LoginForm, Form()]):
    # Поля username/password/remember_me будут прочитаны из form-data
    # и провалидированы Pydantic-моделью.
    return {"username": data.username, "remember_me": data.remember_me}
```

Можно запретить «лишние» поля формы (которых нет в модели), настроив модель:

```python
from pydantic import BaseModel, ConfigDict


class StrictLoginForm(BaseModel):
    model_config = ConfigDict(extra="forbid")  # неизвестные поля -> ошибка 422

    username: str
    password: str
```

### 4. Загрузка файлов: `bytes` vs `UploadFile`

**Вариант A — `bytes` (файл целиком в память):**

```python
from typing import Annotated

from fastapi import FastAPI, File

app = FastAPI()


@app.post("/files/")
async def create_file(file: Annotated[bytes, File()]):
    # Весь файл загружается в оперативную память как bytes.
    # Подходит только для МАЛЕНЬКИХ файлов.
    return {"file_size": len(file)}
```

**Вариант B — `UploadFile` (рекомендуется):**

```python
from fastapi import FastAPI, UploadFile

app = FastAPI()


@app.post("/uploadfile/")
async def create_upload_file(file: UploadFile):
    # UploadFile использует "spooled" буфер: маленькие файлы держатся в памяти,
    # большие автоматически сбрасываются на диск.
    contents = await file.read()  # читаем содержимое (async!)
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents),
    }
```

**Различия `bytes` vs `UploadFile`:**

| | `bytes` | `UploadFile` |
|--|---------|--------------|
| Память | Весь файл в RAM | «Spooled»: малые — в RAM, большие — на диск |
| Большие файлы | Опасно (OOM) | Безопасно |
| Метаданные | Нет (только байты) | `filename`, `content_type`, `headers` |
| Методы | — | `read()`, `write()`, `seek()`, `close()` (async) |
| Интерфейс | Готовые байты | Файлоподобный объект (`file` — `SpooledTemporaryFile`) |
| Документация OpenAPI | binary | binary |

> Вывод: для реальных загрузок используйте **`UploadFile`** — он экономит память и даёт метаданные. `bytes` оправдан только для заведомо мелких данных.

### 5. `File()` для дополнительной настройки

`UploadFile` можно использовать и без `File()`, но `File()` нужен, чтобы добавить метаданные (описание для документации) или объявить опциональность/несколько файлов явно:

```python
from typing import Annotated

from fastapi import File, UploadFile


@app.post("/upload-described/")
async def upload_described(
    file: Annotated[UploadFile, File(description="Файл, выбранный пользователем")],
):
    return {"filename": file.filename}


# Опциональный файл:
@app.post("/upload-optional/")
async def upload_optional(file: Annotated[UploadFile | None, File()] = None):
    if file is None:
        return {"message": "Файл не передан"}
    return {"filename": file.filename}
```

### 6. Несколько файлов

```python
from typing import Annotated

from fastapi import File, UploadFile


@app.post("/multi-bytes/")
async def multi_bytes(files: Annotated[list[bytes], File()]):
    return {"sizes": [len(f) for f in files]}


@app.post("/multi-upload/")
async def multi_upload(files: list[UploadFile]):
    return {"filenames": [f.filename for f in files]}
```

В HTML это поле с атрибутом `multiple`. Все файлы приходят под одним именем поля.

### 7. Метаданные файла

`UploadFile` несёт полезные атрибуты и async-методы:

```python
@app.post("/inspect/")
async def inspect(file: UploadFile):
    return {
        "filename": file.filename,       # имя файла на клиенте
        "content_type": file.content_type,  # MIME-тип, напр. image/png
        "headers": dict(file.headers),   # заголовки части multipart
    }
    # Методы (все awaitable, кроме доступа к .file):
    # await file.read(size)   - прочитать содержимое
    # await file.write(data)  - записать
    # await file.seek(offset) - перемотать (seek(0) — в начало)
    # await file.close()      - закрыть
    # file.file               - сырой SpooledTemporaryFile (для sync-библиотек)
```

> Поскольку методы асинхронные, обработчик удобно делать `async def`. Если нужна синхронная библиотека (например, Pillow), работайте с `file.file` напрямую.

### 8. Формы и файлы вместе

Можно в одном `multipart/form-data` запросе передать и текстовые поля, и файлы:

```python
from typing import Annotated

from fastapi import FastAPI, File, Form, UploadFile

app = FastAPI()


@app.post("/submit/")
async def submit(
    token: Annotated[str, Form()],            # текстовое поле формы
    note: Annotated[str, Form()],             # ещё одно поле
    file: Annotated[UploadFile, File()],      # файл
):
    return {
        "token": token,
        "note": note,
        "filename": file.filename,
    }
```

Это легитимно, потому что и `Form`, и `File` живут в одном формате тела — `multipart/form-data`.

### 9. Ограничение: нельзя `Form` и JSON-`Body` одновременно

Тело HTTP-запроса имеет **один** Content-Type. Нельзя одновременно объявить `Form`-поля и JSON-тело (`Body`/Pydantic-модель без `Form`):

```python
from typing import Annotated

from fastapi import Body, Form
from pydantic import BaseModel


class Payload(BaseModel):
    x: int


# НЕ РАБОТАЕТ так, как ожидается:
@app.post("/bad/")
async def bad(
    name: Annotated[str, Form()],   # требует multipart/form-data
    payload: Payload,               # требует application/json
):
    ...
# Запрос не может быть одновременно form-data и JSON — клиент не сможет
# корректно отправить такое тело. Выбирайте один формат.
```

Решение: либо всё через `Form`/`File` (и тогда сложные данные передавать как form-поля, при необходимости как JSON-строку в одном поле с ручным парсингом), либо всё через JSON-`Body` (но тогда нельзя слать файлы напрямую). Смешивать `Form` + `File` — можно; смешивать `Form` + JSON-`Body` — нельзя.

## Полный рабочий пример

```python
from typing import Annotated

from fastapi import FastAPI, File, Form, HTTPException, UploadFile, status
from pydantic import BaseModel, ConfigDict

app = FastAPI(title="Forms & Files API")

MAX_SIZE = 5 * 1024 * 1024  # 5 МБ
ALLOWED_TYPES = {"image/png", "image/jpeg"}


# ---------- Form model ----------
class ProfileForm(BaseModel):
    model_config = ConfigDict(extra="forbid")  # лишние поля -> 422

    username: str
    bio: str = ""
    subscribe: bool = False


@app.post("/profile/")
async def update_profile(data: Annotated[ProfileForm, Form()]):
    """Поля профиля приходят как form-data и валидируются моделью."""
    return {"username": data.username, "subscribe": data.subscribe}


# ---------- Загрузка одного файла (UploadFile) ----------
@app.post("/avatar/", status_code=status.HTTP_201_CREATED)
async def upload_avatar(
    file: Annotated[UploadFile, File(description="Аватар пользователя")],
):
    # Проверяем MIME-тип по метаданным.
    if file.content_type not in ALLOWED_TYPES:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=f"Недопустимый тип: {file.content_type}",
        )
    contents = await file.read()  # async-чтение
    if len(contents) > MAX_SIZE:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Файл слишком большой",
        )
    await file.seek(0)  # перемотать, если файл будут читать снова
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents),
    }


# ---------- Несколько файлов ----------
@app.post("/gallery/")
async def upload_gallery(files: list[UploadFile]):
    result = []
    for f in files:
        data = await f.read()
        result.append({"filename": f.filename, "size": len(data)})
    return {"uploaded": result}


# ---------- Форма + файл одновременно ----------
@app.post("/post/")
async def create_post(
    title: Annotated[str, Form()],            # текстовое поле
    body: Annotated[str, Form()],             # текстовое поле
    attachment: Annotated[UploadFile, File()],  # файл
):
    data = await attachment.read()
    return {
        "title": title,
        "body_len": len(body),
        "attachment": attachment.filename,
        "attachment_size": len(data),
    }


# ---------- Мелкие данные как bytes ----------
@app.post("/raw/")
async def upload_raw(file: Annotated[bytes, File()]):
    """bytes — только для маленьких файлов: весь контент в памяти."""
    return {"size": len(file)}
```

## Частые вопросы на собеседовании (Q/A)

**1. Почему для `Form()` и `File()` нужен `python-multipart`?**
Тело формы и multipart-загрузки не являются JSON; их формат (`application/x-www-form-urlencoded` и `multipart/form-data`) парсит именно библиотека `python-multipart`. Без неё FastAPI не сможет разобрать тело и сообщит об отсутствующей зависимости.

**2. В чём разница между `Form` и `Body`?**
`Body` ожидает JSON (`application/json`) и поддерживает вложенные структуры. `Form` читает плоские поля HTML-формы (`urlencoded`/`multipart`). Это разные Content-Type, поэтому в одном запросе их не смешивают.

**3. Когда использовать форму, а когда JSON?**
JSON — для REST API и SPA (структурированные данные). Формы — для классических HTML-форм, загрузки файлов и протоколов вроде OAuth2 `password` flow, где `username`/`password` обязаны быть полями формы.

**4. `bytes` или `UploadFile` — что выбрать и почему?**
`UploadFile` в большинстве случаев. Он использует «spooled» буфер: малые файлы держит в памяти, большие сбрасывает на диск, не съедая RAM. Плюс даёт метаданные (`filename`, `content_type`) и async-методы. `bytes` грузит весь файл в память — годится только для заведомо маленьких данных.

**5. Какие метаданные доступны у `UploadFile`?**
`filename` (имя на клиенте), `content_type` (MIME-тип), `headers` (заголовки части multipart), а также файлоподобный объект `file` (SpooledTemporaryFile).

**6. Почему методы `UploadFile` асинхронные?**
Чтение/запись могут затрагивать диск (когда файл «спулится» на диск). Async-методы (`await file.read()`, `await file.write()`, `await file.seek()`, `await file.close()`) не блокируют event loop. Для синхронных библиотек используют `file.file` напрямую.

**7. Как принять несколько файлов?**
Объявить `files: list[UploadFile]` (или `list[bytes]` через `File()`). Все файлы приходят под одним именем поля; в HTML это `<input type="file" multiple>`.

**8. Можно ли в одном запросе передать и поля формы, и файлы?**
Да. И `Form`, и `File` относятся к `multipart/form-data`, поэтому их можно комбинировать в одном обработчике.

**9. Почему нельзя одновременно использовать `Form` и JSON-`Body`?**
У запроса один Content-Type. `Form` требует `multipart/form-data` (или urlencoded), а JSON-`Body` — `application/json`. Клиент физически не может отправить тело сразу в двух форматах. Выбирайте один.

**10. Как объявить поля формы через Pydantic-модель?**
`data: Annotated[MyModel, Form()]`. Поля модели читаются из form-data и валидируются. Через `ConfigDict(extra="forbid")` можно запретить лишние поля.

**11. Как ограничить размер загружаемого файла?**
Прочитать содержимое и проверить длину (`len(await file.read())`), либо проверять `Content-Length`/потоково. Превышение — отвечать `400`/`413`. Сам FastAPI лимит по размеру не выставляет автоматически — это на стороне приложения или прокси (nginx).

**12. Как проверить тип файла?**
По `file.content_type` (заявленный клиентом MIME-тип). Для надёжности дополнительно проверяют сигнатуру (magic bytes), т.к. `content_type` подделывается.

**13. Что делать после `await file.read()`, если файл нужно прочитать снова?**
Перемотать в начало: `await file.seek(0)`. Иначе повторное чтение вернёт пустые данные (курсор уже в конце).

**14. Можно ли сделать файл опциональным?**
Да: `file: Annotated[UploadFile | None, File()] = None`. Если файл не прислали — параметр будет `None`.

## Подводные камни (gotchas)

- **Забыли `python-multipart`** — самая частая ошибка; формы/файлы не работают, FastAPI кидает ошибку о зависимости.
- **`bytes` для больших файлов** — риск переполнения памяти (OOM). Используйте `UploadFile`.
- **Смешивание `Form` и JSON-`Body`** — не работает; запрос имеет один Content-Type.
- **Повторное чтение без `seek(0)`** — второй `read()` вернёт пусто, потому что курсор в конце.
- **Доверие к `content_type`** — клиент может подделать MIME-тип; проверяйте содержимое.
- **Синхронный код с `UploadFile`** — методы async; для sync-библиотек берите `file.file`.
- **Нет авто-лимита размера** — без проверки злоумышленник зальёт огромный файл; настройте лимиты в приложении и/или на прокси.
- **`Form` поля не вложенные** — сложную вложенность форма не передаёт; либо несколько плоских полей, либо JSON-строка в одном поле с ручным парсингом.
- **`UploadFile` — это Starlette-объект**, а не Pydantic-модель; нельзя описать его как обычное поле модели тела.

## Лучшие практики

1. **Ставьте `python-multipart`** в зависимости проекта, если есть формы/файлы.
2. **Используйте `UploadFile`**, а не `bytes`, для реальных загрузок — память и метаданные.
3. **Делайте обработчики `async def`** и читайте файлы через `await file.read()`.
4. **Валидируйте файлы**: тип (`content_type` + сигнатура), размер, при необходимости имя.
5. **Перематывайте `seek(0)`**, если читаете содержимое несколько раз.
6. **Формы — через модель** (`Annotated[Model, Form()]`) для валидации и читаемости; `extra="forbid"` против лишних полей.
7. **Не смешивайте** `Form`/`File` с JSON-телом; выбирайте единый формат запроса.
8. **Ставьте лимиты размера** на уровне приложения и обратного прокси.
9. **Закрывайте/освобождайте ресурсы** при необходимости (`await file.close()`), особенно при потоковой обработке.

## Шпаргалка

```python
from typing import Annotated
from fastapi import FastAPI, Form, File, UploadFile
from pydantic import BaseModel

app = FastAPI()
# pip install python-multipart   <-- обязательно для Form/File

# --- поля формы ---
@app.post("/login")
async def login(
    username: Annotated[str, Form()],
    password: Annotated[str, Form()],
): ...

# --- форма через модель ---
class FormModel(BaseModel):
    username: str
    remember: bool = False

@app.post("/m")
async def m(data: Annotated[FormModel, Form()]): ...

# --- файл целиком в память (только мелкие!) ---
@app.post("/b")
async def b(file: Annotated[bytes, File()]): ...

# --- UploadFile (рекомендуется): spooled + метаданные ---
@app.post("/u")
async def u(file: UploadFile):
    data = await file.read()        # async
    file.filename                   # имя
    file.content_type               # MIME
    await file.seek(0)              # перемотать
    await file.close()

# --- несколько файлов ---
@app.post("/many")
async def many(files: list[UploadFile]): ...

# --- форма + файл (можно, оба = multipart/form-data) ---
@app.post("/mix")
async def mix(
    title: Annotated[str, Form()],
    file: Annotated[UploadFile, File()],
): ...

# --- НЕЛЬЗЯ: Form + JSON Body одновременно (разные Content-Type) ---
```

| Аспект | `bytes` | `UploadFile` |
|--------|---------|--------------|
| Память | весь файл в RAM | spooled (RAM→диск) |
| Большие файлы | нет | да |
| Метаданные | нет | filename, content_type, headers |
| Методы | нет | async read/write/seek/close |

| Формат | Content-Type | Инструмент |
|--------|--------------|-----------|
| JSON | `application/json` | `Body()` / модель |
| Форма | `application/x-www-form-urlencoded` | `Form()` |
| Форма с файлами | `multipart/form-data` | `Form()` + `File()` |
```
