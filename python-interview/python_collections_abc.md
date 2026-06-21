# collections.abc — подготовка к собеседованию

## Что это и зачем

`collections.abc` — это модуль стандартной библиотеки Python, содержащий **абстрактные базовые классы (Abstract Base Classes, ABC)** для контейнеров и протоколов. Они формализуют «утиную типизацию»: вместо вопроса «есть ли у объекта метод `__iter__`?» мы спрашиваем «является ли объект `Iterable`?».

Зачем это нужно:

1. **Стандартизация протоколов.** ABC описывают, какие методы должен реализовать объект, чтобы считаться итерируемым, последовательностью, отображением и т. д. Это контракт.
2. **Миксины «бесплатно».** Если унаследоваться от ABC и реализовать минимальный набор абстрактных методов, остальные методы вы получаете автоматически. Например, для `Sequence` достаточно `__getitem__` и `__len__`, а `__contains__`, `__iter__`, `__reversed__`, `index`, `count` приходят как готовые миксины.
3. **`isinstance`/`issubclass` проверки.** Можно проверять не конкретный тип, а способность объекта: `isinstance(x, Iterable)`, `isinstance(x, Hashable)`.
4. **Структурная типизация через `__subclasshook__`.** Некоторые ABC (например, `Iterable`, `Hashable`, `Sized`) распознают объект как свой подкласс, если у него просто есть нужные методы — без явного наследования.
5. **Виртуальная регистрация.** Через `ABC.register(MyClass)` можно объявить класс подклассом ABC, не наследуясь от него.

Важно: модуль `collections.abc` нужно отличать от исторического `collections` — раньше эти классы лежали прямо в `collections`, но начиная с Python 3.3 они в `collections.abc`, а с Python 3.10 старые алиасы удалены.

```python
from collections.abc import Iterable, Sized, Hashable

# Любой list — итерируемый, имеет размер и НЕ хешируемый
print(isinstance([1, 2, 3], Iterable))   # => True
print(isinstance([1, 2, 3], Sized))      # => True
print(isinstance([1, 2, 3], Hashable))   # => False (списки не хешируемы)

# Кортеж из хешируемых элементов — хешируемый
print(isinstance((1, 2, 3), Hashable))   # => True
print(isinstance("строка", Iterable))    # => True
print(isinstance(42, Iterable))          # => False (int не итерируемый)
```

## Ключевые концепции

### Абстрактный базовый класс (ABC)

ABC — это класс, который нельзя инстанцировать напрямую, если в нём остались нереализованные абстрактные методы. Он задаёт интерфейс. Технически ABC создаются с метаклассом `ABCMeta` (или наследованием от `abc.ABC`), а абстрактные методы помечаются декоратором `@abstractmethod`.

```python
from abc import ABC, abstractmethod

class Repository(ABC):
    @abstractmethod
    def get(self, id): ...

try:
    Repository()  # попытка создать экземпляр
except TypeError as e:
    print(e)  # => Can't instantiate abstract class Repository ... abstract method get
```

### Абстрактные методы vs миксины

В каждом ABC из `collections.abc` есть две группы методов:

- **Абстрактные методы** — вы ОБЯЗАНЫ их реализовать в своём классе.
- **Миксин-методы** — реализованы в самом ABC поверх абстрактных. Вы получаете их даром.

Это ключевая идея: реализуй минимум, получи максимум.

### Структурная типизация и `__subclasshook__`

Некоторые ABC переопределяют `__subclasshook__`, чтобы `issubclass`/`isinstance` возвращали `True` для любого класса с нужными методами, даже без наследования. Это называется **структурной (или duck) типизацией на уровне ABC**.

```python
from collections.abc import Iterable

class MyOwn:
    def __iter__(self):
        return iter([1, 2, 3])

# Мы НЕ наследовались от Iterable, но...
print(issubclass(MyOwn, Iterable))   # => True (благодаря __subclasshook__)
print(isinstance(MyOwn(), Iterable)) # => True
```

### Виртуальные подклассы и `register()`

Можно явно зарегистрировать класс как «виртуальный подкласс» ABC. Тогда `isinstance`/`issubclass` дадут `True`, но **миксины наследоваться НЕ будут** и проверки реализации методов не произойдёт.

```python
from collections.abc import Iterable

class Strange:
    pass

Iterable.register(Strange)
print(issubclass(Strange, Iterable))  # => True
print(isinstance(Strange(), Iterable))# => True
# но Strange() нельзя итерировать — register ничего не проверяет!
```

