# Inspect — подготовка к собеседованию

## Что это и зачем

Модуль `inspect` — это часть стандартной библиотеки Python, предоставляющая инструменты для **интроспекции** (самоанализа) живых объектов программы во время выполнения: функций, классов, методов, модулей, фреймов стека вызовов, трейсбэков и т.д.

Зачем он нужен на практике:

- **Анализ сигнатур функций** — узнать, какие параметры принимает функция, какие у них значения по умолчанию, аннотации типов, к какому «виду» (kind) относится каждый параметр. Это основа для написания декораторов, валидаторов, фреймворков (FastAPI, Click, pytest используют именно `inspect.signature`).
- **Получение исходного кода** — `getsource`, `getsourcelines`, `getsourcefile` для отладки, генерации документации, метапрограммирования.
- **Проверка типа объекта** — `isfunction`, `isclass`, `iscoroutinefunction` и т.п. — для диспетчеризации поведения (например, async vs sync).
- **Работа со стеком вызовов** — `stack()`, `currentframe()`, `getframeinfo()` — для логирования, отладчиков, профилировщиков, фреймворков логирования (loguru, logging определяет, откуда пришёл вызов).
- **Документация** — `getdoc`, `getmembers` для автодокументирования и REPL-помощников.

Ключевая идея: `inspect` работает с **живыми объектами** (runtime), в отличие от модуля `ast`, который работает со статическим исходным кодом. Если вам нужно понять, что объект представляет собой *сейчас*, в памяти — это `inspect`.

```python
import inspect

def greet(name: str, greeting: str = "Привет") -> str:
    """Возвращает приветствие."""
    return f"{greeting}, {name}!"

# Узнаём сигнатуру, не вызывая функцию
print(inspect.signature(greet))          # (name: str, greeting: str = 'Привет') -> str
print(inspect.getdoc(greet))             # Возвращает приветствие.
print(inspect.isfunction(greet))         # True
```

---

## Ключевые концепции

1. **Объект `Signature`** — описывает вызываемый интерфейс функции/метода: упорядоченный набор параметров и аннотацию возвращаемого значения. Неизменяемый (immutable). Создаётся через `inspect.signature(obj)`.

2. **Объект `Parameter`** — описывает один параметр: имя, вид (`kind`), значение по умолчанию (`default`), аннотацию (`annotation`). Тоже неизменяемый.

3. **Виды параметров (`Parameter.kind`)** — отражают, как параметр может быть передан при вызове. Это центральная и часто спрашиваемая на собеседовании тема:
   - `POSITIONAL_ONLY` — только позиционно (до `/` в сигнатуре).
   - `POSITIONAL_OR_KEYWORD` — позиционно или по имени (обычный случай).
   - `VAR_POSITIONAL` — `*args`.
   - `KEYWORD_ONLY` — только по имени (после `*` или `*args`).
   - `VAR_KEYWORD` — `**kwargs`.

4. **`empty`-сентинел** (`Parameter.empty` / `Signature.empty`) — специальное значение-маркер, обозначающее «значения нет». Используется вместо `None`, потому что `None` может быть легитимным значением по умолчанию или аннотацией.

5. **Связывание аргументов (`bind` / `bind_partial`)** — сопоставление переданных при вызове `*args, **kwargs` с формальными параметрами сигнатуры. Возвращает `BoundArguments`. `apply_defaults()` подставляет значения по умолчанию.

6. **Интроспекция стека** — фреймы (`frame`), `FrameInfo`, `stack()`, `currentframe()`, `trace()`. Требует осторожности: фреймы создают **циклические ссылки** и могут вызывать утечки памяти.

7. **`unwrap`** — снятие обёрток декораторов, использующих `functools.wraps` (через атрибут `__wrapped__`), чтобы добраться до оригинальной функции.

---

## Основные функции/классы/методы

### inspect.signature и объекты Signature / Parameter

`inspect.signature(callable)` возвращает объект `Signature`. Это рекомендуемый современный способ интроспекции сигнатур (заменяет устаревший `getargspec`).

```python
import inspect

def func(a, b, /, c, d=10, *args, e, f=20, **kwargs):
    pass

sig = inspect.signature(func)
print(sig)  # (a, b, /, c, d=10, *args, e, f=20, **kwargs)

# sig.parameters — это OrderedDict (mappingproxy) {имя: Parameter}
for name, param in sig.parameters.items():
    print(f"{name:8} kind={param.kind!s:25} default={param.default!r}")
```

Вывод:

```
a        kind=POSITIONAL_ONLY          default=<class 'inspect._empty'>
b        kind=POSITIONAL_ONLY          default=<class 'inspect._empty'>
c        kind=POSITIONAL_OR_KEYWORD    default=<class 'inspect._empty'>
d        kind=POSITIONAL_OR_KEYWORD    default=10
args     kind=VAR_POSITIONAL           default=<class 'inspect._empty'>
e        kind=KEYWORD_ONLY             default=<class 'inspect._empty'>
f        kind=KEYWORD_ONLY             default=20
kwargs   kind=VAR_KEYWORD              default=<class 'inspect._empty'>
```

