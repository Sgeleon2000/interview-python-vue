# unittest.mock — подготовка к собеседованию

## Что это и зачем

`unittest.mock` — встроенная библиотека для создания «заглушек» (mock-объектов), заменяющих в тестах реальные зависимости: сетевые вызовы, БД, файловую систему, время, внешние сервисы.

Зачем нужно мокать:
- **Изоляция.** Тестируем свою логику, а не работоспособность внешнего API.
- **Скорость.** Не ходим в сеть/БД, тесты выполняются мгновенно.
- **Детерминизм.** Внешний мир (текущее время, случайность, сбои сети) делается предсказуемым.
- **Проверка взаимодействия.** Убеждаемся, что наш код *вызвал* зависимость правильно (с нужными аргументами, нужное число раз).
- **Симуляция ошибок.** Легко смоделировать таймаут, исключение, которые в реальности воспроизвести трудно.

Главные сущности: `Mock`, `MagicMock`, `patch`, `PropertyMock`, `AsyncMock`, `sentinel`, `call`, `ANY`.

## Ключевые концепции

- **Mock-объект** «принимает» любой вызов и обращение к любому атрибуту, создавая дочерние моки на лету. Запоминает, как с ним взаимодействовали.
- **return_value** — что вернуть при вызове мока.
- **side_effect** — функция/исключение/итерируемое, выполняемое при вызове (для динамики, ошибок, последовательностей).
- **Проверки взаимодействия** — семейство `assert_called*` для проверки факта и параметров вызова.
- **patch** — временная подмена объекта в области имён на время теста с автоматическим восстановлением.
- **«Где патчить»** — ключевое правило: патчим имя *там, где оно используется*, а не там, где определено.
- **spec / autospec** — ограничение интерфейса мока сигнатурой реального объекта, чтобы ловить опечатки и неверные вызовы.

## Основные функции/классы/методы

### Mock и MagicMock

```python
from unittest.mock import Mock, MagicMock

m = Mock()
# Любой атрибут существует и сам является Mock
print(m.foo)               # <Mock id=...>
# Любой вызов проходит и возвращает Mock
print(m.bar(1, 2))         # <Mock ...>

# Настраиваем возвращаемое значение
m.calculate.return_value = 42
assert m.calculate(1, 2, 3) == 42

# Mock запомнил вызов:
m.calculate.assert_called_once_with(1, 2, 3)
```

**Разница Mock vs MagicMock.** `MagicMock` — подкласс `Mock`, который дополнительно поддерживает «магические» (dunder) методы: `__len__`, `__getitem__`, `__iter__`, `__enter__`, `__contains__`, `__str__` и т. д. Обычный `Mock` их НЕ поддерживает.

```python
mm = MagicMock()
mm.__len__.return_value = 5
assert len(mm) == 5                       # работает у MagicMock

mm.__getitem__.return_value = "value"
assert mm["any_key"] == "value"

# Как контекст-менеджер:
mm.__enter__.return_value = "resource"
with mm as r:
    assert r == "resource"

m = Mock()
# len(m)  -> TypeError: object of type 'Mock' has no len()
```

По умолчанию `patch` подставляет `MagicMock` — поэтому он «просто работает» с большинством объектов.

### return_value

```python
service = Mock()
service.get_user.return_value = {"id": 1, "name": "Иван"}

# Сколько бы раз ни вызвали — вернётся одно и то же
assert service.get_user(1) == {"id": 1, "name": "Иван"}
assert service.get_user(999) == {"id": 1, "name": "Иван"}  # аргументы игнорируются
```

### side_effect

`side_effect` мощнее `return_value`. Три режима:

```python
from unittest.mock import Mock

# 1) Исключение — мок "бросит" его при вызове
m = Mock(side_effect=ConnectionError("сеть недоступна"))
try:
    m()
except ConnectionError as e:
    print(e)

# 2) Итерируемое — на каждый вызов следующий элемент
m2 = Mock(side_effect=[1, 2, 3])
assert m2() == 1
assert m2() == 2
assert m2() == 3
# m2()  -> StopIteration на 4-м вызове

# 3) Функция — динамический ответ в зависимости от аргументов
def fake_divide(a, b):
    if b == 0:
        raise ZeroDivisionError
    return a / b

m3 = Mock(side_effect=fake_divide)
assert m3(10, 2) == 5.0

# Комбинация: если side_effect-функция вернёт DEFAULT,
# будет использован return_value
from unittest.mock import DEFAULT
m4 = Mock(return_value="default")
m4.side_effect = lambda *a: DEFAULT
assert m4() == "default"
```