### Иерархия ABC

Классы образуют иерархию наследования. Например:

```
Container ─┐
Iterable  ─┼─→ Collection ─→ Sequence ─→ MutableSequence
Sized     ─┘                 Mapping   ─→ MutableMapping
                             Set       ─→ MutableSet

Iterable  ─→ Iterator ─→ Generator
```

`Collection` объединяет `Container`, `Iterable`, `Sized` — это база для `Sequence`, `Mapping`, `Set`.

## Основные функции/классы/методы

### Таблица ABC: абстрактные методы и миксины

| ABC | Наследует | Абстрактные методы (нужно реализовать) | Миксины (получаете бесплатно) |
|-----|-----------|----------------------------------------|-------------------------------|
| `Hashable` | — | `__hash__` | — |
| `Sized` | — | `__len__` | — |
| `Callable` | — | `__call__` | — |
| `Container` | — | `__contains__` | — |
| `Iterable` | — | `__iter__` | — |
| `Iterator` | `Iterable` | `__next__` | `__iter__` |
| `Generator` | `Iterator` | `send`, `throw` | `close`, `__iter__`, `__next__` |
| `Reversible` | `Iterable` | `__reversed__` | — |
| `Collection` | `Sized`, `Iterable`, `Container` | `__contains__`, `__iter__`, `__len__` | — |
| `Sequence` | `Reversible`, `Collection` | `__getitem__`, `__len__` | `__contains__`, `__iter__`, `__reversed__`, `index`, `count` |
| `MutableSequence` | `Sequence` | `__getitem__`, `__setitem__`, `__delitem__`, `__len__`, `insert` | наследует от Sequence + `append`, `reverse`, `extend`, `pop`, `remove`, `__iadd__` |
| `Set` | `Collection` | `__contains__`, `__iter__`, `__len__` | `__le__`, `__lt__`, `__eq__`, `__ne__`, `__gt__`, `__ge__`, `__and__`, `__or__`, `__sub__`, `__xor__`, `isdisjoint` |
| `MutableSet` | `Set` | `__contains__`, `__iter__`, `__len__`, `add`, `discard` | наследует от Set + `clear`, `pop`, `remove`, `__ior__`, `__iand__`, `__ixor__`, `__isub__` |
| `Mapping` | `Collection` | `__getitem__`, `__len__`, `__iter__` | `__contains__`, `keys`, `items`, `values`, `get`, `__eq__`, `__ne__` |
| `MutableMapping` | `Mapping` | `__getitem__`, `__setitem__`, `__delitem__`, `__iter__`, `__len__` | наследует от Mapping + `pop`, `popitem`, `clear`, `update`, `setdefault` |

Также есть представления: `MappingView`, `KeysView`, `ItemsView`, `ValuesView`, и `Awaitable`, `Coroutine`, `AsyncIterable`, `AsyncIterator`, `AsyncGenerator` для async-кода.

### `Iterable` и `Iterator`: протокол итерации

`Iterable` — у объекта есть `__iter__`, возвращающий итератор. `Iterator` — у объекта есть `__next__` (и `__iter__`, возвращающий `self`).

```python
from collections.abc import Iterator

class CountDown(Iterator):
    """Итератор обратного отсчёта. Достаточно реализовать __next__."""
    def __init__(self, start):
        self._current = start

    def __next__(self):
        if self._current <= 0:
            raise StopIteration       # сигнал окончания итерации
        self._current -= 1
        return self._current + 1

# __iter__ -> self мы получили как миксин от Iterator
cd = CountDown(3)
print(list(cd))   # => [3, 2, 1]
print(iter(cd) is cd)  # => True (итератор возвращает сам себя)
```

### `Container`: оператор `in`

```python
from collections.abc import Container

class EvenNumbers(Container):
    """Бесконечный «контейнер» чётных чисел — хранить ничего не нужно."""
    def __contains__(self, item):
        return isinstance(item, int) and item % 2 == 0

evens = EvenNumbers()
print(4 in evens)   # => True
print(7 in evens)   # => False
print(isinstance(evens, Container))  # => True
```

### `Hashable` и `Sized`