#### Разбор всех видов параметров (kind)

```python
import inspect
from inspect import Parameter

def demo(pos_only, /, normal, *args, kw_only, **kwargs):
    pass

sig = inspect.signature(demo)
p = sig.parameters

# POSITIONAL_ONLY: только позиционно, нельзя передать по имени.
# В сигнатуре стоит ДО символа '/'.
assert p["pos_only"].kind == Parameter.POSITIONAL_ONLY

# POSITIONAL_OR_KEYWORD: можно и позиционно, и по имени (обычный параметр).
assert p["normal"].kind == Parameter.POSITIONAL_OR_KEYWORD

# VAR_POSITIONAL: это *args — собирает лишние позиционные аргументы.
assert p["args"].kind == Parameter.VAR_POSITIONAL

# KEYWORD_ONLY: только по имени. Стоит ПОСЛЕ '*' или '*args'.
assert p["kw_only"].kind == Parameter.KEYWORD_ONLY

# VAR_KEYWORD: это **kwargs — собирает лишние именованные аргументы.
assert p["kwargs"].kind == Parameter.VAR_KEYWORD
```

Важные детали по видам:

- Порядок видов в сигнатуре строго определён: `POSITIONAL_ONLY` → `POSITIONAL_OR_KEYWORD` → `VAR_POSITIONAL` → `KEYWORD_ONLY` → `VAR_KEYWORD`.
- `KEYWORD_ONLY` появляется либо после `*args`, либо после «голой» звёздочки `*` в сигнатуре: `def f(a, *, b)` — здесь `b` это `KEYWORD_ONLY`.
- `POSITIONAL_ONLY` синтаксис с `/` доступен с Python 3.8. До этого встречался только у некоторых C-функций (например, у встроенных).

#### default, annotation и empty

```python
import inspect
from inspect import Parameter, Signature

def f(a, b: int, c="по умолчанию", d: str = "оба") -> bool:
    return True

sig = inspect.signature(f)

# default — значение по умолчанию. Если его нет — это Parameter.empty.
print(sig.parameters["a"].default is Parameter.empty)  # True (нет default)
print(sig.parameters["c"].default)                     # 'по умолчанию'

# annotation — аннотация типа. Если её нет — это Parameter.empty.
print(sig.parameters["a"].annotation is Parameter.empty)  # True
print(sig.parameters["b"].annotation)                     # <class 'int'>

# Аннотация возвращаемого значения — в самом Signature.
print(sig.return_annotation)  # <class 'bool'>

# Почему empty, а не None?
# Потому что None может быть ЛЕГИТИМНЫМ значением по умолчанию или аннотацией:
def g(x=None, y: None = None):
    pass

gsig = inspect.signature(g)
print(gsig.parameters["x"].default)                       # None — это РЕАЛЬНЫЙ default
print(gsig.parameters["x"].default is Parameter.empty)    # False
# Если бы использовали None как маркер «нет значения», мы бы не отличили
# «default отсутствует» от «default равен None». Поэтому есть отдельный sentinel.

# Parameter.empty и Signature.empty — это одно и то же (inspect._empty)
print(Parameter.empty is Signature.empty)  # True
```

#### Создание и модификация Signature вручную

`Signature` и `Parameter` неизменяемы, но есть метод `replace()`, возвращающий копию с изменениями.

```python
import inspect
from inspect import Parameter, Signature

# Создаём сигнатуру программно (полезно для метапрограммирования)
params = [
    Parameter("x", Parameter.POSITIONAL_OR_KEYWORD, annotation=int),
    Parameter("y", Parameter.POSITIONAL_OR_KEYWORD, default=0, annotation=int),
]
sig = Signature(params, return_annotation=int)
print(sig)  # (x: int, y: int = 0) -> int

# replace() — создаёт изменённую копию (объекты immutable)
new_param = sig.parameters["y"].replace(default=100)
print(new_param)  # y: int = 100

# Можно присвоить кастомную сигнатуру функции через __signature__
def real_impl(*args, **kwargs):
    return sum(args)

real_impl.__signature__ = sig
print(inspect.signature(real_impl))  # (x: int, y: int = 0) -> int
# Это используют декораторы, чтобы "прозрачно" показывать сигнатуру обёрнутой функции.
```

### bind / bind_partial и apply_defaults

`sig.bind(*args, **kwargs)` сопоставляет переданные аргументы с параметрами и возвращает `BoundArguments`. Бросает `TypeError`, если аргументы не подходят (как при реальном вызове). `bind_partial` — то же, но допускает частичное связывание (не все обязательные аргументы должны быть переданы).

