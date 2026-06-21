# urllib — подготовка к собеседованию

## Что это и зачем

`urllib` — это пакет стандартной библиотеки Python для работы с URL: выполнение HTTP-запросов, разбор и сборка URL, кодирование параметров, обработка ошибок. Главное преимущество — **встроен в Python, не требует установки зависимостей**.

Пакет состоит из модулей:
- `urllib.request` — открытие и чтение URL (HTTP/HTTPS/FTP/file), формирование запросов, обработчики (handlers), opener'ы.
- `urllib.parse` — разбор URL на компоненты, сборка обратно, кодирование/декодирование query-string и percent-encoding.
- `urllib.error` — исключения (`URLError`, `HTTPError`).
- `urllib.robotparser` — разбор `robots.txt`.

На практике для HTTP-клиента чаще берут стороннюю библиотеку `requests` или `httpx` — они удобнее. Но `urllib` важно знать: для окружений без зависимостей, для скриптов, для понимания «как устроено под капотом», и потому что `urllib.parse` используется повсеместно (даже внутри `requests`).

## Ключевые концепции

- **`urlopen` vs `Request`.** `urlopen(url)` — быстрый GET. `Request` — объект-запрос: можно задать метод, заголовки, тело, и затем передать его в `urlopen`.
- **Opener и handlers.** `urlopen` использует глобальный opener. Через `build_opener` можно собрать кастомный opener с обработчиками (прокси, cookies, авторизация, редиректы).
- **Компоненты URL** (RFC 3986): `scheme://netloc/path;params?query#fragment`. `urlparse` разбивает строку на эти части.
- **Percent-encoding** (URL encoding): спецсимволы и не-ASCII кодируются как `%XX`. `quote`/`unquote` — для path; `urlencode` — для query-string.
- **Контекстный менеджер.** Ответ `urlopen` — файлоподобный объект, который нужно закрывать (`with`).

## Основные функции/классы/методы

### `urllib.request.urlopen` — простой GET

```python
from urllib.request import urlopen

with urlopen("https://httpbin.org/get", timeout=10) as resp:
    print(resp.status)              # 200 (HTTPResponse)
    print(resp.headers["Content-Type"])
    data = resp.read()              # bytes!
    text = data.decode("utf-8")     # декодировать вручную
```

Важно: `urlopen` возвращает `bytes`, не `str`. Кодировку определяйте сами (из заголовка `Content-Type` или явно).

### `Request` — заголовки, методы, тело

```python
from urllib.request import Request, urlopen
import json

# POST с JSON-телом
payload = json.dumps({"name": "Иван", "age": 30}).encode("utf-8")
req = Request(
    "https://httpbin.org/post",
    data=payload,                       # наличие data => метод POST
    method="POST",                      # можно указать явно (GET/PUT/DELETE/...)
    headers={
        "Content-Type": "application/json",
        "User-Agent": "my-app/1.0",     # urllib по умолчанию ставит Python-urllib/X.Y
    },
)
with urlopen(req, timeout=10) as resp:
    result = json.loads(resp.read().decode("utf-8"))
    print(result)
```

Ключевая деталь: **если передать `data`, метод по умолчанию становится POST**. Для GET-запроса с заголовками — `data=None`.

### Загрузка с обработкой ошибок

```python
from urllib.request import Request, urlopen
from urllib.error import HTTPError, URLError

req = Request("https://httpbin.org/status/404")
try:
    with urlopen(req, timeout=5) as resp:
        body = resp.read()
except HTTPError as e:               # 4xx/5xx — статус-коды
    print("HTTP-ошибка:", e.code, e.reason)
    print("Тело ошибки:", e.read().decode())   # HTTPError тоже файлоподобный!
except URLError as e:                # сеть, DNS, отказ соединения, timeout
    print("Сетевая ошибка:", e.reason)
```

### `urllib.parse` — разбор и сборка URL

```python
from urllib.parse import urlparse, urlsplit, urlunparse, urljoin

u = urlparse("https://user:pass@example.com:8080/path/to;params?q=1&x=2#frag")
print(u.scheme)     # 'https'
print(u.netloc)     # 'user:pass@example.com:8080'
print(u.hostname)   # 'example.com'
print(u.port)       # 8080
print(u.username)   # 'user'
print(u.path)       # '/path/to'
print(u.params)     # 'params'
print(u.query)      # 'q=1&x=2'
print(u.fragment)   # 'frag'

# Сборка обратно
url = urlunparse(("https", "example.com", "/api", "", "a=1", ""))
# 'https://example.com/api?a=1'

# urlsplit — как urlparse, но без отдельного params (path содержит ;params)
# urljoin — относительные ссылки
urljoin("https://x.com/a/b/c", "../d")     # 'https://x.com/a/d'
urljoin("https://x.com/a/b", "/abs")       # 'https://x.com/abs'
```