`side_effect` со списком — частый приём для эмуляции «сначала ошибка, потом успех» (retry-логика):

```python
m = Mock(side_effect=[TimeoutError, TimeoutError, "ok"])
# первые два вызова бросят TimeoutError, третий вернёт "ok"
```

### Проверки вызовов (assert_called*)

```python
m = Mock()
m(1, 2, key="value")
m(3)

m.assert_called()                       # был вызван хотя бы раз
m.assert_called_once()                  # ровно один раз -> упадёт (вызван дважды)
m.assert_called_with(3)                 # ПОСЛЕДНИЙ вызов был с (3)
m.assert_any_call(1, 2, key="value")    # такой вызов был среди всех
m.assert_not_called()                   # упадёт, был вызван

# Все вызовы по порядку:
from unittest.mock import call
assert m.call_args_list == [call(1, 2, key="value"), call(3)]
assert m.call_count == 2
assert m.call_args == call(3)           # последний вызов

# Проверка последовательности (может допускать промежуточные вызовы)
m.assert_has_calls([call(1, 2, key="value"), call(3)])
```

`assert_called_once_with(...)` объединяет проверку «вызван ровно раз» и «с такими аргументами».

```python
m = Mock()
m("a", b=2)
m.assert_called_once_with("a", b=2)     # ок
```

**ANY** — игнорировать конкретный аргумент:

```python
from unittest.mock import ANY
m = Mock()
m(timestamp=1234567890, user="bob")
# Не важно, какой timestamp — важен user
m.assert_called_once_with(timestamp=ANY, user="bob")
```

### patch как декоратор / контекст / вручную

```python
from unittest.mock import patch

# myapp/service.py:
#   import requests
#   def fetch(url):
#       return requests.get(url).json()

# 1) Декоратор: мок передаётся в тест как аргумент
@patch("myapp.service.requests")
def test_fetch_decorator(mock_requests):
    mock_requests.get.return_value.json.return_value = {"ok": True}
    from myapp.service import fetch
    assert fetch("http://x") == {"ok": True}
    mock_requests.get.assert_called_once_with("http://x")


# 2) Контекст-менеджер: подмена только внутри with
def test_fetch_context():
    with patch("myapp.service.requests") as mock_requests:
        mock_requests.get.return_value.json.return_value = {"ok": True}
        from myapp.service import fetch
        assert fetch("http://x") == {"ok": True}
    # вне with — настоящий requests


# 3) Вручную (start/stop), например в setUp/tearDown
import unittest
class TestFetch(unittest.TestCase):
    def setUp(self):
        self.patcher = patch("myapp.service.requests")
        self.mock_requests = self.patcher.start()
        # Гарантированная остановка даже при падении теста
        self.addCleanup(self.patcher.stop)

    def test_it(self):
        self.mock_requests.get.return_value.json.return_value = {}
```

**Важно про порядок аргументов при нескольких декораторах** — они применяются *снизу вверх*, а в аргументы идут в обратном порядке:

```python
@patch("myapp.service.db")          # верхний декоратор -> ПОСЛЕДНИЙ аргумент
@patch("myapp.service.cache")       # нижний декоратор -> ПЕРВЫЙ аргумент
def test_order(mock_cache, mock_db):
    ...
```

### ГДЕ ПАТЧИТЬ (главное правило)

Патчить нужно имя в том модуле, где оно *ищется при вызове*, а не там, где определено.

```python
# utils.py
def get_time():
    return 1000

# service.py
from utils import get_time          # импортировали имя В service

def report():
    return get_time()

# ТЕСТ:
# НЕПРАВИЛЬНО — патчим оригинал, но service уже держит свою ссылку:
with patch("utils.get_time", return_value=0):
    report()   # вернёт 1000 — мок НЕ сработал!

# ПРАВИЛЬНО — патчим имя там, где оно используется:
with patch("service.get_time", return_value=0):
    report()   # вернёт 0
```