```python
import inspect

def func(a, b, c=3, *, d):
    pass

sig = inspect.signature(func)

# bind: полное связывание. Требует все обязательные аргументы.
bound = sig.bind(1, 2, d=4)
print(bound.arguments)  # {'a': 1, 'b': 2, 'd': 4}  — c НЕ попал (не передан)
print(bound.args)       # (1, 2)
print(bound.kwargs)     # {'d': 4}

# apply_defaults() — подставляет значения по умолчанию для непереданных параметров
bound.apply_defaults()
print(bound.arguments)  # {'a': 1, 'b': 2, 'c': 3, 'd': 4}  — теперь c=3 появился

# bind_partial: можно связать не все аргументы (не упадёт, если чего-то не хватает)
partial = sig.bind_partial(1)
print(partial.arguments)  # {'a': 1}  — b и d не обязательны при partial

# bind упадёт, если не хватает обязательных:
try:
    sig.bind(1)  # нет b и d
except TypeError as e:
    print("Ошибка:", e)  # missing a required argument: 'b'
```

`BoundArguments` — изменяемый объект; можно менять `.arguments` (это `OrderedDict`) перед фактическим вызовом, что удобно для декораторов-валидаторов.

#### Практика: декоратор, проверяющий типы аргументов по аннотациям

```python
import inspect
import functools

def validate_types(func):
    """Декоратор: проверяет, что аргументы соответствуют аннотациям типов."""
    sig = inspect.signature(func)

    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        # Связываем переданные аргументы с параметрами
        bound = sig.bind(*args, **kwargs)
        bound.apply_defaults()  # подставляем дефолты, чтобы проверить и их

        for name, value in bound.arguments.items():
            param = sig.parameters[name]
            annotation = param.annotation
            # Пропускаем параметры без аннотаций и *args/**kwargs
            if annotation is inspect.Parameter.empty:
                continue
            if param.kind in (inspect.Parameter.VAR_POSITIONAL,
                              inspect.Parameter.VAR_KEYWORD):
                continue
            # Проверяем только простые типы (isinstance не работает с typing.List и т.п.)
            if isinstance(annotation, type) and not isinstance(value, annotation):
                raise TypeError(
                    f"Аргумент '{name}'={value!r} должен быть типа "
                    f"{annotation.__name__}, а получен {type(value).__name__}"
                )
        return func(*args, **kwargs)

    return wrapper


@validate_types
def divide(a: int, b: int) -> float:
    return a / b

print(divide(10, 2))        # 5.0 — OK
try:
    divide(10, "2")         # b не int
except TypeError as e:
    print(e)  # Аргумент 'b'='2' должен быть типа int, а получен str
```

#### Практика: декоратор логирования вызовов с именами аргументов

```python
import inspect
import functools

def log_calls(func):
    sig = inspect.signature(func)

    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        bound = sig.bind(*args, **kwargs)
        bound.apply_defaults()
        # Красиво печатаем "имя=значение" для всех аргументов
        arg_str = ", ".join(f"{k}={v!r}" for k, v in bound.arguments.items())
        print(f"-> {func.__name__}({arg_str})")
        result = func(*args, **kwargs)
        print(f"<- {func.__name__} вернул {result!r}")
        return result

    return wrapper

@log_calls
def power(base, exp=2):
    return base ** exp

power(3)  # -> power(base=3, exp=2)  /  <- power вернул 9
```

### getmembers

`inspect.getmembers(object, predicate=None)` возвращает список `(имя, значение)` всех атрибутов объекта, отсортированных по имени. С `predicate` фильтрует.

```python
import inspect

class Robot:
    """Робот."""
    model = "R2"

    def __init__(self, name):
        self.name = name

    def speak(self):
        return "бип-боп"

    @property
    def label(self):
        return f"Robot({self.name})"

# Только методы класса
methods = inspect.getmembers(Robot, predicate=inspect.isfunction)
print([name for name, _ in methods])  # ['__init__', 'speak']

# Все члены экземпляра (включая унаследованные dunder-методы)
r = Robot("Бендер")
data = inspect.getmembers(r, predicate=lambda v: not callable(v))
# Найдём пользовательские атрибуты данных
print([(n, v) for n, v in data if not n.startswith("__")])
# [('label', 'Robot(Бендер)'), ('model', 'R2'), ('name', 'Бендер')]

# Полезные предикаты: isfunction, ismethod, isclass, isroutine, isdatadescriptor
```

В Python 3.11+ есть `getmembers_static` — не запускает дескрипторы (`property`, `__getattr__`), что безопаснее при интроспекции «хитрых» объектов.

### getsource / getsourcefile / getsourcelines

Получение исходного кода живого объекта (работает только если код доступен в файле, не в REPL и не в скомпилированном .pyc без исходника).

```python
import inspect

def factorial(n):
    """Факториал."""
    return 1 if n <= 1 else n * factorial(n - 1)

# getsource — строка с исходным кодом
print(inspect.getsource(factorial))

# getsourcelines — кортеж (список строк, номер первой строки)
lines, start_line = inspect.getsourcelines(factorial)
print(f"Начинается со строки {start_line}, всего строк: {len(lines)}")

# getsourcefile — путь к файлу, где определён объект
print(inspect.getsourcefile(factorial))  # /путь/к/файлу.py

# getfile — похоже, но возвращает файл и для C-модулей (или бросает TypeError)
print(inspect.getfile(factorial))
```

Подводные камни: для встроенных функций (написанных на C) `getsource` бросает `TypeError: ... is a built-in function`. В интерактивной сессии (REPL) бросает `OSError: could not get source code`.

