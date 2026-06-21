# Contextlib — подготовка к собеседованию

## Что это и зачем

`contextlib` — это модуль стандартной библиотеки Python, который предоставляет
утилиты для работы с **менеджерами контекста** (context managers) — объектами,
которые используются с инструкцией `with`.

Базовая идея менеджера контекста — гарантировать корректный **захват** и
**освобождение** ресурса (файл, сетевое соединение, блокировка, транзакция БД,
временная директория и т. д.) даже если внутри блока произошло исключение.

```python
# Классический пример: файл закроется в любом случае,
# даже если внутри блока возникнет исключение.
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
# Здесь файл уже гарантированно закрыт.
```

Чтобы написать собственный менеджер контекста, обычно нужно реализовать класс с
методами `__enter__` и `__exit__`. Это многословно. `contextlib` решает эту
проблему:

- `@contextmanager` — позволяет писать менеджеры контекста как генераторы.
- `ExitStack` / `AsyncExitStack` — динамическое управление несколькими
  ресурсами.
- `suppress` — лаконичное подавление исключений.
- `closing` — превращает объект с методом `close()` в менеджер контекста.
- `redirect_stdout` / `redirect_stderr` — временное перенаправление вывода.
- `nullcontext` — «пустой» менеджер контекста (заглушка).
- `AbstractContextManager` / `AbstractAsyncContextManager` — абстрактные базовые
  классы.
- `ContextDecorator` — менеджер контекста, который можно использовать и как
  декоратор.

**Зачем это знать на собеседовании:** контекст-менеджеры — фундаментальная тема.
Senior-инженер должен понимать протокол `__enter__/__exit__` до деталей
(что возвращает `__exit__`, как подавляются исключения), уметь применять
`ExitStack` для динамического числа ресурсов и владеть асинхронными аналогами.

---

## Ключевые концепции

### Протокол менеджера контекста

Менеджер контекста — это объект, реализующий два метода:

- `__enter__(self)` — вызывается при входе в блок `with`. Возвращаемое значение
  привязывается к переменной после `as`.
- `__exit__(self, exc_type, exc_value, traceback)` — вызывается при выходе из
  блока (как при нормальном завершении, так и при исключении).

```python
# Развёрнутая семантика инструкции:
#   with EXPR as VAR:
#       BODY
#
# Примерно эквивалентна следующему коду:

mgr = EXPR                       # вычисляем выражение
VAR = mgr.__enter__()            # вход; результат -> VAR
exc = True
try:
    try:
        BODY                     # тело блока
    except:
        exc = False
        # Если __exit__ вернёт True — исключение подавляется.
        if not mgr.__exit__(*sys.exc_info()):
            raise
finally:
    if exc:
        mgr.__exit__(None, None, None)  # нормальный выход
```

### Что возвращает `__exit__` (КЛЮЧЕВОЙ момент собеседования)

- Если `__exit__` возвращает **истинное** значение (`True`) — исключение,
  возникшее в теле блока, **подавляется** (считается обработанным).
- Если возвращает **ложное** значение (`False`, `None`, отсутствие `return`) —
  исключение **пробрасывается дальше** (re-raise).

Это самая частая ловушка: случайно вернуть `True` из `__exit__` и
«проглотить» все исключения.

### Аргументы `__exit__`

При нормальном выходе все три аргумента — `None`.
При исключении:
- `exc_type` — класс исключения (например `ValueError`),
- `exc_value` — экземпляр исключения,
- `traceback` — объект трассировки.

### Асинхронные менеджеры контекста

Для `async with` используется протокол с корутинами:
- `__aenter__(self)` — корутина, вызываемая при входе.
- `__aexit__(self, exc_type, exc_value, traceback)` — корутина при выходе.

```python
async with open_connection() as conn:
    await conn.send("ping")
```

---

## Основные функции/классы/методы

### `@contextmanager` — декоратор-генератор

Превращает функцию-генератор в менеджер контекста. Код **до** `yield`
выполняется в `__enter__`, код **после** `yield` — в `__exit__`. Значение,
переданное в `yield`, становится результатом `__enter__` (то, что после `as`).