Если же импорт был `import utils` и вызов `utils.get_time()`, то патчить нужно `utils.get_time`, потому что имя `get_time` ищется в модуле `utils` в момент вызова.

```python
# service2.py
import utils
def report2():
    return utils.get_time()

# Здесь правильно:
with patch("utils.get_time", return_value=0):
    report2()   # вернёт 0
```

Мнемоника: **«patch where it's looked up, not where it's defined»**.

### spec и autospec

Без `spec` мок принимает *любой* атрибут и *любую* сигнатуру — это маскирует опечатки и поломки контракта.

```python
from unittest.mock import Mock, create_autospec

class Api:
    def get(self, url, timeout=5):
        ...

# Без spec — опечатка проходит молча:
m = Mock()
m.gett("x")                # ок (а должно бы упасть)

# spec ограничивает набор атрибутов:
m = Mock(spec=Api)
m.get("x")                 # ок
# m.gett("x")              # AttributeError: нет такого метода

# autospec проверяет ещё и СИГНАТУРУ вызова:
m = create_autospec(Api)
m.get("http://x", timeout=10)     # ок
# m.get()                          # TypeError: пропущен url
# m.get("x", "y", "z")            # TypeError: лишние аргументы
```

В `patch` есть параметры `spec`/`autospec`:

```python
@patch("myapp.service.Api", autospec=True)
def test_with_autospec(MockApi):
    instance = MockApi.return_value
    instance.get.return_value = {"ok": 1}
    ...
```

`spec_set` строже `spec`: запрещает не только чтение, но и *установку* несуществующих атрибутов.

### patch.object, patch.dict, PropertyMock

```python
from unittest.mock import patch, PropertyMock

class Config:
    timeout = 30

# Патч конкретного атрибута объекта/класса:
with patch.object(Config, "timeout", 1):
    assert Config.timeout == 1

# Патч словаря (например, os.environ):
import os
with patch.dict(os.environ, {"API_KEY": "test"}, clear=False):
    assert os.environ["API_KEY"] == "test"
# вне with ключ восстановлен

# Мок свойства (property):
class User:
    @property
    def is_admin(self):
        return False

with patch.object(User, "is_admin", new_callable=PropertyMock) as mock_prop:
    mock_prop.return_value = True
    u = User()
    assert u.is_admin is True
```

### Мок методов, бросающих исключения, и цепочек

```python
m = Mock()
# Цепочка вызовов: response.json()["data"]
m.get.return_value.json.return_value = {"data": [1, 2]}
assert m.get("url").json()["data"] == [1, 2]

# Метод бросает исключение:
m.save.side_effect = IOError("диск полон")
```

### AsyncMock

Для асинхронного кода (корутины) нужен `AsyncMock` — его вызов возвращает awaitable.

```python
import asyncio
from unittest.mock import AsyncMock, patch

# async def fetch_user(client, uid):
#     return await client.get(uid)

async def fetch_user(client, uid):
    return await client.get(uid)

def test_async():
    client = AsyncMock()
    client.get.return_value = {"id": 1}      # будет "обёрнут" в awaitable

    result = asyncio.run(fetch_user(client, 1))
    assert result == {"id": 1}
    client.get.assert_awaited_once_with(1)   # спец-проверки для await
```

`MagicMock` с версии 3.8 автоматически создаёт `AsyncMock` для атрибутов, которые при `autospec` оказываются корутинами. У `AsyncMock` свои проверки: `assert_awaited`, `assert_awaited_once`, `assert_awaited_with`, атрибут `await_count`.

```python
async def coro():
    ...

m = AsyncMock()
# side_effect тоже работает: можно вернуть последовательность или бросить
m.side_effect = [1, 2]
```

## Частые вопросы на собеседовании

**Q1. В чём разница между Mock и MagicMock?**
`MagicMock` — подкласс `Mock` с предконфигурированными «магическими» методами (`__len__`, `__getitem__`, `__iter__`, `__enter__`, `__str__` и др.). Обычный `Mock` их не поддерживает и упадёт на `len(m)`, `m[0]`, `with m`. `patch` по умолчанию использует `MagicMock`, поэтому он «просто работает».

**Q2. return_value vs side_effect — когда что?**
`return_value` — статичный результат вызова (всегда одно и то же). `side_effect` — динамика: функция (ответ зависит от аргументов), исключение (мок бросит его), итерируемое (на каждый вызов следующий элемент). Если заданы оба, и side_effect-функция вернула `DEFAULT`, используется `return_value`.