### isfunction / isclass / ismethod / isgenerator / iscoroutinefunction / isbuiltin / ismodule

Предикаты-проверки типа объекта. Возвращают `bool`.

```python
import inspect
import asyncio

def regular_func():
    pass

async def async_func():
    pass

def gen_func():
    yield 1

class MyClass:
    def method(self):
        pass

obj = MyClass()

# isfunction — обычная Python-функция (def или lambda), НЕ метод связанный
print(inspect.isfunction(regular_func))      # True
print(inspect.isfunction(MyClass.method))    # True  (несвязанный метод = функция в Py3)
print(inspect.isfunction(obj.method))        # False (это связанный метод!)

# ismethod — СВЯЗАННЫЙ метод (bound method, есть __self__)
print(inspect.ismethod(obj.method))          # True
print(inspect.ismethod(MyClass.method))      # False

# isclass — это класс
print(inspect.isclass(MyClass))              # True
print(inspect.isclass(obj))                  # False

# iscoroutinefunction — функция объявлена через 'async def'
print(inspect.iscoroutinefunction(async_func))  # True
print(inspect.iscoroutinefunction(regular_func)) # False

# isgeneratorfunction — функция-генератор (содержит yield)
print(inspect.isgeneratorfunction(gen_func)) # True
# isgenerator — это сам объект-генератор (результат вызова gen_func())
print(inspect.isgenerator(gen_func()))       # True
print(inspect.isgenerator(gen_func))         # False (это функция, а не генератор)

# isbuiltin — встроенная функция/метод, реализованные на C
print(inspect.isbuiltin(len))                # True
print(inspect.isbuiltin(regular_func))       # False

# ismodule — это модуль
print(inspect.ismodule(inspect))             # True

# iscoroutine — это сам объект-корутина (результат вызова async-функции)
coro = async_func()
print(inspect.iscoroutine(coro))             # True
coro.close()  # чтобы не было предупреждения "coroutine was never awaited"

# isawaitable — можно ли await-ить (корутины, Future, объекты с __await__)
print(inspect.isawaitable(asyncio.sleep(0)))  # True
```

Практическое применение — диспетчеризация sync/async:

```python
import inspect

async def maybe_await(func, *args, **kwargs):
    """Вызывает функцию и await-ит результат, если она асинхронная."""
    if inspect.iscoroutinefunction(func):
        return await func(*args, **kwargs)
    return func(*args, **kwargs)
```

### getfullargspec (и почему getargspec устарел)

`getfullargspec(func)` возвращает именованный кортеж `FullArgSpec` с детальной информацией о сигнатуре. Это «старый» низкоуровневый API; для нового кода предпочтительнее `signature()`.

```python
import inspect

def func(a, b=1, *args, c, d=2, **kwargs) -> int:
    pass

spec = inspect.getfullargspec(func)
print(spec.args)         # ['a', 'b']       — позиционные параметры
print(spec.varargs)      # 'args'           — имя *args
print(spec.varkw)        # 'kwargs'         — имя **kwargs
print(spec.defaults)     # (1,)             — дефолты позиционных (с конца)
print(spec.kwonlyargs)   # ['c', 'd']       — keyword-only параметры
print(spec.kwonlydefaults) # {'d': 2}       — дефолты keyword-only
print(spec.annotations)  # {'return': int}  — аннотации
```

Почему `getargspec` устарел (deprecated, удалён в Python 3.11):

- `getargspec` (старейший) **не поддерживал** keyword-only аргументы (появились в Python 3) и аннотации. Из-за `*args/**kwargs` в новых сигнатурах он просто падал или терял информацию.
- На замену пришёл `getfullargspec`, который понимает keyword-only и аннотации.
- Но `getfullargspec` тоже считается «legacy»: он **не поддерживает** многие callable (например, `functools.partial`, классы с `__call__`, некоторые C-функции) так же гибко, как `signature()`.
- **Рекомендация:** используйте `inspect.signature()` — это единый, расширяемый, корректно работающий с декораторами и `__signature__` механизм.

```python
# Эквивалент через signature (предпочтительно):
import inspect, functools

p = functools.partial(lambda a, b, c: None, 1)
print(inspect.signature(p))           # (b, c) — работает!
# inspect.getfullargspec(p)           # TypeError: unsupported callable
```

### getdoc

`inspect.getdoc(object)` возвращает docstring объекта, **очищенный** от лишних отступов (через `inspect.cleandoc`). В отличие от прямого `obj.__doc__`, наследует docstring от родителя/базового класса, если у самого объекта его нет (с Python 3.5).

```python
import inspect

class Base:
    def method(self):
        """Документация базового метода."""
        pass

class Child(Base):
    def method(self):  # docstring НЕ задан
        pass

# getdoc наследует docstring от родителя
print(inspect.getdoc(Child.method))   # "Документация базового метода."
print(Child.method.__doc__)           # None — прямой доступ НЕ наследует

# cleandoc убирает общий отступ:
def f():
    """Первая строка.

        Отступ выровнен,
        как в исходнике.
    """
print(repr(inspect.getdoc(f)))
# 'Первая строка.\n\nОтступ выровнен,\nкак в исходнике.'
```

