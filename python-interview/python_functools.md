# functools — подготовка к собеседованию

## Что это и зачем

`functools` — модуль стандартной библиотеки Python для работы с функциями высшего порядка и «вызываемыми объектами» (callable). Он помогает:
- **Мемоизировать** результаты функций (`lru_cache`, `cache`, `cached_property`) — кэширование для ускорения.
- **Частично применять** аргументы (`partial`, `partialmethod`).
- **Сворачивать** последовательности в одно значение (`reduce`).
- **Корректно писать декораторы** (`wraps`).
- **Автоматически генерировать методы сравнения** (`total_ordering`).
- **Реализовывать перегрузку функций по типу** (`singledispatch`).

```python
import functools

# Пример: кэшируем тяжёлую функцию одним декоратором
@functools.lru_cache(maxsize=None)
def slow_square(n):
    print(f'вычисляю {n}')
    return n * n

print(slow_square(4))  # вычисляю 4 \n 16
print(slow_square(4))  # 16  — без печати, взято из кэша
```

## Ключевые концепции

1. **Функция высшего порядка** — принимает функции как аргументы и/или возвращает функцию.
2. **Мемоизация** — кэширование результатов функции по её аргументам, чтобы не пересчитывать.
3. **Декоратор** — функция, оборачивающая другую функцию, добавляя поведение.
4. **Частичное применение (partial application)** — фиксация части аргументов функции, получение новой функции с меньшим числом параметров.
5. **Хешируемость аргументов** — для кэширования аргументы должны быть хешируемыми (кортежи можно, списки/словари — нет).

## Основные функции/классы/методы

### reduce — свёртка последовательности

**`reduce(function, iterable[, initializer])`** — последовательно применяет бинарную функцию, «сворачивая» итерируемое в одно значение.

```python
from functools import reduce
import operator

# Сумма (хотя для суммы лучше встроенный sum)
print(reduce(operator.add, [1, 2, 3, 4]))      # 10
# Шаги: ((1+2)+3)+4

# Произведение
print(reduce(operator.mul, [1, 2, 3, 4]))      # 24

# С начальным значением (initializer)
print(reduce(operator.add, [1, 2, 3], 100))    # 106

# initializer защищает от ошибки на пустом итерируемом
print(reduce(operator.add, [], 0))             # 0  (без initializer -> TypeError)

# Нетривиальный пример: найти максимум по длине
words = ['кот', 'собака', 'я']
print(reduce(lambda a, b: a if len(a) >= len(b) else b, words))  # 'собака'
```

### partial — частичное применение

**`partial(func, *args, **keywords)`** — создаёт новый вызываемый объект с зафиксированными аргументами.

```python
from functools import partial

def power(base, exponent):
    return base ** exponent

# Фиксируем exponent через позиционный аргумент base
square = partial(power, exponent=2)
cube = partial(power, exponent=3)
print(square(5))  # 25
print(cube(2))    # 8

# Фиксируем первый позиционный аргумент
power_of_two = partial(power, 2)   # base=2 зафиксировано
print(power_of_two(10))  # 1024  (2 ** 10)

# Частое применение: int с фиксированным основанием
from_binary = partial(int, base=2)
print(from_binary('1010'))  # 10

# partial объект хранит func, args, keywords
p = partial(power, 3)
print(p.func, p.args, p.keywords)  # <function power> (3,) {}
```

### lru_cache — мемоизация с ограничением размера

**`lru_cache(maxsize=128, typed=False)`** — декоратор, кэширующий результаты по аргументам. LRU = Least Recently Used (вытесняет давно не используемые).

```python
from functools import lru_cache

@lru_cache(maxsize=None)  # None -> кэш без ограничения размера
def fib(n):
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)

print(fib(50))  # 12586269025  — мгновенно благодаря кэшу

# Статистика кэша
print(fib.cache_info())
# CacheInfo(hits=48, misses=51, maxsize=None, currsize=51)

# Очистка кэша
fib.cache_clear()

# typed=True: 3 и 3.0 кэшируются раздельно
@lru_cache(typed=True)
def f(x):
    return x
f(3); f(3.0)
print(f.cache_info())  # currsize=2 (раздельно), при typed=False было бы 1
```

Важно про `maxsize`:
- `maxsize=128` (по умолчанию) — хранит максимум 128 результатов, вытесняя старые.
- `maxsize=None` — неограниченный кэш (быстрее, но может расти бесконечно).

### cache — простой неограниченный кэш (Python 3.9+)