```python
from contextlib import contextmanager
import time


@contextmanager
def timer(label: str):
    """Замеряет время выполнения блока."""
    start = time.perf_counter()      # код __enter__
    try:
        yield start                  # отдаём значение в "as", приостанавливаемся
    finally:
        # код __exit__: finally гарантирует выполнение даже при исключении
        elapsed = time.perf_counter() - start
        print(f"[{label}] заняло {elapsed:.4f} c")


with timer("вычисления") as t0:
    total = sum(range(1_000_000))
# Вывод: [вычисления] заняло 0.0xxx c
```

**Важно про обработку исключений в генераторе.** Если в теле `with` возникнет
исключение, оно «пробрасывается» в точку `yield` внутри генератора. Поэтому:

```python
from contextlib import contextmanager


@contextmanager
def managed_resource():
    print("захват ресурса")
    resource = {"open": True}
    try:
        yield resource
    except ValueError:
        # Можно обработать конкретное исключение.
        # Если просто проглотить (не пробросить) — исключение подавляется.
        print("поймали ValueError, подавляем")
        # отсутствие raise == исключение подавлено (аналог return True в __exit__)
    finally:
        resource["open"] = False
        print("освобождение ресурса")


with managed_resource() as r:
    raise ValueError("упс")
print("после with — мы здесь, исключение подавлено")
```

Чтобы пробросить исключение дальше — используйте `raise` (без аргументов
перевыбрасывает текущее):

```python
@contextmanager
def strict_resource():
    try:
        yield
    except Exception:
        print("логируем и пробрасываем")
        raise  # перевыброс — исключение НЕ подавляется
    finally:
        print("cleanup")
```

> Подсказка: `@contextmanager` объекты также являются `ContextDecorator`
> (см. ниже), то есть их можно использовать как декораторы.

---

### `closing` — менеджер для объектов с методом `close()`

Оборачивает объект так, чтобы при выходе вызвался `obj.close()`. Удобно для
объектов, у которых нет встроенной поддержки `with`.

```python
from contextlib import closing
from urllib.request import urlopen

# urlopen возвращает объект с методом close(), но в старом коде
# можно явно гарантировать закрытие так:
with closing(urlopen("https://example.com")) as page:
    data = page.read()
# здесь page.close() уже вызван
```

Реализация по сути такова:

```python
from contextlib import contextmanager


@contextmanager
def closing(thing):
    try:
        yield thing
    finally:
        thing.close()
```

---

### `suppress` — подавление указанных исключений

Лаконичная замена `try/except/pass` для конкретных исключений.

```python
from contextlib import suppress
import os

# Вместо:
#   try:
#       os.remove("tmp.txt")
#   except FileNotFoundError:
#       pass

with suppress(FileNotFoundError):
    os.remove("tmp.txt")  # если файла нет — тихо игнорируем

# Можно указать несколько типов:
with suppress(FileNotFoundError, PermissionError):
    os.remove("locked.txt")
```

`suppress` подавляет исключение **и его подклассы**. После подавления
выполнение продолжается **после** блока `with` (не с места ошибки).

```python
with suppress(ValueError):
    print("до ошибки")
    raise ValueError
    print("эта строка не выполнится")  # пропускается
print("продолжаем здесь")
```

---

### `redirect_stdout` / `redirect_stderr` — перенаправление вывода

Временно перенаправляют `sys.stdout` / `sys.stderr` в любой файлоподобный
объект. Полезно при тестировании и захвате вывода библиотек, которые печатают
через `print`.

```python
from contextlib import redirect_stdout
import io

buffer = io.StringIO()
with redirect_stdout(buffer):
    print("Это попадёт в буфер, а не в консоль")
    help(len)  # вывод help() тоже перехватится

captured = buffer.getvalue()
print("Перехвачено символов:", len(captured))
```

```python
from contextlib import redirect_stderr
import sys, io

err_buf = io.StringIO()
with redirect_stderr(err_buf):
    print("ошибка!", file=sys.stderr)
# err_buf.getvalue() содержит "ошибка!\n"
```

Можно даже перенаправить вывод в файл:

```python
from contextlib import redirect_stdout

with open("log.txt", "w", encoding="utf-8") as f:
    with redirect_stdout(f):
        print("эта строка уйдёт в файл log.txt")
```

> Эти менеджеры **не потокобезопасны** (они меняют глобальное состояние
> `sys.stdout`), и эффект распространяется на весь процесс на время блока.

---

### `nullcontext` — менеджер-заглушка

Возвращает менеджер контекста, который ничего не делает. Используется, когда по
условию ресурс может быть либо реальным менеджером, либо «ничем», но код хочется
писать единообразно.

```python
from contextlib import nullcontext


def process(data, *, use_lock=False, lock=None):
    # Если блокировка нужна — используем её, иначе пустой контекст.
    cm = lock if use_lock else nullcontext()
    with cm:
        return sum(data)
```

`nullcontext(enter_result)` может вернуть заданное значение из `__enter__`:

```python
from contextlib import nullcontext


def get_file(path):
    # Либо открываем реальный файл, либо подставляем уже открытый stdout.
    return open(path) if path else nullcontext(sys.stdout)
```

Начиная с Python 3.10 `nullcontext` также поддерживает протокол `async with`.

---

### `ExitStack` — динамическое управление ресурсами

`ExitStack` позволяет программно регистрировать произвольное число менеджеров
контекста и колбэков очистки. Все они закрываются в **обратном** порядке (LIFO)
при выходе из стека. Решает проблему, когда число ресурсов неизвестно заранее.

```python
from contextlib import ExitStack

filenames = ["a.txt", "b.txt", "c.txt"]

with ExitStack() as stack:
    # Открываем заранее неизвестное число файлов:
    files = [stack.enter_context(open(name, encoding="utf-8"))
             for name in filenames]
    # Все файлы будут закрыты автоматически при выходе из with,
    # в порядке, обратном открытию.
    for f in files:
        print(f.readline())
```

Ключевые методы `ExitStack`:

```python
from contextlib import ExitStack

with ExitStack() as stack:
    # 1) enter_context(cm): входит в менеджер cm, регистрирует его __exit__.
    conn = stack.enter_context(open("data.txt"))

    # 2) callback(fn, *args, **kwargs): регистрирует произвольную функцию
    #    очистки (вызовется при выходе).
    stack.callback(print, "очистка выполнена")

    # 3) push(cm): регистрирует объект с методом __exit__ БЕЗ вызова __enter__.
    #    Удобно для уже открытых ресурсов.
    # stack.push(some_obj_with_exit)

    # 4) pop_all(): переносит все зарегистрированные колбэки в НОВЫЙ ExitStack
    #    и очищает текущий. Используется для "передачи владения".
```

**Паттерн «всё или ничего» (атомарная инициализация ресурсов)** с помощью
`pop_all`:

```python
from contextlib import ExitStack


def open_all(filenames):
    """Открывает все файлы атомарно: если хоть один не открылся —
    все уже открытые корректно закроются."""
    with ExitStack() as stack:
        files = [stack.enter_context(open(fn)) for fn in filenames]
        # Если дошли сюда — все файлы открылись успешно.
        # pop_all() "отвязывает" cleanup от этого with,
        # чтобы файлы НЕ закрылись прямо сейчас.
        stack.pop_all()
    return files
```

**Отложенный re-raise / условная очистка:**

```python
from contextlib import ExitStack

stack = ExitStack()
stack.enter_context(open("a.txt"))
# ... работаем ...
stack.close()  # явно закрываем всё, что зарегистрировано
```

---

### `@asynccontextmanager` — асинхронный аналог `@contextmanager`

Превращает асинхронную генераторную функцию в асинхронный менеджер контекста
для `async with`. Код до `yield` — это `__aenter__`, после — `__aexit__`.

```python
from contextlib import asynccontextmanager
import asyncio


@asynccontextmanager
async def get_connection(host: str):
    print(f"подключаемся к {host}")
    conn = {"host": host, "open": True}
    await asyncio.sleep(0.1)        # имитация асинхронного захвата
    try:
        yield conn                  # отдаём ресурс в "as"
    finally:
        conn["open"] = False
        await asyncio.sleep(0.1)    # имитация асинхронного освобождения
        print(f"отключились от {host}")


async def main():
    async with get_connection("db.local") as conn:
        print("используем:", conn)


asyncio.run(main())
```