### getmro

`inspect.getmro(cls)` возвращает кортеж классов в порядке разрешения методов (Method Resolution Order, MRO) — порядок, в котором Python ищет атрибуты при наследовании (C3-линеаризация). Эквивалентно `cls.__mro__`, но работает и для старых классов-стилей.

```python
import inspect

class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass

print(inspect.getmro(D))
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
# Порядок C3: D -> B -> C -> A -> object
# Это объясняет, как разрешается ромбовидное наследование (diamond problem).

print(D.__mro__ == inspect.getmro(D))  # True
print([c.__name__ for c in inspect.getmro(D)])  # ['D', 'B', 'C', 'A', 'object']
```

### Стек вызовов: stack, currentframe, getframeinfo, FrameInfo, trace

`inspect` даёт доступ к фреймам выполнения (frame objects) — внутренним структурам интерпретатора, описывающим каждый вызов функции.

```python
import inspect

def inner():
    # currentframe() — текущий фрейм выполнения
    frame = inspect.currentframe()
    print("Текущая функция:", frame.f_code.co_name)  # inner
    print("Локальные переменные:", frame.f_locals)

    # getframeinfo(frame) — извлекает удобную информацию о фрейме (Traceback)
    info = inspect.getframeinfo(frame)
    print(f"Файл: {info.filename}, строка: {info.lineno}, функция: {info.function}")

    # stack() — список FrameInfo от текущего фрейма ВВЕРХ до корня (кто нас вызвал)
    for frame_info in inspect.stack():
        # FrameInfo = (frame, filename, lineno, function, code_context, index)
        print(f"  стек: {frame_info.function} @ {frame_info.lineno}")

def outer():
    inner()

outer()
```

`FrameInfo` — именованный кортеж: `(frame, filename, lineno, function, code_context, index)`. В Python 3.11+ добавлены поля `positions` (точные позиции в строке).

```python
import inspect

def who_called_me():
    """Возвращает имя функции, которая вызвала текущую."""
    stack = inspect.stack()
    # stack[0] — это сам who_called_me, stack[1] — вызывающий
    caller = stack[1]
    return caller.function

def some_caller():
    return who_called_me()

print(some_caller())  # 'some_caller'
```

`trace()` — в отличие от `stack()`, возвращает фреймы между текущим местом и точкой, где было выброшено исключение (то есть фреймы трейсбэка). Используется внутри обработчика `except`.

```python
import inspect

def faulty():
    x = 42
    raise ValueError("упс")

try:
    faulty()
except ValueError:
    # trace() — фреймы ОТ try ВНИЗ к месту исключения
    for frame_info in inspect.trace():
        print(f"Исключение в {frame_info.function}, строка {frame_info.lineno}")
        # Можно посмотреть локальные переменные в момент падения:
        print("  локальные:", frame_info.frame.f_locals)
    # Вывод покажет faulty с x=42
```

#### Осторожность с фреймами и утечками памяти

Фреймы держат ссылки на **все локальные переменные** и на вызывающие фреймы (`f_back`). Это создаёт две проблемы:

1. **Циклические ссылки.** Фрейм ссылается на свои локальные переменные; если среди них есть объект, который (прямо или косвенно) ссылается на фрейм — возникает цикл. Сборщик мусора его разберёт, но не мгновенно, и объекты с `__del__` могут задержаться.

2. **Удержание больших объектов в памяти.** Пока вы держите ссылку на frame-объект, живы ВСЕ его локальные переменные (включая большие списки, соединения с БД и т.п.).

Правило: **не храните frame-объекты дольше необходимого. Явно удаляйте ссылку.**

```python
import inspect

def safe_inspect():
    frame = inspect.currentframe()
    try:
        # ... работаем с frame ...
        return frame.f_lineno
    finally:
        # ЯВНО удаляем ссылку, чтобы разорвать возможный цикл
        del frame

# Аналогично stack() и trace() возвращают список с frame-объектами:
def safe_stack_use():
    stack = inspect.stack()
    try:
        return stack[1].function
    finally:
        # Удаляем весь список FrameInfo (каждый держит frame)
        del stack
```

В документации Python это прямо рекомендовано: «Keeping references to frame objects ... can cause your program to create reference cycles. ... it is suggested that you remove the frame ... with `del frame` in a `finally` clause.»

### getclosurevars

`inspect.getclosurevars(func)` возвращает информацию о том, какие имена функция использует из замыкания (closure), глобальной области и встроенных, а также какие имена не разрешены.

```python
import inspect

factor = 3  # глобальная переменная

def make_multiplier(coef):
    bonus = 100  # переменная замыкания (свободная переменная)

    def multiply(x):
        # multiply использует: x (локально), coef и bonus (из замыкания),
        # factor (глобально), len (встроенная)
        return x * coef * factor + bonus + len(str(x))

    return multiply

mul = make_multiplier(5)
cv = inspect.getclosurevars(mul)
print(cv.nonlocals)   # {'coef': 5, 'bonus': 100} — захваченные из внешней функции
print(cv.globals)     # {'factor': 3}            — использованные глобальные
print(cv.builtins)    # {'len': <built-in...>, 'str': ...} — встроенные
print(cv.unbound)     # set()                    — имена, которые нигде не нашлись
```