```python
from collections.abc import Hashable, Sized

class Point(Hashable):
    __slots__ = ("x", "y")
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __hash__(self):
        return hash((self.x, self.y))
    def __eq__(self, other):
        return isinstance(other, Point) and (self.x, self.y) == (other.x, other.y)

p = Point(1, 2)
print(hash(p) == hash(Point(1, 2)))   # => True
s = {Point(1, 2), Point(1, 2), Point(3, 4)}
print(len(s))   # => 2 (дубликаты схлопнулись)
```

### Собственный `Sequence` — реализуем минимум, получаем максимум

Достаточно реализовать `__getitem__` и `__len__`, и вы автоматически получаете `__contains__`, `__iter__`, `__reversed__`, `index`, `count`.

```python
from collections.abc import Sequence

class Range2(Sequence):
    """Упрощённый аналог range, демонстрирует миксины Sequence."""
    def __init__(self, start, stop):
        self._start = start
        self._stop = stop

    def __len__(self):
        return max(0, self._stop - self._start)

    def __getitem__(self, index):
        if isinstance(index, slice):
            # поддержка срезов: вернём список
            return [self[i] for i in range(*index.indices(len(self)))]
        if index < 0:
            index += len(self)
        if not 0 <= index < len(self):
            raise IndexError("index out of range")
        return self._start + index

r = Range2(10, 15)            # 10, 11, 12, 13, 14
print(len(r))                 # => 5
print(r[0], r[-1])            # => 10 14
print(list(r))                # => [10, 11, 12, 13, 14]  (миксин __iter__)
print(12 in r)                # => True                  (миксин __contains__)
print(r.index(13))            # => 3                      (миксин index)
print(r.count(11))            # => 1                      (миксин count)
print(list(reversed(r)))      # => [14, 13, 12, 11, 10]   (миксин __reversed__)
```

### Собственный `Mapping` и `MutableMapping`

`Mapping` (только чтение): реализуйте `__getitem__`, `__len__`, `__iter__` — получите `__contains__`, `keys`, `items`, `values`, `get`, `__eq__`, `__ne__`.

```python
from collections.abc import Mapping, MutableMapping

class FrozenDict(Mapping):
    """Неизменяемое отображение."""
    def __init__(self, data):
        self._data = dict(data)
    def __getitem__(self, key):
        return self._data[key]
    def __iter__(self):
        return iter(self._data)
    def __len__(self):
        return len(self._data)

fd = FrozenDict({"a": 1, "b": 2})
print(fd["a"])            # => 1
print("b" in fd)          # => True            (миксин __contains__)
print(list(fd.keys()))    # => ['a', 'b']      (миксин keys)
print(fd.get("z", 0))     # => 0               (миксин get)
print(dict(fd.items()))   # => {'a': 1, 'b': 2}(миксин items)
```

`MutableMapping` добавляет запись: реализуйте `__setitem__`, `__delitem__` (плюс те три) — получите `pop`, `popitem`, `clear`, `update`, `setdefault`.

```python
class CaseInsensitiveDict(MutableMapping):
    """Словарь с ключами без учёта регистра."""
    def __init__(self):
        self._store = {}
    def _norm(self, key):
        return key.lower() if isinstance(key, str) else key
    def __getitem__(self, key):
        return self._store[self._norm(key)]
    def __setitem__(self, key, value):
        self._store[self._norm(key)] = value
    def __delitem__(self, key):
        del self._store[self._norm(key)]
    def __iter__(self):
        return iter(self._store)
    def __len__(self):
        return len(self._store)

d = CaseInsensitiveDict()
d["Content-Type"] = "json"
print(d["content-type"])     # => json
d.update({"X-Token": "abc"}) # миксин update работает
print(d.setdefault("Host", "localhost"))  # => localhost (миксин setdefault)
print(len(d))                # => 3
```

## Частые вопросы на собеседовании

**Q1: Зачем вообще нужны абстрактные базовые классы, если в Python есть утиная типизация?**

ABC формализуют утиную типизацию и дают три преимущества. Во-первых, **контракт**: ABC явно перечисляет, какие методы обязательны, и Python не даст создать экземпляр класса с нереализованными абстрактными методами — ошибка возникает рано, при инстанцировании, а не где-то глубоко в коде. Во-вторых, **переиспользование кода через миксины**: реализовав минимум методов, вы получаете десятки производных бесплатно. В-третьих, **возможность проверки через `isinstance(x, ABC)`** по способности, а не по конкретному типу. То есть ABC — это утиная типизация плюс инфраструктура.

---

**Q2: В чём разница между `Iterable` и `Iterator`?**