**`cache`** — это `lru_cache(maxsize=None)`, но короче и чуть быстрее (нет учёта LRU).

```python
from functools import cache

@cache
def factorial(n):
    return n * factorial(n - 1) if n else 1

print(factorial(5))  # 120
print(factorial(6))  # 720  — factorial(5) взят из кэша
```

### cached_property — кэширующее свойство

**`cached_property`** — дескриптор, превращающий метод в свойство, вычисляемое один раз и сохраняемое в `__dict__` экземпляра.

```python
from functools import cached_property

class Dataset:
    def __init__(self, data):
        self.data = data

    @cached_property
    def stats(self):
        print('считаю статистику...')  # выполнится только при первом обращении
        return {'min': min(self.data), 'max': max(self.data), 'sum': sum(self.data)}

ds = Dataset([3, 1, 4, 1, 5])
print(ds.stats)  # считаю статистику... \n {'min': 1, 'max': 5, 'sum': 14}
print(ds.stats)  # {'min': 1, 'max': 5, 'sum': 14}  — без печати, из кэша

# ВАЖНО: значение хранится в экземпляре, можно «сбросить» через del
del ds.stats
print(ds.stats)  # считаю статистику... снова

# Отличие от property: property вычисляется КАЖДЫЙ раз, cached_property — один раз.
# ВАЖНО: класс должен иметь __dict__ (не работает с __slots__ без 'd'-слота).
```

### wraps — сохранение метаданных при декорировании

**`wraps(func)`** — декоратор для обёрток, копирующий `__name__`, `__doc__`, `__module__`, `__wrapped__` и т.д. из оборачиваемой функции.

```python
from functools import wraps

def my_decorator(func):
    @wraps(func)  # без этого wrapper «потеряет» имя и docstring исходной функции
    def wrapper(*args, **kwargs):
        print('до вызова')
        result = func(*args, **kwargs)
        print('после вызова')
        return result
    return wrapper

@my_decorator
def greet(name):
    """Приветствует пользователя."""
    return f'Привет, {name}!'

print(greet('Аня'))   # до вызова \n после вызова \n Привет, Аня!
print(greet.__name__) # 'greet'   (без @wraps было бы 'wrapper')
print(greet.__doc__)  # 'Приветствует пользователя.'
print(greet.__wrapped__)  # доступ к исходной необёрнутой функции
```

### total_ordering — автогенерация сравнений

**`total_ordering`** — декоратор класса: если определены `__eq__` и хотя бы один из `__lt__`/`__le__`/`__gt__`/`__ge__`, автоматически добавляет остальные.

```python
from functools import total_ordering

@total_ordering
class Version:
    def __init__(self, major, minor):
        self.major = major
        self.minor = minor

    def __eq__(self, other):
        return (self.major, self.minor) == (other.major, other.minor)

    def __lt__(self, other):
        return (self.major, self.minor) < (other.major, other.minor)

    # __le__, __gt__, __ge__ сгенерированы автоматически!

a = Version(1, 2)
b = Version(1, 5)
print(a < b)   # True
print(a >= b)  # False  — работает, хотя __ge__ мы не писали
print(a <= b)  # True
print(b > a)   # True
```

### singledispatch — перегрузка по типу первого аргумента

**`singledispatch`** — превращает функцию в «обобщённую» (generic), у которой можно регистрировать реализации для разных типов первого аргумента.

```python
from functools import singledispatch

@singledispatch
def describe(arg):
    return f'Объект: {arg!r}'  # реализация по умолчанию

@describe.register
def _(arg: int):
    return f'Целое число: {arg}'

@describe.register
def _(arg: list):
    return f'Список из {len(arg)} элементов'

@describe.register(str)  # можно указать тип в скобках
def _(arg):
    return f'Строка длиной {len(arg)}'

print(describe(42))        # Целое число: 42
print(describe([1, 2, 3])) # Список из 3 элементов
print(describe('привет'))  # Строка длиной 6
print(describe(3.14))      # Объект: 3.14  — реализация по умолчанию

# Регистрация нескольких типов одной реализацией (3.7+)
@describe.register(float)
@describe.register(complex)
def _(arg):
    return f'Дробное/комплексное: {arg}'
```

Существует также `singledispatchmethod` для методов классов (диспетчеризация по второму аргументу `self`-метода).

### partialmethod — partial для методов класса

**`partialmethod`** — как `partial`, но для определения методов класса с зафиксированными аргументами.