Полезно для отладки замыканий и понимания, что именно «захватила» функция.

### unwrap для декорированных функций

Когда функция обёрнута декоратором, использующим `functools.wraps`, у обёртки появляется атрибут `__wrapped__`, указывающий на оригинал. `inspect.unwrap()` проходит по цепочке `__wrapped__` до самой нижней функции.

```python
import inspect
import functools

def decorator_a(func):
    @functools.wraps(func)  # ВАЖНО: устанавливает __wrapped__ = func
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

def decorator_b(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@decorator_a
@decorator_b
def original(a, b):
    """Оригинальная функция."""
    return a + b

# unwrap снимает ВСЕ обёртки (идёт по цепочке __wrapped__)
real = inspect.unwrap(original)
print(real.__name__)  # 'original'

# Связь с signature: signature() сам делает unwrap по умолчанию (follow_wrapped=True)
print(inspect.signature(original))  # (a, b) — видим РЕАЛЬНУЮ сигнатуру, а не (*args, **kwargs)

# Если нужна сигнатура самой обёртки, отключаем разворачивание:
print(inspect.signature(original, follow_wrapped=False))  # (*args, **kwargs)

# unwrap со stop — остановиться по условию:
half = inspect.unwrap(original, stop=lambda f: not hasattr(f, "__my_marker__"))
```

Важно: если декоратор **не** использует `functools.wraps`, атрибут `__wrapped__` не ставится, и `unwrap`/`signature` увидят обёртку (`*args, **kwargs`) вместо оригинала. Поэтому всегда используйте `@functools.wraps` в своих декораторах.

---

## Частые вопросы на собеседовании

**Q1. Чем `inspect.signature()` лучше `inspect.getfullargspec()` и старого `getargspec()`?**

A: `getargspec` устарел и удалён в 3.11 — он не поддерживает keyword-only аргументы и аннотации. `getfullargspec` их поддерживает, но плохо работает с `functools.partial`, callable-объектами, декораторами и кастомными `__signature__`. `signature()` — единый, расширяемый API: понимает `__wrapped__` (декораторы), `__signature__` (кастомные сигнатуры), partial, методы, классы. Для нового кода всегда `signature()`.

**Q2. Перечислите пять видов параметров (Parameter.kind) и где каждый встречается в сигнатуре.**

A: `POSITIONAL_ONLY` (до `/`), `POSITIONAL_OR_KEYWORD` (обычный параметр), `VAR_POSITIONAL` (`*args`), `KEYWORD_ONLY` (после `*` или `*args`), `VAR_KEYWORD` (`**kwargs`). Порядок в сигнатуре строго такой же.

**Q3. Почему для отсутствующего значения по умолчанию используется `Parameter.empty`, а не `None`?**

A: Потому что `None` — валидное значение по умолчанию (`def f(x=None)`). Если бы маркером «нет значения» был `None`, нельзя было бы отличить «default отсутствует» от «default равен None». `empty` — это отдельный sentinel-объект (`inspect._empty`), который заведомо не совпадает ни с одним реальным значением.

**Q4. В чём разница между `bind` и `bind_partial`? Что делает `apply_defaults()`?**

A: `bind(*args, **kwargs)` требует все обязательные аргументы и бросает `TypeError`, если их не хватает (как реальный вызов). `bind_partial` допускает неполное связывание. Оба возвращают `BoundArguments`. `apply_defaults()` дописывает в `bound.arguments` значения по умолчанию для параметров, которые не были переданы явно.

**Q5. Как написать декоратор-валидатор аргументов через `signature`?**

A: Получить `sig = inspect.signature(func)` один раз при декорировании. В обёртке вызвать `bound = sig.bind(*args, **kwargs)`, затем `bound.apply_defaults()`, пройтись по `bound.arguments`, сверяя значения с `sig.parameters[name].annotation`. (См. пример `validate_types` выше.) Это устойчиво к позиционным/именованным аргументам и значениям по умолчанию.

**Q6. Чем отличается `isfunction` от `ismethod`?**

A: `isfunction` — True для обычных функций (`def`, `lambda`) и для функций, доступных через класс (`MyClass.method` в Python 3 — это просто функция). `ismethod` — True только для **связанных** методов (bound method), у которых есть `__self__` (например, `instance.method`). Несвязанных методов в Python 3 нет — `MyClass.method` это функция.

**Q7. Что делает `inspect.unwrap` и как он связан с `functools.wraps`?**

A: `unwrap` идёт по цепочке атрибутов `__wrapped__` до оригинальной функции. `functools.wraps` при декорировании устанавливает `wrapper.__wrapped__ = func`. Без `wraps` атрибута нет, и `unwrap`/`signature` увидят обёртку (`*args, **kwargs`). Поэтому декораторы должны использовать `@functools.wraps`.

**Q8. Почему опасно хранить frame-объекты? Как это правильно делать?**