`Iterable` — объект, который можно перебрать: у него есть метод `__iter__()`, возвращающий итератор. `Iterator` — объект, который ведёт сам перебор: у него есть `__next__()`, выдающий следующий элемент или бросающий `StopIteration`, а также `__iter__()`, возвращающий `self`.

Ключевые следствия:
- Каждый `Iterator` является `Iterable`, но не наоборот. `list` — итерируемый, но не итератор (у него нет `__next__`, и `iter(lst) is lst` даёт `False`).
- `Iterable` можно перебирать многократно (каждый раз `iter()` создаёт новый итератор). `Iterator` обычно одноразовый — после исчерпания он пуст.

```python
from collections.abc import Iterable, Iterator
lst = [1, 2, 3]
print(isinstance(lst, Iterable))   # => True
print(isinstance(lst, Iterator))   # => False
it = iter(lst)
print(isinstance(it, Iterator))    # => True
print(list(it), list(it))          # => [1, 2, 3] []  (одноразовость)
```

---

**Q3: Опишите протокол итератора. Что произойдёт, если `__next__` не бросит `StopIteration`?**

Протокол: `iter(obj)` вызывает `obj.__iter__()` и получает итератор; затем цикл многократно вызывает `next(it)` -> `it.__next__()`, пока тот не бросит `StopIteration` — это сигнал «элементы кончились», который `for` ловит и завершает цикл штатно. `StopIteration` намеренно НЕ распространяется наружу.

Если `__next__` никогда не бросает `StopIteration` (и всегда что-то возвращает), итератор становится бесконечным — `for`, `list()`, `sum()` зависнут навсегда. Это легитимный приём для бесконечных последовательностей (например, `itertools.count`), но потребитель обязан ограничивать перебор (`break`, `itertools.islice`).

```python
from collections.abc import Iterator
import itertools

class Naturals(Iterator):
    def __init__(self):
        self._n = 0
    def __next__(self):
        self._n += 1
        return self._n   # никогда не StopIteration -> бесконечен

print(list(itertools.islice(Naturals(), 5)))  # => [1, 2, 3, 4, 5]
```

---

**Q4: Я реализую `Sequence`. Какие методы обязан написать сам, а какие получу бесплатно?**

Обязательны только `__getitem__` и `__len__`. Бесплатно (как миксины) приходят: `__contains__` (оператор `in`), `__iter__` (перебор через индексы 0, 1, 2…), `__reversed__`, `index`, `count`. То есть с двумя методами объект становится полноценной последовательностью.

Нюанс: миксин `__iter__` у `Sequence` реализован через последовательный доступ по целым индексам начиная с 0 и до `IndexError`. Поэтому ваш `__getitem__` должен корректно бросать `IndexError` за границами — иначе перебор не остановится. Также для нормальной работы желательно поддерживать срезы и отрицательные индексы.

---

**Q5: Чем `Mapping` отличается от `MutableMapping`? Когда какой выбрать?**

`Mapping` — отображение только для чтения: абстрактные `__getitem__`, `__len__`, `__iter__`; даёт `keys`, `items`, `values`, `get`, `__contains__`, `__eq__`. `MutableMapping` добавляет запись и удаление: дополнительно абстрактные `__setitem__`, `__delitem__`; даёт `pop`, `popitem`, `clear`, `update`, `setdefault`.

Выбор: если у вас неизменяемая или read-only структура (например, обёртка над конфигом), наследуйтесь от `Mapping` — это и сигнал намерения, и защита (нет случайной мутации). Если нужна полноценная замена `dict` со вставкой/удалением — `MutableMapping`. Наследование от `MutableMapping` особенно ценно: реализовав 5 методов, вы получаете `update`, `setdefault` и прочее, которые легко реализовать с ошибками вручную.

---

**Q6: Что такое виртуальный подкласс и метод `register()`? В чём отличие от обычного наследования?**

`ABC.register(Cls)` объявляет `Cls` виртуальным подклассом ABC. После этого `issubclass(Cls, ABC)` и `isinstance(obj, ABC)` дают `True`, но:
- Миксин-методы **НЕ** наследуются — Python не вставляет ABC в MRO `Cls`.
- **Никакой проверки** реализации методов не происходит — регистрация «верит на слово».