Обработка исключений работает так же, как у синхронного `@contextmanager`:

```python
@asynccontextmanager
async def transaction(db):
    await db.begin()
    try:
        yield db
    except Exception:
        await db.rollback()  # откат при ошибке
        raise                # пробрасываем дальше
    else:
        await db.commit()    # коммит при успехе
```

---

### `AsyncExitStack` — асинхронный `ExitStack`

Поддерживает как синхронные, так и асинхронные менеджеры/колбэки.

```python
from contextlib import AsyncExitStack
import asyncio


@asynccontextmanager
async def resource(name):
    print("открыт", name)
    try:
        yield name
    finally:
        print("закрыт", name)


async def main():
    async with AsyncExitStack() as stack:
        # enter_async_context — для async context managers
        r1 = await stack.enter_async_context(resource("A"))
        r2 = await stack.enter_async_context(resource("B"))

        # enter_context — для обычных (синхронных) менеджеров
        f = stack.enter_context(open("data.txt"))

        # push_async_callback — асинхронный колбэк очистки
        async def cleanup():
            print("асинхронная очистка")
        stack.push_async_callback(cleanup)

        # callback — синхронный колбэк
        stack.callback(print, "синхронная очистка")

        print("работаем с", r1, r2)
    # Всё закрывается в обратном порядке (LIFO).


asyncio.run(main())
```

Полезные методы `AsyncExitStack`:
- `enter_async_context(cm)` — для `async with` менеджеров.
- `enter_context(cm)` — для синхронных менеджеров.
- `push_async_exit(cm)` / `push(cm)` — регистрация `__aexit__` / `__exit__`.
- `push_async_callback(fn, ...)` — регистрация async-колбэка.
- `callback(fn, ...)` — регистрация sync-колбэка.
- `aclose()` — явное асинхронное закрытие всего стека.

---

### `AbstractContextManager` / `AbstractAsyncContextManager`

Абстрактные базовые классы (ABC). Предоставляют реализацию `__enter__`
(возвращает `self`) и абстрактный `__exit__`. Поддерживают проверку через
`isinstance` по «структурной» сигнатуре (наличие нужных методов).

```python
from contextlib import AbstractContextManager


class Resource(AbstractContextManager):
    def __init__(self, name):
        self.name = name

    # __enter__ уже определён в базовом классе и возвращает self,
    # но можно переопределить:
    def __enter__(self):
        print("захват", self.name)
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        print("освобождение", self.name)
        # Возвращаем False (None) — исключения не подавляем.
        return False


with Resource("db") as r:
    print("используем", r.name)
```

```python
from contextlib import AbstractContextManager

# isinstance работает по наличию методов __enter__/__exit__:
print(isinstance(open("x.txt"), AbstractContextManager))  # True (если файл открыт)
```

Асинхронный аналог:

```python
from contextlib import AbstractAsyncContextManager


class AsyncResource(AbstractAsyncContextManager):
    async def __aenter__(self):
        print("async захват")
        return self

    async def __aexit__(self, exc_type, exc_value, tb):
        print("async освобождение")
        return False
```

---

### `ContextDecorator` — менеджер контекста как декоратор

Позволяет использовать один и тот же объект и в `with`, и как декоратор функции.

```python
from contextlib import ContextDecorator


class logged(ContextDecorator):
    def __init__(self, label):
        self.label = label

    def __enter__(self):
        print(f"[{self.label}] вход")
        return self

    def __exit__(self, exc_type, exc_value, tb):
        print(f"[{self.label}] выход")
        return False


# Вариант 1: как менеджер контекста
with logged("блок"):
    print("работа в блоке")


# Вариант 2: как декоратор — оборачивает каждый вызовы функции
@logged("функция")
def do_work():
    print("работа в функции")


do_work()
# [функция] вход
# работа в функции
# [функция] выход
```

`@contextmanager` уже наследует поведение `ContextDecorator`, поэтому
генераторные менеджеры тоже можно применять как декораторы:

```python
from contextlib import contextmanager


@contextmanager
def tag(name):
    print(f"<{name}>")
    yield
    print(f"</{name}>")


@tag("section")
def render():
    print("содержимое")


render()
# <section>
# содержимое
# </section>
```

> Важно: при использовании генераторного менеджера как декоратора генератор
> «перезапускается» на каждый вызов функции (нельзя переиспользовать
> исчерпанный генератор).

---

## Частые вопросы на собеседовании

**Q1: Что возвращает метод `__exit__` и на что это влияет?**
A: `__exit__` возвращает булево значение. Если оно **истинно** (`True`),
возникшее в блоке исключение **подавляется** (не пробрасывается). Если
**ложно** (`False`/`None`/нет `return`) — исключение пробрасывается дальше.
По умолчанию следует возвращать `False`, чтобы случайно не «проглотить» ошибки.

**Q2: В чём разница между `@contextmanager` и классом с `__enter__/__exit__`?**
A: Функционально они эквивалентны. `@contextmanager` короче и читабельнее для
простых случаев, использует генератор: код до `yield` = `__enter__`, после =
`__exit__`. Класс предпочтительнее, когда нужно хранить сложное состояние,
переиспользовать менеджер несколько раз, наследоваться или иметь
дополнительные методы. Генераторный менеджер из `@contextmanager` **одноразовый**
(нельзя войти в один и тот же объект дважды).

**Q3: Как в `@contextmanager` корректно обрабатывать исключения?**
A: Исключение из тела `with` «пробрасывается» в точку `yield`. Чтобы код
очистки выполнился всегда — оборачивают `yield` в `try/finally`. Чтобы
подавить исключение — ловят его и **не** перевыбрасывают. Чтобы пробросить —
используют `raise`. Важно не использовать «голый» `return` для подавления — он
не подавляет, нужно именно поглотить исключение в `except`.

**Q4: Зачем нужен `ExitStack`?**
A: Когда число ресурсов неизвестно заранее (например, открыть список файлов),
или когда нужно условно/динамически регистрировать очистку. `ExitStack`
гарантирует освобождение всех ресурсов в обратном порядке даже при ошибке, а
`pop_all()` позволяет реализовать атомарную инициализацию («всё или ничего»).

**Q5: Чем отличается `suppress(Exc)` от `try/except Exc: pass`?**
A: Семантически почти эквивалентны, но `suppress` лаконичнее и выразительнее
для нескольких типов. Нюанс: после подавления `suppress` выполнение
продолжается **после** всего блока `with`, тогда как `try/except` может
продолжить выполнение со следующей строки внутри `try` (если ошибка не на
последней строке). `suppress` ловит указанный тип и его подклассы.

**Q6: Что делает `closing` и когда он нужен?**
A: `closing(obj)` создаёт менеджер контекста, который при выходе вызывает
`obj.close()`. Нужен для объектов, у которых есть `close()`, но нет встроенной
поддержки `with` (нет `__enter__/__exit__`).

**Q7: Как перехватить вывод `print` в тестах?**
A: Через `redirect_stdout(io.StringIO())`. Внутри блока весь `print`
направляется в буфер, после чего `buffer.getvalue()` возвращает текст. Для
ошибок — `redirect_stderr`. Минус: меняется глобальный `sys.stdout`, не
потокобезопасно.

**Q8: Зачем `nullcontext`?**
A: Чтобы единообразно писать код, где менеджер контекста может присутствовать
или нет (например, опциональная блокировка). Вместо ветвления `if lock: with
lock: ...` пишут `with (lock or nullcontext()): ...`. `nullcontext(value)` может
вернуть заданное значение из `__enter__`.

**Q9: Чем `async with` отличается от `with`?**
A: Использует протокол `__aenter__`/`__aexit__`, которые являются корутинами и
могут выполнять `await` (асинхронный захват/освобождение). Применяется только
внутри `async def`. Аналоги в `contextlib`: `@asynccontextmanager`,
`AsyncExitStack`, `AbstractAsyncContextManager`.