A: Фрейм держит ссылки на все свои локальные переменные и на вызывающие фреймы (`f_back`), что создаёт циклические ссылки и удерживает память (большие объекты не освобождаются). Нужно явно удалять ссылку: `del frame` в `finally`, и не сохранять фреймы в долгоживущих структурах. То же касается списков от `stack()` и `trace()`.

**Q9. В чём разница между `inspect.stack()` и `inspect.trace()`?**

A: `stack()` возвращает фреймы от текущей точки **вверх** к корню (кто вызвал текущую функцию). `trace()` используется в обработчике `except` и возвращает фреймы от точки `try` **вниз** к месту, где было выброшено исключение.

**Q10. Что вернёт `inspect.getdoc()` для метода без docstring, переопределяющего метод родителя с docstring?**

A: Начиная с Python 3.5, `getdoc` наследует docstring от родителя/базового класса, если у самого объекта его нет. То есть вернёт docstring родительского метода. Прямой доступ `obj.__doc__` вернёт `None`.

**Q11. Как `inspect.signature` обрабатывает декорированную функцию и `functools.partial`?**

A: По умолчанию `signature(..., follow_wrapped=True)` разворачивает `__wrapped__`, показывая сигнатуру оригинала. Для `partial` — корректно «вычитает» уже зафиксированные аргументы: `signature(partial(f, 1))` покажет оставшиеся параметры. Можно отключить разворачивание через `follow_wrapped=False`.

**Q12. Что такое MRO и как его получить через inspect?**

A: MRO (Method Resolution Order) — порядок, в котором Python ищет атрибуты/методы при наследовании, вычисляется алгоритмом C3-линеаризации. `inspect.getmro(cls)` (или `cls.__mro__`) возвращает кортеж классов в этом порядке. Объясняет разрешение ромбовидного наследования.

**Q13. Когда `getsource` бросит исключение?**

A: Для встроенных функций/типов на C (`TypeError`), для объектов, определённых в REPL или динамически через `exec`/`compile` без файла (`OSError: could not get source code`), и для объектов из `.pyc` без исходника.

**Q14. Что показывает `getclosurevars` и зачем это нужно?**

A: Возвращает `ClosureVars(nonlocals, globals, builtins, unbound)` — какие имена функция берёт из замыкания, глобальной области, встроенных, и какие не разрешаются. Полезно для отладки замыканий: понять, что именно «захватила» вложенная функция, и нет ли неразрешённых имён.

---

## Подводные камни (gotchas)

- **`empty` ≠ `None`.** Всегда сравнивайте отсутствие default/annotation через `is Parameter.empty`, а не с `None`. Иначе перепутаете «нет дефолта» с «дефолт = None».

- **Frame leaks.** `currentframe()`, `stack()`, `trace()` возвращают фреймы. Не храните их; удаляйте через `del` в `finally`. В долгоживущих логгерах/декораторах это частый источник утечек памяти и задержки сборки мусора.

- **`isfunction` для методов экземпляра возвращает False.** `instance.method` — это bound method (`ismethod` → True), а `Class.method` — функция (`isfunction` → True). Легко ошибиться при фильтрации в `getmembers`.

- **Декораторы без `functools.wraps` ломают интроспекцию.** `signature`/`unwrap`/`getdoc` увидят обёртку (`*args, **kwargs`, пустой docstring) вместо оригинала. Всегда оборачивайте `@functools.wraps`.

- **`getsource` не работает в REPL и для C-функций.** Будет `OSError`/`TypeError`. Оборачивайте в try/except, если код может прийти откуда угодно.

- **`getmembers` запускает дескрипторы.** Доступ к `property` и подобным может выполнить код с побочными эффектами или исключением. В Python 3.11+ используйте `getmembers_static`.

- **`signature.bind` бросает `TypeError`, как реальный вызов.** Если хотите частичную проверку — берите `bind_partial`. Не забывайте обрабатывать `TypeError`.

- **`getfullargspec` не работает с partial и многими callable.** Не используйте его как универсальный инструмент — для произвольных вызываемых берите `signature`.

- **`iscoroutinefunction` vs `iscoroutine`.** Первое — про функцию (`async def`), второе — про объект-корутину (результат её вызова). Путаница приводит к неверной диспетчеризации async-кода. Аналогично `isgeneratorfunction` vs `isgenerator`.

- **Аннотации могут быть строками.** При `from __future__ import annotations` (PEP 563) все аннотации становятся строками. `param.annotation` вернёт строку `'int'`, а не тип `int`. Для разрешения используйте `inspect.signature(func, eval_str=True)` (Python 3.10+) или `typing.get_type_hints()`.

- **`signature` и `*args/**kwargs`.** При связывании `VAR_POSITIONAL` и `VAR_KEYWORD` в `bound.arguments` хранятся как кортеж/словарь под именем `args`/`kwargs`. Учитывайте это при обходе.

---

## Лучшие практики

- Для интроспекции сигнатур используйте **`inspect.signature()`**, а не `getargspec`/`getfullargspec`. Это единый, корректный и расширяемый API.