Обычное наследование, наоборот, вставляет ABC в MRO (вы получаете миксины) и проверяет наличие абстрактных методов при инстанцировании. `register` полезен, когда нельзя менять чужой класс или нужно объявить совместимость задним числом. Именно через `register` встроенные `tuple`, `str`, `list` зарегистрированы как `Sequence`, а `dict` — как `MutableMapping`.

```python
from collections.abc import Sequence
print(issubclass(tuple, Sequence))  # => True (зарегистрирован, не наследник)
print(Sequence in tuple.__mro__)    # => False (его нет в MRO)
```

---

**Q7: Как работает `__subclasshook__` и что такое структурная типизация на уровне ABC?**

`__subclasshook__(cls, C)` — classmethod, который `ABCMeta.__subclasscheck__` вызывает при `issubclass`. Если он вернёт `True`/`False`, это и есть ответ; `NotImplemented` — значит «решай обычным способом». ABC вроде `Iterable`, `Hashable`, `Sized`, `Container`, `Callable` переопределяют его так: «если у класса есть нужный метод (в нём самом или родителях), считай его подклассом». Это и есть структурная типизация — соответствие по структуре (наличию методов), без явного наследования.

```python
from collections.abc import Sized
class Box:
    def __len__(self): return 0
# Не наследовались от Sized, но __subclasshook__ видит __len__
print(issubclass(Box, Sized))   # => True
```

Важно: структурная проверка у этих ABC «поверхностная» — проверяется только наличие метода по имени, не сигнатура. И работает она лишь для простых одно-методных ABC; для `Sequence`/`Mapping` (много методов) `__subclasshook__` НЕ срабатывает — нужно явное наследование или `register`.

---

**Q8: Почему `isinstance(x, Iterable)` иногда возвращает `False` для объекта, который на деле перебирается?**

Потому что в Python есть «старый» протокол итерации через `__getitem__` (последовательный доступ по индексам 0, 1, …). Объект только с `__getitem__`, но без `__iter__`, перебираем в `for`, однако `isinstance(x, Iterable)` даст `False`, так как `Iterable.__subclasshook__` проверяет именно наличие `__iter__`.

```python
from collections.abc import Iterable
class OldStyle:
    def __getitem__(self, i):
        if i > 2: raise IndexError
        return i * 10
o = OldStyle()
print(list(o))                    # => [0, 10, 20] (перебирается!)
print(isinstance(o, Iterable))    # => False (нет __iter__)
```

Вывод: `isinstance(x, Iterable)` — не стопроцентная гарантия перебираемости; надёжнее обернуть в `iter(x)` и поймать `TypeError`.

---

**Q9: В чём разница между `collections.abc` ABC и `typing.Protocol`?**

`collections.abc` ABC — это **номинальная** типизация (для большинства, кроме одно-методных через `__subclasshook__`): нужно либо наследоваться, либо вызвать `register`. Они работают в рантайме (`isinstance`) и дают миксины.

`typing.Protocol` (PEP 544) — это **структурная** типизация для статических чекеров (mypy/pyright): класс совместим с протоколом, если у него есть нужные методы, без наследования и регистрации. По умолчанию протоколы работают только статически; чтобы их можно было проверять через `isinstance`, нужен декоратор `@runtime_checkable` (и он проверяет лишь наличие методов, не сигнатуры). Протоколы НЕ дают миксинов.

Когда что: `collections.abc` — когда нужны готовые миксины и рантайм-проверки контейнерных протоколов. `Protocol` — когда нужна гибкая структурная совместимость для аннотаций без навязывания иерархии наследования (например, «любой объект с методом `read`»).

---

**Q10: Какие требования к `Hashable` и почему важна связка `__hash__`/`__eq__`?**

Чтобы объект был хешируемым, он должен иметь `__hash__`, возвращающий int, и быть «иммутабельным по идентичности» для целей хеша. Контракт: если `a == b`, то `hash(a) == hash(b)`. Поэтому `__hash__` и `__eq__` определяют вместе и согласованно.

Подводный камень: если вы определяете `__eq__` в классе, Python автоматически устанавливает `__hash__ = None`, делая экземпляры нехешируемыми — нужно явно задать `__hash__`. И наоборот, изменяемые объекты (как `list`) специально нехешируемы, чтобы не ломать структуры на хешах при мутации ключа.

```python
class A:
    def __init__(self, v): self.v = v
    def __eq__(self, o): return self.v == o.v
print(A.__hash__)  # => None  (определили __eq__ -> __hash__ обнулён)
# {A(1)}  -> TypeError: unhashable type: 'A'
```