**Q10: Что произойдёт, если `__enter__` бросит исключение?**
A: `__exit__` НЕ будет вызван (так как вход не завершился успешно), а тело блока
не выполнится. Исключение пробросится из инструкции `with`. Поэтому очистку
частично захваченных ресурсов внутри `__enter__` нужно делать самостоятельно
(или использовать `ExitStack`).

**Q11: В каком порядке закрываются ресурсы в `ExitStack`?**
A: В обратном порядке регистрации (LIFO, как стек). Последний вошедший —
первый вышедший. Это соответствует семантике вложенных `with`.

**Q12: Что такое `ContextDecorator` и зачем он?**
A: Базовый класс, позволяющий использовать менеджер контекста ещё и как
декоратор функции. Тогда тело функции оборачивается в `with`. `@contextmanager`
наследует это поведение автоматически.

**Q13: Можно ли переиспользовать менеджер из `@contextmanager`?**
A: Нет. Генератор одноразовый: после выхода из `with` он исчерпан. Повторный
вход в тот же объект вызовет `RuntimeError`. Для повторного использования нужно
вызывать функцию-фабрику заново каждый раз или писать класс.

**Q14: Подавит ли `__exit__`, вернувший `True`, исключение `KeyboardInterrupt`
или `SystemExit`?**
A: Да, технически вернув `True` из `__exit__`, вы подавите любое исключение,
включая `KeyboardInterrupt`/`SystemExit` (они тоже передаются в `__exit__`).
Это опасно — обычно их подавлять не следует. `suppress` подавляет только
переданные ему типы, поэтому он безопаснее «голого» `return True`.

---

## Подводные камни (gotchas)

1. **Случайное подавление исключений.** Возврат истинного значения из `__exit__`
   (или поглощение исключения в `except` внутри `@contextmanager` без `raise`)
   тихо «проглатывает» ошибки. Если не намеревались подавлять — всегда
   возвращайте `False`/`None`.

   ```python
   class Bad:
       def __enter__(self): return self
       def __exit__(self, *exc):
           return True  # ОПАСНО: подавит ЛЮБОЕ исключение в блоке!

   with Bad():
       raise RuntimeError("потеряется")  # тихо проглочено
   ```

2. **`@contextmanager` без `try/finally`.** Если код после `yield` не обёрнут в
   `finally`, то при исключении в теле `with` очистка не выполнится.

   ```python
   @contextmanager
   def leak():
       res = acquire()
       yield res
       release(res)  # НЕ вызовется при исключении в блоке!
   # Правильно — обернуть в try/finally.
   ```

3. **Исключение в `__enter__` => `__exit__` не вызывается.** Частично
   захваченные ресурсы внутри `__enter__` нужно подчищать вручную.

4. **`redirect_stdout` не потокобезопасен.** Он меняет глобальный
   `sys.stdout`, поэтому в многопоточном коде перенаправление затронет все
   потоки на время блока.

5. **Переиспользование исчерпанного генераторного менеджера.** Объект из
   `@contextmanager` нельзя использовать в двух `with` подряд — будет
   `RuntimeError`.

   ```python
   cm = some_contextmanager()
   with cm: ...
   with cm: ...  # RuntimeError: generator didn't yield / already used
   ```

6. **`suppress` подавляет и подклассы.** `suppress(Exception)` подавит почти
   всё — будьте конкретны в указании типов.

7. **Несколько `yield` в `@contextmanager`.** Генератор обязан сделать ровно
   один `yield`. Два `yield` приведут к `RuntimeError`
   (`generator didn't stop`).

8. **Забытый `await` для асинхронных менеджеров.** В `AsyncExitStack` нужно
   `await stack.enter_async_context(...)`, а сам блок — `async with`. Пропуск
   `await` ведёт к незавершённой корутине.

9. **Возврат `True` из асинхронного `__aexit__`** так же подавляет исключение,
   как и в синхронном случае.

10. **`pop_all` без последующего управления.** Если вызвали `stack.pop_all()`
    и потеряли ссылку на новый стек, ресурсы не закроются. `pop_all` именно
    «передаёт владение» — кто-то должен закрыть полученный стек.

---

## Лучшие практики