**Q3. Главное правило: где патчить?**
Патчим имя *там, где оно используется (looked up)*, а не там, где определено. Если в модуле `service` сделан `from utils import f`, патчить надо `service.f`, потому что `service` держит собственную ссылку на функцию. Если же `import utils` и вызов `utils.f()`, патчим `utils.f`.

**Q4. Чем spec/autospec лучше «голого» Mock?**
Голый Mock принимает любой атрибут и любую сигнатуру, маскируя опечатки и поломки контракта (рефакторинг переименовал метод — тест всё равно зелёный). `spec` ограничивает набор разрешённых атрибутов, `autospec`/`create_autospec` дополнительно проверяет сигнатуру вызова. Это делает тесты устойчивыми к изменению интерфейса.

**Q5. Как замокать метод, который бросает исключение?**
Через `side_effect`: `m.method.side_effect = ValueError("boom")`. При вызове мок поднимет это исключение. Можно передать класс или экземпляр исключения. Для последовательности «ошибка, потом успех»: `side_effect=[TimeoutError, "ok"]`.

**Q6. В каком порядке идут аргументы при нескольких @patch?**
Декораторы применяются снизу вверх, а в параметры функции моки приходят в обратном порядке: *самый нижний* декоратор -> *первый* аргумент. То есть порядок аргументов читается «изнутри наружу».

**Q7. Разница между assert_called_with и assert_called_once_with?**
`assert_called_with(...)` проверяет аргументы *последнего* вызова (не важно, сколько было всего). `assert_called_once_with(...)` дополнительно требует, чтобы вызов был *ровно один*. Для строгой проверки чаще нужен второй.

**Q8. Как проверить, что мок НЕ вызывался / вызывался N раз / с любым аргументом?**
`assert_not_called()` — не вызывался. `m.call_count == N` — число вызовов. `assert_any_call(args)` — такой вызов был среди всех. `ANY` (`from unittest.mock import ANY`) — игнорировать конкретный аргумент в проверке (например, timestamp).

**Q9. Как мокать асинхронный код?**
Использовать `AsyncMock` — его вызов возвращает awaitable, поэтому `await mock()` работает. Проверки: `assert_awaited`, `assert_awaited_once_with`, `await_count`. С версии 3.8 `MagicMock`/`autospec` автоматически создаёт `AsyncMock` для корутин.

**Q10. Что такое patch.dict и зачем он?**
Временно изменяет содержимое словаря (часто `os.environ`) на время блока и восстанавливает исходное состояние после. `clear=True` сначала очищает словарь. Полезно для подмены переменных окружения и конфигов без утечки между тестами.

**Q11. Как замокать property?**
Обычное присваивание `instance.prop = value` не сработает для property класса. Нужно `patch.object(Cls, "prop", new_callable=PropertyMock)` и задать `mock.return_value`. `PropertyMock` срабатывает при *обращении* к атрибуту, а не при вызове.

**Q12. patch.start()/stop() — когда применять и как не забыть stop?**
Когда подмена нужна на весь тест-класс без вложенности `with`. Запускают в `setUp`, останавливают в `tearDown`. Надёжнее — `self.addCleanup(patcher.stop)` сразу после `start()`, чтобы stop вызвался даже при падении теста; иначе мок «протечёт» в другие тесты.

**Q13. Чем sentinel и call полезны?**
`sentinel.NAME` — уникальные именованные объекты-маркеры для проверки «что именно прокинули» без создания заглушек. `call(...)` описывает ожидаемый вызов для сравнения с `call_args`/`call_args_list` и `assert_has_calls`.

**Q14. Почему тест с моком может «зеленеть», хотя код сломан?**
Типичные причины: запатчено не то место (looked up vs defined); голый Mock без spec проглатывает опечатки/неверные сигнатуры; проверяли `assert_called` вместо `assert_called_once_with` с аргументами; забыли вообще проверить взаимодействие. Лекарство — `autospec` и строгие `assert_*_with`.

## Подводные камни (gotchas)