---

**Q11: Что делает `Generator` ABC и чем генератор отличается от обычного итератора?**

`Generator` наследует `Iterator` и добавляет методы двусторонней коммуникации: абстрактные `send(value)` и `throw(...)`, а также миксин `close()`. Обычный итератор только выдаёт значения через `__next__`; генератор умеет ещё **принимать** значения через `send()`, в него можно бросить исключение через `throw()` и корректно завершить через `close()`. Реальные генераторные функции (с `yield`) автоматически реализуют этот протокол и регистрируются как `Generator`.

```python
from collections.abc import Generator
def gen():
    while True:
        x = yield
        print("получил:", x)
g = gen()
print(isinstance(g, Generator))  # => True
next(g)            # запуск до первого yield
g.send("привет")   # => получил: привет
g.close()
```

---

**Q12: Можно ли наследоваться от нескольких ABC сразу и какие тут подводные камни?**

Да, это нормально — например, класс может быть и `Sized`, и `Callable`. Подводные камни связаны с MRO и абстрактными методами: при множественном наследовании нужно реализовать объединение всех абстрактных методов всех баз, иначе класс останется абстрактным. Также возможны конфликты миксинов (одно имя метода в разных ABC) — порядок баз в MRO решает, чья реализация победит. На практике для контейнеров обычно достаточно одного «главного» ABC (`Sequence`/`Mapping`/`Set`), так как они уже агрегируют `Sized`, `Iterable`, `Container` через `Collection`.

## Подводные камни (gotchas)

### 1. `register()` ничего не проверяет

Регистрация делает `isinstance` истинным, но объект может вообще не уметь то, что обещает ABC.

```python
from collections.abc import Iterable
class Fake: pass
Iterable.register(Fake)
print(isinstance(Fake(), Iterable))  # => True
# но: list(Fake()) -> TypeError, объект не итерируемый
```

### 2. Определение `__eq__` обнуляет `__hash__`

```python
class Money:
    def __init__(self, amount): self.amount = amount
    def __eq__(self, o): return self.amount == o.amount
print(Money.__hash__)   # => None  (экземпляры стали нехешируемыми)
# Чтобы исправить — добавьте __hash__ явно:
class Money2(Money):
    def __hash__(self): return hash(self.amount)
print(hash(Money2(5)) == hash(Money2(5)))  # => True
```

### 3. Миксин `__iter__` у `Sequence` требует корректного `IndexError`

Если `__getitem__` за границами не бросает `IndexError` (например, бросает `KeyError` или возвращает `None`), перебор и `in` зациклятся или сломаются.

```python
from collections.abc import Sequence
class Bad(Sequence):
    def __len__(self): return 3
    def __getitem__(self, i): return i   # никогда не IndexError!
# list(Bad()) -> бесконечный цикл / неправильное поведение
```

### 4. `isinstance(x, Iterable)` не гарантирует перебор через `__getitem__`-объекты

См. Q8 — объект со «старым» протоколом (`__getitem__` без `__iter__`) перебирается, но `Iterable` его не распознаёт. Для надёжной проверки используйте `iter(x)` в `try/except TypeError`.

### 5. `__subclasshook__` работает только для простых ABC

`issubclass(MyClass, Sequence)` НЕ станет `True` автоматически только из-за наличия `__getitem__`/`__len__` — нужно явное наследование или `register`. Структурная магия есть только у `Iterable`, `Iterator`, `Hashable`, `Sized`, `Container`, `Callable`, `Reversible` и подобных одно-методных.

```python
from collections.abc import Sequence
class Seqish:
    def __getitem__(self, i): return i
    def __len__(self): return 0
print(issubclass(Seqish, Sequence))  # => False (нет __subclasshook__ для Sequence)
```

### 6. Импорт из `collections` вместо `collections.abc`

В Python 3.10+ `collections.Iterable` удалён — только `collections.abc.Iterable`. Старый код упадёт с `AttributeError`.

### 7. `bytes`/`str` — это `Sequence`, и `1 in "123"` ведёт себя нетривиально

`str` — последовательность символов, поэтому `in` для строки проверяет подстроку, а не элемент: `"23" in "123"` -> `True`. Это особенность миксина `__contains__`, переопределённого для строк.

## Лучшие практики

1. **Наследуйтесь от подходящего ABC, а не реализуйте протокол «на глаз».** Это документирует намерение, даёт миксины и раннюю проверку при инстанцировании.