```python
from functools import partialmethod

class Switch:
    def __init__(self):
        self.state = False

    def set_state(self, state):
        self.state = state

    # Создаём методы с зафиксированным аргументом
    turn_on = partialmethod(set_state, True)
    turn_off = partialmethod(set_state, False)

s = Switch()
s.turn_on()
print(s.state)   # True
s.turn_off()
print(s.state)   # False
```

## Частые вопросы на собеседовании

**Q1. Зачем нужен `functools.wraps` в декораторах?**
A: Без него функция-обёртка теряет метаданные оригинала: `__name__` станет `'wrapper'`, `__doc__` исчезнет, что ломает интроспекцию, документацию, отладку и некоторые фреймворки. `@wraps(func)` копирует эти атрибуты. Также даёт доступ к оригиналу через `__wrapped__`.

**Q2. Чем `lru_cache` отличается от `cache`?**
A: `cache` (Python 3.9+) — это `lru_cache(maxsize=None)`: неограниченный кэш без логики вытеснения, чуть быстрее. `lru_cache(maxsize=128)` ограничивает размер и вытесняет давно не используемые записи (LRU).

**Q3. Какие ограничения у `lru_cache`/`cache` по аргументам?**
A: Все аргументы должны быть **хешируемыми** (числа, строки, кортежи, frozenset). Список или словарь в аргументах вызовут `TypeError: unhashable type`. Также кэш держит ссылки на аргументы и результаты — возможны утечки памяти при `maxsize=None`.

**Q4. В чём опасность мемоизации функции с побочными эффектами?**
A: Кэш возвращает сохранённый результат, не вызывая функцию повторно. Если функция делает запрос к БД, читает файл, генерирует случайное число или зависит от времени — кэшированный результат «застрянет» и будет неверным. Мемоизировать стоит только чистые (pure) функции.

**Q5. Чем `cached_property` отличается от `property`?**
A: `property` вычисляется при КАЖДОМ обращении. `cached_property` вычисляется один раз, результат сохраняется в `__dict__` экземпляра и возвращается при последующих обращениях. Подходит для дорогих вычислений, чьё значение не меняется. Можно сбросить через `del obj.attr`. Не работает с `__slots__` (нужен `__dict__`).

**Q6. Что делает `reduce` и когда его НЕ стоит использовать?**
A: `reduce` сворачивает последовательность в одно значение, применяя бинарную функцию накопительно. Не стоит использовать там, где есть встроенные `sum`, `max`, `min`, `any`, `all` — они читаемее и быстрее. `reduce` оправдан для нестандартных свёрток.

**Q7. Как работает `partial` и чем отличается от lambda?**
A: `partial(f, x)` фиксирует часть аргументов и возвращает новый вызываемый объект. По сравнению с `lambda`: `partial` быстрее, хранит `.func/.args/.keywords` (интроспекция), лучше сериализуется (picklable), не создаёт замыкание. Lambda гибче для сложной логики.

**Q8. Как `total_ordering` экономит код?**
A: Достаточно определить `__eq__` и один оператор сравнения (например `__lt__`), а `total_ordering` сгенерирует остальные три. Минус — производительность чуть ниже, чем у вручную написанных всех методов.

**Q9. Что такое `singledispatch` и чем отличается от обычного `if isinstance`?**
A: `singledispatch` реализует одиночную диспетчеризацию (перегрузку) по типу первого аргумента: регистрируем отдельные реализации для типов. Это чище и расширяемее, чем длинная цепочка `if isinstance(...)`: новые типы добавляются регистрацией извне, без изменения исходной функции. Диспетчеризация идёт по MRO (учитывает наследование).

**Q10. Можно ли использовать `lru_cache` на методах класса? Есть ли подвох?**
A: Можно, но `self` становится частью ключа кэша, значит экземпляры должны быть хешируемыми, и кэш будет держать ссылку на `self` — это мешает сборке мусора (утечка памяти, экземпляры не освобождаются). Для кэширования на экземпляр лучше `cached_property` или `lru_cache` на отдельной функции.

**Q11. Как проверить статистику кэша и очистить его?**
A: У задекорированной `lru_cache` функции есть `.cache_info()` (возвращает hits/misses/maxsize/currsize) и `.cache_clear()` для сброса.

**Q12. Что делает параметр `typed=True` в `lru_cache`?**
A: При `typed=True` аргументы разных типов с равными значениями кэшируются раздельно: `f(3)` и `f(3.0)` будут двумя записями. При `typed=False` (по умолчанию) — одной.

**Q13. Как зарегистрировать одну реализацию `singledispatch` для нескольких типов?**
A: Складывать декораторы register:
```python
@func.register(int)
@func.register(float)
def _(arg): ...
```