- **ГДЕ ПАТЧИТЬ (самый частый баг).** Патч оригинального модуля не влияет, если целевой модуль сделал `from x import f` — у него своя ссылка. Патчите `target_module.f`, а не `x.f`. Правило: «patch where it's looked up».
- **Опечатка в assert-методе молча проходит.** `m.assert_called_once()` — реальный метод, а `m.asert_called_once()` (опечатка) создаст новый дочерний мок и НЕ проверит ничего, тест зелёный. Защита: `spec`/`autospec` (тогда несуществующий атрибут даст AttributeError) или `mock.assert_called` через включённый строгий режим.
- **Голый Mock без spec.** Принимает любые атрибуты/сигнатуры, маскирует поломки контракта. Всегда предпочитайте `autospec=True` или `spec=`.
- **Забыли patcher.stop().** При ручном `start()` без `stop()` мок «протекает» в следующие тесты. Используйте `addCleanup(patcher.stop)`.
- **return_value у цепочек.** `m.get.return_value.json.return_value = ...` легко перепутать с `m.get().json.return_value`. Помните: `return_value` настраивается на самом моке-методе.
- **side_effect-список исчерпался -> StopIteration.** Если вызовов больше, чем элементов в списке, получите `StopIteration`. Соразмеряйте длину.
- **MagicMock истинность и итерация.** `MagicMock()` по умолчанию truthy и итерируется как пустой? Нет — `__iter__` нужно настроить; зато `__bool__` -> True. Учитывайте при тестировании условий.
- **AsyncMock vs Mock для корутин.** Если замокать корутину обычным `Mock`, `await mock()` упадёт (Mock не awaitable). Нужен `AsyncMock`.
- **patch.object с неверным атрибутом.** Если патчите несуществующий атрибут без `create=True`, получите AttributeError (это хорошо — защищает от опечаток).

## Лучшие практики

- По умолчанию используйте `autospec=True` (или `spec=`) — это ловит изменения интерфейса и опечатки.
- Патчите по правилу «where it's looked up»; держите импорты так, чтобы цель патча была очевидна.
- Проверяйте *взаимодействие* строгими ассертами: `assert_called_once_with(...)`, а не просто `assert_called`.
- Минимизируйте число моков в одном тесте — много моков = тест проверяет реализацию, а не поведение (хрупкость).
- Для очистки ручных патчей применяйте `addCleanup(patcher.stop)`.
- Не мокайте то, что вам не принадлежит, на низком уровне; лучше оборачивайте внешний сервис своим адаптером и мокайте адаптер.
- Используйте `ANY` для нестабильных аргументов (время, UUID) вместо отказа от проверки.
- Для AsyncMock используйте `assert_awaited*` вместо `assert_called*`, чтобы убедиться, что корутину действительно ожидали.

## Шпаргалка

```python
from unittest.mock import (
    Mock, MagicMock, AsyncMock, patch,
    PropertyMock, call, ANY, sentinel, DEFAULT,
)

# Настройка
m.method.return_value = 42                 # статичный ответ
m.method.side_effect = Exc                 # бросить исключение
m.method.side_effect = [1, 2, 3]           # последовательность
m.method.side_effect = lambda x: x * 2     # динамика

# Проверки
m.assert_called()                          # хотя бы раз
m.assert_called_once()                     # ровно раз
m.assert_called_with(a, b=1)               # последний вызов
m.assert_called_once_with(a, b=1)          # раз + аргументы
m.assert_any_call(a)                       # был среди вызовов
m.assert_has_calls([call(1), call(2)])     # последовательность
m.assert_not_called()
m.call_count        # число вызовов
m.call_args         # последний вызов = call(...)
m.call_args_list    # все вызовы

# patch (where it's looked up!)
@patch("pkg.module.dependency")            # декоратор
with patch("pkg.module.dependency") as m:  # контекст
patch.object(Cls, "attr", value)           # атрибут объекта
patch.dict(os.environ, {"K": "v"})         # словарь
patch("...", autospec=True)                # проверка сигнатуры
patch("...", new_callable=PropertyMock)    # property

# Несколько @patch: снизу вверх -> аргументы изнутри наружу
@patch("m.b")   # -> 2-й аргумент
@patch("m.a")   # -> 1-й аргумент
def test(mock_a, mock_b): ...

# Async
am = AsyncMock(); am.return_value = ...
am.assert_awaited_once_with(...)
```