2. **Реализуйте минимально необходимый набор абстрактных методов** и доверяйте миксинам. Не переписывайте `update`, `index`, `get` вручную без причины — реализации в ABC корректны и протестированы.

3. **Для read-only структур выбирайте неизменяемые ABC** (`Mapping`, `Sequence`, `Set`), а не их Mutable-версии — это и сигнал, и защита.

4. **Аннотируйте функции абстрактными типами, а не конкретными.** Принимайте `Iterable`/`Mapping`/`Sequence` вместо `list`/`dict` — функция станет совместимой с большим числом типов («принимай широко, возвращай конкретно»).

```python
from collections.abc import Iterable
def total(values: Iterable[int]) -> int:
    return sum(values)
print(total([1, 2, 3]))        # список
print(total((4, 5)))           # кортеж
print(total(range(3)))         # range
print(total(x for x in [1,1])) # генератор
# => 6, 9, 3, 2
```

5. **Согласуйте `__hash__` и `__eq__`.** Определили `__eq__` для value-объекта — добавьте совместимый `__hash__` (или явно сделайте объект нехешируемым, оставив `__hash__ = None`).

6. **Используйте `register()` осознанно** — только когда нельзя наследоваться (чужой/встроенный класс), и помните, что миксины и проверки при этом не работают.

7. **Для статической структурной типизации предпочитайте `typing.Protocol`,** а `collections.abc` — для рантайм-проверок и получения миксинов. Не путайте их роли.

8. **Проверяйте перебираемость через `iter()`/`try-except`,** а не только `isinstance(x, Iterable)`, если важна совместимость с `__getitem__`-объектами.

## Шпаргалка

```python
from collections.abc import (
    Hashable, Sized, Callable, Container, Iterable, Iterator, Generator,
    Reversible, Collection, Sequence, MutableSequence,
    Set, MutableSet, Mapping, MutableMapping,
)

# --- Минимальные методы для реализации ---
# Iterable          -> __iter__
# Iterator          -> __next__            (+ __iter__ как миксин)
# Container         -> __contains__
# Sized             -> __len__
# Hashable          -> __hash__
# Callable          -> __call__
# Sequence          -> __getitem__, __len__         (бесплатно: in, iter, reversed, index, count)
# MutableSequence   -> + __setitem__, __delitem__, insert  (бесплатно: append, pop, remove, extend...)
# Mapping           -> __getitem__, __iter__, __len__       (бесплатно: keys, items, values, get, in)
# MutableMapping    -> + __setitem__, __delitem__           (бесплатно: pop, update, setdefault, clear)
# Set               -> __contains__, __iter__, __len__      (бесплатно: |, &, -, ^, <=, isdisjoint)
# MutableSet        -> + add, discard                       (бесплатно: pop, clear, |=, &=...)

# --- Проверки isinstance ---
isinstance([], Iterable)        # => True
isinstance([], Iterator)        # => False
isinstance(iter([]), Iterator)  # => True
isinstance({}, Mapping)         # => True
isinstance({}, MutableMapping)  # => True
isinstance((), Hashable)        # => True
isinstance([], Hashable)        # => False
isinstance(len, Callable)       # => True

# --- Структурная типизация (__subclasshook__): работает для простых ABC ---
class S:
    def __len__(self): return 0
issubclass(S, Sized)            # => True  (без наследования)

# --- Виртуальная регистрация (миксины НЕ наследуются, проверок НЕТ) ---
Iterable.register(SomeClass)    # issubclass(SomeClass, Iterable) -> True

# --- Иерархия (упрощённо) ---
# Hashable, Sized, Callable, Container, Iterable        -- базовые одно-методные
# Iterable -> Iterator -> Generator
# Container + Iterable + Sized -> Collection
# Collection -> Sequence -> MutableSequence
# Collection -> Mapping  -> MutableMapping
# Collection -> Set      -> MutableSet

# --- Правила ---
# 1. a == b  => hash(a) == hash(b)
# 2. Определил __eq__ => __hash__ становится None (задай вручную при необходимости)
# 3. __getitem__ за границами -> IndexError (иначе сломается миксин __iter__)
# 4. register() не проверяет реализацию и не даёт миксины
# 5. isinstance(x, Iterable) не видит __getitem__-only объекты
# 6. Аннотируй параметры абстрактными типами (Iterable/Mapping), а не list/dict
```