### `urlencode` — сборка query-string

```python
from urllib.parse import urlencode

params = {"q": "питон поиск", "page": 2, "tags": ["a", "b"]}

print(urlencode(params))
# 'q=%D0%BF%D0%B8%D1%82%D0%BE%D0%BD+%D0%BF%D0%BE%D0%B8%D1%81%D0%BA&page=2&tags=%5B%27a%27%2C+%27b%27%5D'

# Для списков-повторов (?tags=a&tags=b) — doseq=True
print(urlencode({"tags": ["a", "b"]}, doseq=True))   # 'tags=a&tags=b'

# Собрать полный URL
base = "https://example.com/search"
url = f"{base}?{urlencode(params, doseq=True)}"
```

### `parse_qs` / `parse_qsl` — разбор query-string

```python
from urllib.parse import parse_qs, parse_qsl

parse_qs("a=1&a=2&b=3")        # {'a': ['1', '2'], 'b': ['3']}  (значения — списки)
parse_qsl("a=1&a=2&b=3")       # [('a', '1'), ('a', '2'), ('b', '3')]  (порядок сохранён)
```

### `quote` / `unquote` — percent-encoding

```python
from urllib.parse import quote, unquote, quote_plus, unquote_plus

quote("путь/with space")       # 'путь...%2Fwith%20space'  (safe='/' по умолчанию => / не кодируется)
quote("a/b", safe="")          # 'a%2Fb'  (закодировать и слэш)
unquote("%D0%BF%2Fb")          # 'п/b'

# *_plus — для query: пробел -> '+', и наоборот
quote_plus("a b")              # 'a+b'
unquote_plus("a+b")            # 'a b'
```

Правило: `quote/unquote` — для **компонентов пути**; `quote_plus/unquote_plus` (или `urlencode`/`parse_qs`) — для **query-string**, где пробел кодируется как `+`.

### Кастомный opener: прокси, cookies, без авто-редиректов

```python
from urllib.request import build_opener, HTTPCookieProcessor, ProxyHandler
from http.cookiejar import CookieJar

jar = CookieJar()
opener = build_opener(
    HTTPCookieProcessor(jar),                       # хранит cookies между запросами
    ProxyHandler({"https": "http://proxy:3128"}),   # прокси
)
with opener.open("https://httpbin.org/cookies/set?x=1") as resp:
    resp.read()
print([c.name for c in jar])    # ['x']
```

### Скачивание файла

```python
from urllib.request import urlretrieve
# Простой способ (устаревший, но рабочий): сохранить URL в файл
path, headers = urlretrieve("https://example.com/file.zip", "local.zip")

# Современнее — потоково:
from urllib.request import urlopen
with urlopen("https://example.com/file.zip") as resp, open("local.zip", "wb") as f:
    while chunk := resp.read(8192):
        f.write(chunk)
```

## Частые вопросы на собеседовании

**Q: В чём разница между `urllib` и `requests`?**
A: `urllib` — стандартная библиотека, без зависимостей, но низкоуровневая: возвращает `bytes`, требует ручного кодирования параметров, ручной обработки JSON, отдельной настройки сессий/cookies. `requests` — сторонняя, высокоуровневая: `.json()`, `.text`, автоматический query через `params=`, `Session`, удобная обработка ошибок, пулинг соединений. Для скриптов без зависимостей — `urllib`; для приложений — `requests`/`httpx`.

**Q: Что возвращает `urlopen` — строку или байты?**
A: `bytes`. Декодировать в `str` нужно вручную, кодировку взять из заголовка `Content-Type` (`charset=`) или знать заранее.

**Q: Как сделать POST-запрос через urllib?**
A: Передать `data` (байты) в `Request`/`urlopen`. Само наличие `data` делает метод POST. Для JSON — `json.dumps(...).encode()` + заголовок `Content-Type: application/json`.

**Q: Чем `urlparse` отличается от `urlsplit`?**
A: `urlparse` выделяет отдельный компонент `params` (часть после `;` в последнем сегменте path), `urlsplit` — нет (всё остаётся в path). `urlsplit` обычно достаточно и проще.

**Q: Когда `quote`, а когда `quote_plus`?**
A: `quote` — для пути (пробел → `%20`, `/` по умолчанию не трогает). `quote_plus` — для query-параметров (пробел → `+`). Для целой query-строки лучше `urlencode`.

**Q: Почему `parse_qs` возвращает списки в значениях?**
A: Потому что один ключ может встречаться несколько раз (`a=1&a=2`). `parse_qs` собирает все значения в список. Если нужны пары с сохранением порядка и дублей — `parse_qsl`.