**Q14. Эквивалентны ли `@cache` и `@cache()` ?**
A: `cache` применяется без скобок: `@cache`. А `lru_cache` можно и со скобками `@lru_cache()`, и с параметрами `@lru_cache(maxsize=100)`. С Python 3.8 `@lru_cache` без скобок тоже работает.

## Подводные камни (gotchas)

1. **Мемоизация нечистых функций** — кэш «замораживает» первый результат; функции с I/O, случайностью, временем дадут устаревшие данные.

2. **Нехешируемые аргументы** — список/словарь/множество в аргументах кэшируемой функции -> `TypeError`. Передавайте кортежи/frozenset.

3. **`lru_cache(maxsize=None)` -> утечка памяти** — кэш растёт неограниченно. Для долгоживущих процессов ограничивайте `maxsize`.

4. **`lru_cache` на методах держит `self`** — экземпляры не собираются GC, пока живёт кэш. Утечка памяти.

5. **Забыли `@wraps`** — обёртка теряет имя/docstring, ломая интроспекцию и некоторые фреймворки.

6. **`cached_property` и `__slots__`** — не работает без `__dict__`. Также `cached_property` не потокобезопасен по умолчанию (в старых версиях мог вычислять дважды при гонке).

7. **`reduce` без `initializer` на пустом итерируемом** -> `TypeError: reduce() of empty iterable with no initial value`.

8. **`partial` фиксирует значение в момент создания** — если передать изменяемый объект, он разделяется между вызовами.

9. **`singledispatch` диспетчеризует ТОЛЬКО по первому аргументу** — для методов используйте `singledispatchmethod`, для нескольких аргументов он не подходит.

10. **`total_ordering` требует определённого `__eq__`** — иначе сравнения будут некорректны; и он медленнее ручных реализаций.

## Лучшие практики

- Кэшируйте только **чистые функции** (без побочных эффектов, детерминированные).
- В долгоживущих процессах задавайте конечный `maxsize` у `lru_cache`, избегайте `maxsize=None` для растущих входов.
- Всегда используйте `@wraps` при написании декораторов.
- Для дорогих, неизменяемых вычислений на экземпляре — `cached_property` вместо ручного кэша в `__init__`.
- Предпочитайте встроенные `sum/max/min/any/all` вместо `reduce`, где возможно.
- `partial` предпочтительнее lambda для простой фиксации аргументов (быстрее, интроспектируемо, picklable).
- Используйте `singledispatch` вместо длинных цепочек `isinstance` для расширяемой типовой логики.
- Не вешайте `lru_cache` на методы экземпляра без необходимости — берегитесь утечек; кэшируйте на уровне функции.
- Проверяйте эффективность кэша через `.cache_info()` (соотношение hits/misses).

## Шпаргалка

```python
from functools import (reduce, partial, lru_cache, cache, cached_property,
                       wraps, total_ordering, singledispatch, partialmethod)
import operator

# --- reduce: свёртка в одно значение ---
reduce(operator.add, [1,2,3,4])        # 10
reduce(operator.mul, nums, 1)          # произведение с initializer

# --- partial: фиксация аргументов ---
square = partial(pow, exp=2)           # ...
from_bin = partial(int, base=2)        # from_bin('1010') -> 10

# --- мемоизация ---
@lru_cache(maxsize=128)                # ограниченный LRU-кэш
def f(n): ...
@lru_cache(maxsize=None, typed=True)   # неогранич., раздельно по типам
@cache                                 # == lru_cache(maxsize=None), 3.9+
f.cache_info()                         # hits/misses/maxsize/currsize
f.cache_clear()                        # очистить кэш

# --- кэширующее свойство ---
class C:
    @cached_property
    def heavy(self): ...               # вычислится один раз; del obj.heavy -> сброс

# --- декоратор по канону ---
def deco(func):
    @wraps(func)                       # сохранить __name__/__doc__/__wrapped__
    def wrapper(*a, **kw):
        return func(*a, **kw)
    return wrapper

# --- авто-сравнения ---
@total_ordering
class V:
    def __eq__(self, o): ...
    def __lt__(self, o): ...           # остальные сгенерируются

# --- перегрузка по типу ---
@singledispatch
def handle(x): ...                     # по умолчанию
@handle.register
def _(x: int): ...
@handle.register(str)
def _(x): ...

# --- метод с фиксированным аргументом ---
class S:
    def set(self, v): self.v = v
    on  = partialmethod(set, True)
    off = partialmethod(set, False)
```