- Вычисляйте `signature(func)` **один раз** при декорировании (вне обёртки), а не на каждый вызов — это дорого. Кэшируйте.

- В своих декораторах всегда применяйте **`@functools.wraps`**, чтобы сохранить `__name__`, `__doc__`, `__wrapped__` и не сломать интроспекцию вызывающего кода.

- При работе с фреймами **немедленно освобождайте** их через `del` в `finally`. Никогда не сохраняйте frame-объекты в кэшах/глобальных структурах.

- Для проверки «отсутствия» default/annotation сравнивайте с **`Parameter.empty` через `is`**, не с `None`.

- При диспетчеризации sync/async используйте **`iscoroutinefunction`** (и не забывайте про `unwrap`, если функция может быть декорирована — `iscoroutinefunction` в 3.8+ умеет разворачивать `__wrapped__`).

- Оборачивайте `getsource`/`getsourcefile` в **try/except** (`OSError`, `TypeError`) — исходник доступен не всегда.

- Для разрешения строковых аннотаций (PEP 563) используйте **`get_type_hints()`** или `signature(..., eval_str=True)`, а не парсите строки руками.

- В Python 3.11+ предпочитайте **`getmembers_static`** при интроспекции объектов с «магическими» дескрипторами, чтобы не вызывать побочные эффекты.

---

## Шпаргалка

```python
import inspect
from inspect import Parameter, Signature

# --- СИГНАТУРЫ ---
sig = inspect.signature(func)              # объект Signature (предпочтительно)
sig.parameters                              # mappingproxy {имя: Parameter}
sig.return_annotation                       # аннотация возврата (или Signature.empty)
p = sig.parameters["x"]
p.name, p.kind, p.default, p.annotation     # свойства параметра
p.default is Parameter.empty                # проверка отсутствия дефолта
sig.replace(...), p.replace(default=...)    # неизменяемые копии
func.__signature__ = sig                    # задать кастомную сигнатуру

# Виды параметров (kind):
Parameter.POSITIONAL_ONLY        # до '/'
Parameter.POSITIONAL_OR_KEYWORD  # обычный
Parameter.VAR_POSITIONAL         # *args
Parameter.KEYWORD_ONLY           # после '*' / '*args'
Parameter.VAR_KEYWORD            # **kwargs

# --- СВЯЗЫВАНИЕ АРГУМЕНТОВ ---
bound = sig.bind(*args, **kwargs)           # полное (TypeError если не хватает)
bound = sig.bind_partial(*args, **kwargs)   # частичное
bound.apply_defaults()                      # подставить дефолты
bound.arguments, bound.args, bound.kwargs

# --- ЛЕГАСИ-СПЕЦИФИКАЦИЯ ---
inspect.getfullargspec(func)   # FullArgSpec (legacy; getargspec удалён в 3.11)

# --- ПРЕДИКАТЫ ТИПА ---
inspect.isfunction(o)          # def / lambda / Class.method
inspect.ismethod(o)            # bound method (instance.method)
inspect.isclass(o)
inspect.ismodule(o)
inspect.isbuiltin(o)           # C-функция (len и т.п.)
inspect.isgeneratorfunction(o) / inspect.isgenerator(o)
inspect.iscoroutinefunction(o) / inspect.iscoroutine(o) / inspect.isawaitable(o)
inspect.isroutine(o)           # функция/метод/builtin/...

# --- ЧЛЕНЫ И ДОКУМЕНТАЦИЯ ---
inspect.getmembers(o, predicate)        # [(имя, значение), ...] отсортировано
inspect.getmembers_static(o, predicate) # 3.11+: без запуска дескрипторов
inspect.getdoc(o)                       # docstring (cleandoc + наследование)
inspect.getmro(cls)                     # кортеж MRO (== cls.__mro__)

# --- ИСХОДНЫЙ КОД (try/except OSError, TypeError) ---
inspect.getsource(o)                    # строка кода
inspect.getsourcelines(o)               # (список строк, номер 1-й строки)
inspect.getsourcefile(o)                # путь к .py
inspect.getfile(o)

# --- ДЕКОРАТОРЫ И ЗАМЫКАНИЯ ---
inspect.unwrap(func, stop=...)          # снять обёртки по __wrapped__
inspect.signature(func, follow_wrapped=False)  # не разворачивать
inspect.signature(func, eval_str=True)  # 3.10+: разрешить строковые аннотации
inspect.getclosurevars(func)            # ClosureVars(nonlocals, globals, builtins, unbound)

# --- СТЕК ВЫЗОВОВ (осторожно с утечками: del в finally!) ---
frame = inspect.currentframe()          # текущий фрейм; потом del frame
inspect.getframeinfo(frame)             # Traceback(filename, lineno, function, ...)
inspect.stack()                         # [FrameInfo, ...] вверх к корню
inspect.trace()                         # [FrameInfo, ...] вниз к месту исключения (в except)
# FrameInfo = (frame, filename, lineno, function, code_context, index)

# Безопасный шаблон работы с фреймом:
def safe():
    frame = inspect.currentframe()
    try:
        ...
    finally:
        del frame   # разрываем циклические ссылки
```