**Q: В чём разница между `HTTPError` и `URLError`?**
A: `URLError` — базовый класс, ошибки уровня соединения/DNS/таймаута (`reason`). `HTTPError` — подкласс `URLError`, возникает на HTTP-статусы 4xx/5xx, несёт `.code` и `.reason`, и при этом является файлоподобным объектом (`e.read()` вернёт тело ответа). Ловить `HTTPError` нужно ДО `URLError`.

**Q: Как передать заголовки (например, User-Agent)?**
A: Через `Request(url, headers={...})` или `req.add_header(...)`. По умолчанию `urllib` ставит `User-Agent: Python-urllib/X.Y`, который многие серверы блокируют.

**Q: Как сохранять cookies между запросами?**
A: Собрать opener с `HTTPCookieProcessor(CookieJar())` через `build_opener` и использовать его `.open()`.

**Q: Как корректно собрать URL с query из словаря?**
A: `urlencode(params, doseq=True)` и приклеить через `?`. `doseq=True` нужен, чтобы списки разворачивались в повторяющиеся ключи.

**Q: Как поставить таймаут?**
A: Параметр `timeout=` в `urlopen`. Без него запрос может висеть бесконечно (используется глобальный socket-таймаут, по умолчанию `None`).

## Подводные камни (gotchas)

- **`urlopen` без `timeout` может зависнуть навсегда.** Всегда задавайте `timeout`.
- **`urlopen` возвращает `bytes`** — частая ошибка `resp.read()` сравнивать со `str`.
- **`HTTPError` — это исключение И ответ одновременно.** Если не поймать его до `URLError`, потеряете `.code`. Тело ошибки доступно через `e.read()`.
- **`data` превращает запрос в POST.** Чтобы добавить заголовки к GET — оставляйте `data=None`.
- **`quote` по умолчанию `safe='/'`** — слэши не кодируются. Если нужно закодировать `/`, передайте `safe=""`.
- **`urlencode` без `doseq=True`** превратит список в его `repr` (`['a','b']`), а не в повторяющиеся ключи.
- **User-Agent по умолчанию** часто приводит к 403; подменяйте его.
- **Нет автоматического пула соединений.** Каждый `urlopen` — новое TCP-соединение; для множества запросов это медленно (в отличие от `requests.Session`).
- **`urlretrieve` устарел** и не имеет нормальной обработки ошибок/прогресса — для серьёзного кода качайте потоково.
- **SSL-проверка.** По умолчанию сертификаты проверяются; отключать через `ssl._create_unverified_context()` — опасно, только для отладки.

## Лучшие практики

- Всегда используйте `with urlopen(...) as resp:` и задавайте `timeout`.
- Для query-параметров — `urlencode(..., doseq=True)`, не собирайте строку руками.
- Декодируйте ответ явно, ориентируясь на `Content-Type`.
- Обрабатывайте `HTTPError` отдельно от `URLError` (сначала `HTTPError`).
- Ставьте осмысленный `User-Agent`.
- Для множества запросов к одному хосту, сессий, авторизации, JSON — рассмотрите `requests`/`httpx`; `urllib` оставьте для простых случаев и сред без зависимостей.
- Используйте `urllib.parse` (`urlparse`, `urljoin`, `urlencode`) даже если HTTP делаете через `requests` — это надёжные утилиты.

## Шпаргалка

```python
from urllib.request import urlopen, Request, build_opener, HTTPCookieProcessor
from urllib.parse import (urlparse, urlsplit, urljoin, urlencode,
                          parse_qs, parse_qsl, quote, unquote, quote_plus)
from urllib.error import HTTPError, URLError
from http.cookiejar import CookieJar
import json

# GET
with urlopen("https://x.com", timeout=10) as r:
    text = r.read().decode("utf-8")

# GET с query
url = "https://x.com/search?" + urlencode({"q": "питон", "p": 2}, doseq=True)

# POST JSON
req = Request("https://x.com", data=json.dumps({"a": 1}).encode(),
              headers={"Content-Type": "application/json"}, method="POST")
with urlopen(req, timeout=10) as r:
    res = json.loads(r.read().decode())

# Заголовки к GET
req = Request("https://x.com", headers={"User-Agent": "app/1.0"})

# Ошибки
try:
    urlopen("https://x.com/404")
except HTTPError as e:   # 4xx/5xx, есть e.code, e.read()
    ...
except URLError as e:    # сеть/DNS/timeout, есть e.reason
    ...

# Парсинг URL
p = urlparse("https://u:pw@host:80/path?a=1#f")  # .scheme .hostname .port .path .query
parse_qs("a=1&a=2")          # {'a': ['1','2']}
quote("a/b", safe="")        # 'a%2Fb'
quote_plus("a b")            # 'a+b'  (для query)

# Cookies-сессия
op = build_opener(HTTPCookieProcessor(CookieJar()))
with op.open("https://x.com") as r: ...
```