- В `__exit__` по умолчанию возвращайте `False` (или ничего) — не подавляйте
  исключения без явной необходимости.
- Для подавления конкретных исключений используйте `suppress`, а не голый
  `try/except: pass` — читабельнее и безопаснее по объёму подавляемого.
- В `@contextmanager` всегда оборачивайте `yield` в `try/finally`, если есть
  обязательная очистка.
- Для динамического или заранее неизвестного числа ресурсов используйте
  `ExitStack` / `AsyncExitStack` вместо глубокой вложенности `with`.
- Для атомарной инициализации нескольких ресурсов применяйте паттерн
  `ExitStack` + `pop_all()`.
- Используйте `nullcontext()` вместо `if/else` для опциональных менеджеров.
- Для тестов и захвата вывода применяйте `redirect_stdout`/`redirect_stderr`
  с `io.StringIO`.
- Класс с `__enter__/__exit__` предпочтительнее `@contextmanager`, когда нужно
  состояние, наследование, переиспользование или дополнительные методы.
- Наследуйтесь от `AbstractContextManager`/`AbstractAsyncContextManager` для
  явного контракта и удобной проверки `isinstance`.
- Для асинхронного кода используйте `@asynccontextmanager` и `AsyncExitStack`,
  не забывая про `await` и `async with`.
- Реализуйте паттерн транзакции через `try/except: rollback; raise / else:
  commit` внутри `@asynccontextmanager`.

---

## Шпаргалка

```python
# --- Протокол ---
class CM:
    def __enter__(self): return resource     # -> то, что после "as"
    def __exit__(self, exc_type, exc_val, tb):
        return False   # False/None -> пробросить; True -> подавить исключение

# --- async-протокол ---
class ACM:
    async def __aenter__(self): ...
    async def __aexit__(self, et, ev, tb): return False

# --- contextmanager (генератор) ---
from contextlib import contextmanager
@contextmanager
def cm():
    setup()
    try:
        yield value          # код до yield = __enter__
    finally:
        cleanup()            # код после yield = __exit__

# --- asynccontextmanager ---
from contextlib import asynccontextmanager
@asynccontextmanager
async def acm():
    await setup()
    try:
        yield value
    finally:
        await cleanup()

# --- suppress: тихо игнорировать исключения ---
from contextlib import suppress
with suppress(FileNotFoundError):
    os.remove(path)

# --- closing: вызвать .close() на выходе ---
from contextlib import closing
with closing(obj) as o: ...

# --- redirect вывода ---
from contextlib import redirect_stdout, redirect_stderr
import io
buf = io.StringIO()
with redirect_stdout(buf): print("в буфер")

# --- nullcontext: заглушка / опциональный менеджер ---
from contextlib import nullcontext
with (lock or nullcontext()): ...

# --- ExitStack: динамические ресурсы (LIFO) ---
from contextlib import ExitStack
with ExitStack() as stack:
    f = stack.enter_context(open("x"))   # вход в менеджер
    stack.callback(fn, *args)            # колбэк очистки
    stack.push(obj_with_exit)            # регистрация __exit__ без __enter__
    new = stack.pop_all()                # передача владения

# --- AsyncExitStack ---
from contextlib import AsyncExitStack
async with AsyncExitStack() as stack:
    r = await stack.enter_async_context(acm())  # async-менеджер
    f = stack.enter_context(open("x"))          # sync-менеджер
    stack.push_async_callback(async_cleanup)    # async-колбэк
    stack.callback(sync_cleanup)                # sync-колбэк

# --- ContextDecorator: with И декоратор ---
from contextlib import ContextDecorator
class deco(ContextDecorator):
    def __enter__(self): ...
    def __exit__(self, *exc): return False
@deco()              # как декоратор
def f(): ...
with deco():         # как менеджер
    ...

# --- Абстрактные базовые классы ---
from contextlib import AbstractContextManager, AbstractAsyncContextManager

# === Памятка по __exit__ ===
# return True  -> ИСКЛЮЧЕНИЕ ПОДАВЛЕНО
# return False -> исключение проброшено (по умолчанию так и делайте)
# при нормальном выходе: exc_type=exc_val=tb=None
```
