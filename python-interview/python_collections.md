# collections — подготовка к собеседованию

## Что это и зачем

`collections` — модуль стандартной библиотеки Python, который предоставляет специализированные альтернативы встроенным контейнерам `dict`, `list`, `tuple`. Эти структуры решают типовые задачи быстрее (по асимптотике), читабельнее (по коду) и безопаснее (по семантике), чем «ручные» решения на базовых типах.

Зачем это знать на собеседовании:

- Показывает знание **алгоритмической сложности**: `deque` даёт O(1) на обоих концах, тогда как `list.insert(0, x)` — O(n).
- Показывает идиоматичность кода: вместо `if key not in d: d[key] = []` пишут `defaultdict(list)`.
- Часто спрашивают про разницу `OrderedDict` и обычного `dict` после Python 3.7.
- `Counter` — стандартный инструмент для подсчёта частот (топ-N элементов, анаграммы, частотный анализ).

Краткая карта модуля:

| Класс            | Базируется на | Главная фишка                                           |
|------------------|---------------|---------------------------------------------------------|
| `namedtuple`     | `tuple`       | Доступ к полям по имени, неизменяемость                  |
| `deque`          | —             | O(1) добавление/удаление с обоих концов, `maxlen`       |
| `Counter`        | `dict`        | Подсчёт частот, мультимножество, арифметика              |
| `OrderedDict`    | `dict`        | Учёт порядка в `==`, `move_to_end`, `popitem(last)`     |
| `defaultdict`    | `dict`        | Автосоздание значений через `default_factory`           |
| `ChainMap`       | —             | Логическое объединение нескольких словарей без копирования |
| `UserDict`       | `dict`        | Удобное наследование для кастомных словарей              |
| `UserList`       | `list`        | Удобное наследование для кастомных списков               |
| `UserString`     | `str`         | Удобное наследование для кастомных строк                 |

```python
from collections import (
    namedtuple, deque, Counter, OrderedDict,
    defaultdict, ChainMap, UserDict, UserList, UserString,
)
```

## Ключевые концепции

1. **Неизменяемость vs изменяемость.** `namedtuple` неизменяем (как `tuple`), остальные структуры — изменяемые.
2. **Асимптотика контейнеров.** Главный смысл `deque` — O(1) на обоих концах против O(n) у `list` спереди. Нужно понимать, *почему*: `list` хранит элементы в непрерывном массиве, вставка в начало сдвигает все элементы.
3. **Фабрика значений.** `defaultdict` хранит вызываемый объект `default_factory`, который вызывается **без аргументов** при обращении к отсутствующему ключу.
4. **Представление, а не копия.** `ChainMap` не копирует словари — он хранит ссылки на них и ищет ключ по цепочке. Изменения в исходных словарях видны мгновенно.
5. **Мультимножество.** `Counter` — это словарь `{элемент: количество}`, на котором определена арифметика мультимножеств (`+`, `-`, `&`, `|`).
6. **Порядок в dict.** С Python 3.7 обычный `dict` гарантированно сохраняет порядок вставки. Поэтому `OrderedDict` нужен только ради дополнительных возможностей (`move_to_end`, порядок-зависимое сравнение, `popitem(last=False)`).
7. **Делегирование к содержимому.** `UserDict`/`UserList`/`UserString` хранят реальные данные в атрибуте (`.data`) и наследуются обычным образом, что устраняет проблемы прямого наследования от C-типов.

## Основные функции/классы/методы

### namedtuple

Фабрика, создающая подкласс `tuple` с именованными полями. Объект остаётся кортежем: индексируется, распаковывается, неизменяем и хешируем.

```python
from collections import namedtuple

# Определение типа. Поля можно передать строкой "x y" или списком ["x", "y"].
Point = namedtuple("Point", ["x", "y"])

p = Point(1, 2)
print(p)            # => Point(x=1, y=2)
print(p.x, p.y)     # => 1 2
print(p[0])         # => 1  (доступ по индексу всё ещё работает)

x, y = p            # распаковка как у обычного кортежа
print(x, y)         # => 1 2

print(isinstance(p, tuple))  # => True
```

Служебные атрибуты и методы (начинаются с `_`, чтобы не конфликтовать с именами полей):

```python
# _fields — кортеж имён полей
print(Point._fields)            # => ('x', 'y')

# _replace — возвращает НОВЫЙ экземпляр с заменёнными полями (namedtuple неизменяем!)
p2 = p._replace(y=99)
print(p2)                       # => Point(x=1, y=99)
print(p)                        # => Point(x=1, y=2)  (исходный не тронут)

# _asdict — превращает в обычный dict (с 3.8 это именно dict)
print(p._asdict())              # => {'x': 1, 'y': 2}

# _make — создаёт экземпляр из итерируемого (удобно при чтении CSV/строк)
row = [10, 20]
print(Point._make(row))         # => Point(x=10, y=20)
```

Значения по умолчанию через `defaults` (применяются к ПРАВЫМ полям):

```python
# defaults=[0] относится к последнему полю y; x остаётся обязательным
Point = namedtuple("Point", ["x", "y"], defaults=[0])
print(Point(5))                 # => Point(x=5, y=0)

# Два дефолта — для двух правых полей
Coord = namedtuple("Coord", "x y z", defaults=[0, 0])
print(Coord(1))                 # => Coord(x=1, y=0, z=0)
```

Альтернатива — `typing.NamedTuple` (классовый синтаксис + аннотации типов):

```python
from typing import NamedTuple

class Point(NamedTuple):
    x: int
    y: int = 0              # значение по умолчанию

    # Можно добавлять методы и свойства!
    def distance_sq(self) -> int:
        return self.x ** 2 + self.y ** 2

p = Point(3, 4)
print(p)                   # => Point(x=3, y=4)
print(p.distance_sq())     # => 25
print(Point(7))            # => Point(x=7, y=0)
print(Point.__annotations__)  # => {'x': <class 'int'>, 'y': <class 'int'>}
```

### deque

Двусторонняя очередь (double-ended queue). Добавление и удаление с **обоих** концов — O(1). Реализована как двусвязный список блоков.

```python
from collections import deque

d = deque([1, 2, 3])

# Операции справа (как у list)
d.append(4)          # [1, 2, 3, 4]
print(d.pop())       # => 4   ; d == deque([1, 2, 3])

# Операции слева — O(1), у list это было бы O(n)
d.appendleft(0)      # deque([0, 1, 2, 3])
print(d.popleft())   # => 0   ; d == deque([1, 2, 3])
print(d)             # => deque([1, 2, 3])

# extend / extendleft
d.extend([4, 5])         # deque([1, 2, 3, 4, 5])
d.extendleft([0, -1])    # ВНИМАНИЕ: добавляет по очереди слева -> порядок переворачивается
print(d)                 # => deque([-1, 0, 1, 2, 3, 4, 5])
```

`maxlen` — ограниченная очередь (кольцевой буфер): при переполнении элементы вытесняются с противоположного конца. Идеально для «последних N событий», скользящих окон.

```python
window = deque(maxlen=3)
for i in range(5):
    window.append(i)
    print(list(window))
# => [0]
# => [0, 1]
# => [0, 1, 2]
# => [1, 2, 3]   (0 вытеснен слева)
# => [2, 3, 4]   (1 вытеснен слева)
```

`rotate` — циклический сдвиг. Положительное число — вправо, отрицательное — влево.

```python
d = deque([1, 2, 3, 4, 5])
d.rotate(2)
print(d)             # => deque([4, 5, 1, 2, 3])
d.rotate(-2)
print(d)             # => deque([1, 2, 3, 4, 5])
```

Почему `deque` быстрее `list` спереди:

```python
# list.insert(0, x) и list.pop(0) — O(n): сдвигают все элементы
# deque.appendleft / popleft — O(1)
#
# НО: доступ по индексу в середине у deque — O(n),
#     у list — O(1). Выбирайте структуру под паттерн доступа.

# Потокобезопасность: append/appendleft/pop/popleft атомарны под GIL,
# поэтому deque часто используют как простую очередь между потоками.
```

### Counter

Подкласс `dict` для подсчёта. Значения — количества (могут быть нулевыми и отрицательными). Отсутствующий ключ даёт `0`, а не `KeyError`.

```python
from collections import Counter

c = Counter("abracadabra")
print(c)                  # => Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})
print(c["a"])             # => 5
print(c["z"])             # => 0   (нет KeyError для отсутствующих)

# Создание из разных источников
print(Counter([1, 1, 2, 3, 3, 3]))     # => Counter({3: 3, 1: 2, 2: 1})
print(Counter({'x': 4, 'y': 2}))        # => Counter({'x': 4, 'y': 2})
print(Counter(a=2, b=1))                # => Counter({'a': 2, 'b': 1})
```

`most_common` — N самых частых (по убыванию). Без аргумента — все.

```python
c = Counter("abracadabra")
print(c.most_common(2))    # => [('a', 5), ('b', 2)]
print(c.most_common())     # => [('a', 5), ('b', 2), ('r', 2), ('c', 1), ('d', 1)]
```

`elements` — итератор, разворачивающий счётчик обратно в элементы (количество <= 0 пропускается):

```python
c = Counter(a=3, b=1, c=0, d=-1)
print(sorted(c.elements()))   # => ['a', 'a', 'a', 'b']
```

`update` (прибавляет) и `subtract` (вычитает, допускает отрицательные/нулевые):

```python
c = Counter(a=3, b=1)
c.update(["a", "c", "c"])       # прибавить
print(c)                        # => Counter({'a': 4, 'c': 2, 'b': 1})

c.subtract({"a": 5})            # вычесть, может уйти в минус
print(c)                        # => Counter({'c': 2, 'b': 1, 'a': -1})
```

Арифметика мультимножеств. `+` и `-` отбрасывают неположительные результаты; `&` — минимум, `|` — максимум:

```python
a = Counter(x=3, y=1)
b = Counter(x=1, y=2, z=5)

print(a + b)    # => Counter({'z': 5, 'x': 4, 'y': 3})           сложение
print(a - b)    # => Counter({'x': 2})                            только положительные!
print(a & b)    # => Counter({'y': 1, 'x': 1})                    пересечение (min)
print(a | b)    # => Counter({'z': 5, 'x': 3, 'y': 2})            объединение (max)

# Унарные операторы: +c убирает неположительные, -c меняет знак
mixed = Counter(a=2, b=-3, c=0)
print(+mixed)   # => Counter({'a': 2})
print(-mixed)   # => Counter({'b': 3})

# total() (Python 3.10+) — сумма всех значений
print(Counter(a=2, b=3).total())   # => 5
```

### OrderedDict

Словарь, помнящий порядок вставки. С Python 3.7 обычный `dict` тоже это делает, но `OrderedDict` имеет важные отличия.

```python
from collections import OrderedDict

od = OrderedDict()
od["a"] = 1
od["b"] = 2
od["c"] = 3
print(od)        # => OrderedDict({'a': 1, 'b': 2, 'c': 3})
```

`move_to_end` — переместить ключ в конец (или в начало при `last=False`):

```python
od = OrderedDict(a=1, b=2, c=3)
od.move_to_end("a")
print(list(od))            # => ['b', 'c', 'a']
od.move_to_end("a", last=False)
print(list(od))            # => ['a', 'b', 'c']
```

`popitem(last=...)` — удалить с конца (LIFO) или с начала (FIFO):

```python
od = OrderedDict(a=1, b=2, c=3)
print(od.popitem(last=True))    # => ('c', 3)   с конца
print(od.popitem(last=False))   # => ('a', 1)   с начала
print(od)                       # => OrderedDict({'b': 2})
# Примечание: обычный dict.popitem() тоже есть, но только LIFO (last=True),
# параметр last НЕ поддерживается.
```

Главное отличие — **порядок-зависимое сравнение**:

```python
# Обычные dict равны при одинаковом содержимом, порядок не важен
print({'a': 1, 'b': 2} == {'b': 2, 'a': 1})       # => True

# OrderedDict при сравнении С OrderedDict учитывает порядок
print(OrderedDict(a=1, b=2) == OrderedDict(b=2, a=1))   # => False
print(OrderedDict(a=1, b=2) == OrderedDict(a=1, b=2))   # => True

# Но OrderedDict == dict сравнивается БЕЗ учёта порядка
print(OrderedDict(a=1, b=2) == {'b': 2, 'a': 1})        # => True
```

Когда `OrderedDict` всё ещё нужен: реализация LRU-кэша (через `move_to_end` + `popitem(last=False)`), очереди с FIFO-удалением, и код, где важно сравнение с учётом порядка.

### defaultdict

`dict`, который при обращении к **отсутствующему** ключу автоматически создаёт значение, вызывая `default_factory()` без аргументов.

```python
from collections import defaultdict

# Группировка: list как фабрика
dd = defaultdict(list)
dd["fruits"].append("apple")     # ключа нет -> создаётся [] -> .append
dd["fruits"].append("banana")
print(dd)        # => defaultdict(<class 'list'>, {'fruits': ['apple', 'banana']})

# Подсчёт: int как фабрика (int() == 0)
counts = defaultdict(int)
for ch in "mississippi":
    counts[ch] += 1
print(dict(counts))   # => {'m': 1, 'i': 4, 's': 4, 'p': 2}

# Множества: set как фабрика
graph = defaultdict(set)
graph["a"].add("b")
graph["a"].add("b")        # дубликат игнорируется
print(dict(graph))         # => {'a': {'b'}}
```

Важные нюансы фабрики:

```python
# default_factory можно задать вручную и поменять/обнулить
dd = defaultdict(int)
dd.default_factory = list
dd["x"].append(1)
print(dd["x"])             # => [1]

# Кастомная фабрика через lambda (вызывается БЕЗ аргументов)
dd = defaultdict(lambda: "N/A")
print(dd["missing"])       # => N/A

# ОПАСНО: само обращение dd[key] СОЗДАЁТ ключ. Используйте .get() для проверки.
dd = defaultdict(list)
_ = dd["ghost"]            # ключ 'ghost' теперь существует!
print(dict(dd))            # => {'ghost': []}

# default_factory=None -> ведёт себя как обычный dict (KeyError на отсутствующем)
dd = defaultdict(None)
try:
    dd["x"]
except KeyError as e:
    print("KeyError:", e)  # => KeyError: 'x'
```

Частый паттерн группировки по ключу:

```python
people = [("eng", "Alice"), ("sales", "Bob"), ("eng", "Carol")]
by_dept = defaultdict(list)
for dept, name in people:
    by_dept[dept].append(name)
print(dict(by_dept))   # => {'eng': ['Alice', 'Carol'], 'sales': ['Bob']}
```

### ChainMap

Объединяет несколько словарей в одно логическое отображение **без копирования**. Поиск идёт по цепочке слева направо — побеждает первый найденный.

```python
from collections import ChainMap

defaults = {"color": "red", "user": "guest"}
overrides = {"color": "blue"}

cfg = ChainMap(overrides, defaults)
print(cfg["color"])    # => blue   (найден в overrides — он первый)
print(cfg["user"])     # => guest  (нет в overrides -> ищем в defaults)

# maps — список словарей в порядке поиска
print(cfg.maps)        # => [{'color': 'blue'}, {'color': 'red', 'user': 'guest'}]
```

Записи и удаления затрагивают только **первый** словарь:

```python
cfg["color"] = "green"      # пишется в overrides (maps[0])
cfg["new"] = 1
print(overrides)            # => {'color': 'green', 'new': 1}
print(defaults)             # => {'color': 'red', 'user': 'guest'}  (не тронут)

# Удаление ключа, которого нет в первом словаре, -> KeyError,
# даже если он есть глубже по цепочке.
try:
    del cfg["user"]         # 'user' живёт в defaults, не в maps[0]
except KeyError as e:
    print("KeyError:", e)   # => KeyError: "Key not found in the first mapping: 'user'"
```

`new_child` и `parents` — управление слоями (типично для вложенных областей видимости/контекстов):

```python
base = {"a": 1}
chain = ChainMap(base)

# new_child добавляет НОВЫЙ пустой (или заданный) словарь в начало
scoped = chain.new_child({"a": 99, "b": 2})
print(scoped["a"])         # => 99  (новый слой переопределяет)
print(scoped["b"])         # => 2
print(chain["a"])          # => 1   (исходная цепочка не изменилась)

# parents — цепочка без первого слоя
print(scoped.parents["a"]) # => 1
```

Зачем нужно: конфигурации с приоритетами (CLI-аргументы > env > файл > дефолты), эмуляция областей видимости, наложение настроек без слияния словарей.

### UserDict / UserList / UserString

Обёртки-основы для **наследования**. Реальные данные хранятся в атрибуте `.data`, а класс наследуется обычным Python-способом. Это решает проблему прямого наследования от `dict`/`list`/`str` (C-типы), где встроенные методы могут **не вызывать** переопределённые `__setitem__`/`__getitem__`.

```python
# ПРОБЛЕМА с прямым наследованием от dict:
class UpperDict(dict):
    def __setitem__(self, key, value):
        super().__setitem__(key.upper(), value)

d = UpperDict()
d["a"] = 1
print(d)                  # => {'A': 1}   ок, __setitem__ сработал
# Но __init__ и update НЕ проходят через __setitem__:
d2 = UpperDict(b=2)
print(d2)                 # => {'b': 2}   ОШИБКА ожиданий — ключ не в верхнем регистре!
d2.update(c=3)
print(d2)                 # => {'b': 2, 'c': 3}  тоже мимо __setitem__
```

```python
# РЕШЕНИЕ через UserDict — все пути проходят через __setitem__:
from collections import UserDict

class UpperDict(UserDict):
    def __setitem__(self, key, value):
        super().__setitem__(key.upper(), value)

d = UpperDict(b=2)        # инициализация идёт через __setitem__
d["a"] = 1
d.update(c=3)
print(d.data)             # => {'B': 2, 'A': 1, 'C': 3}   все ключи в верхнем регистре!
```

```python
from collections import UserList, UserString

# UserList: список, запрещающий отрицательные значения
class PositiveList(UserList):
    def append(self, item):
        if item < 0:
            raise ValueError("только неотрицательные числа")
        super().append(item)

pl = PositiveList([1, 2])
pl.append(3)
print(pl.data)            # => [1, 2, 3]

# UserString: строка с дополнительным поведением
class ShoutString(UserString):
    def shout(self):
        return self.data.upper() + "!"

s = ShoutString("hello")
print(s.shout())          # => HELLO!
print(s.upper())          # => HELLO   (унаследованные str-методы работают)
print(len(s))             # => 5
```

Когда использовать: если нужно **переопределить базовое поведение** контейнера так, чтобы оно работало единообразно через все методы — берите `UserDict`/`UserList`/`UserString`. Если переопределять поведение не нужно (просто добавить методы) — обычно достаточно прямого наследования или композиции.

## Частые вопросы на собеседовании

**Q1. В чём практическая разница между `deque` и `list`? Когда что использовать?**

A. `list` — это динамический массив: элементы лежат в непрерывной памяти. Доступ по индексу — O(1), `append`/`pop` с конца — амортизированно O(1), но `insert(0, x)` и `pop(0)` — O(n) (сдвигаются все элементы). `deque` — двусвязный список блоков: `append`/`appendleft`/`pop`/`popleft` — все O(1), но доступ по индексу в середине — O(n). Вывод: для очередей и стеков с операциями на обоих концах, для скользящих окон (`maxlen`) — `deque`. Для произвольного индексного доступа и слайсинга — `list`.

**Q2. Нужен ли `OrderedDict`, если обычный `dict` с Python 3.7 сохраняет порядок?**

A. Для простого сохранения порядка — нет. Но `OrderedDict` всё ещё полезен из-за: (1) `move_to_end(key, last=...)` — нет аналога у `dict`; (2) `popitem(last=False)` — удаление с начала (FIFO), `dict.popitem()` умеет только LIFO; (3) **порядок-зависимое сравнение**: два `OrderedDict` равны только при совпадении порядка ключей, тогда как `dict` сравниваются без учёта порядка. Это делает `OrderedDict` идеальным для LRU-кэшей и логики, чувствительной к порядку.

**Q3. Как работает `default_factory` в `defaultdict` и какой главный подводный камень?**

A. `default_factory` — это вызываемый объект, который вызывается **без аргументов** при доступе к отсутствующему ключу через `__getitem__` (то есть `dd[key]`). Результат становится значением ключа, а ключ **создаётся**. Главный подводный камень: само чтение `dd[key]` для отсутствующего ключа создаёт запись. Поэтому для проверки наличия без побочных эффектов используют `key in dd` или `dd.get(key)`. Ещё: `defaultdict` срабатывает только на `__getitem__`, а не на `.get()`.

**Q4. Объясните арифметику `Counter`: чем `+`/`-` отличаются от `&`/`|` и от `update`/`subtract`?**

A. `+` складывает счётчики, `-` вычитает, но **оба отбрасывают неположительные** результаты. `&` берёт поэлементный минимум (пересечение мультимножеств), `|` — максимум (объединение). `update` прибавляет (аналог `+`, но **на месте** и **сохраняет** отрицательные/нулевые значения), `subtract` вычитает на месте и тоже сохраняет отрицательные. То есть для «честного» вычитания с возможным минусом — `subtract`; для «мультимножественного» — оператор `-`.

```python
a, b = Counter(x=1), Counter(x=3)
print(a - b)              # => Counter()        (1-3=-2 отброшено)
c = Counter(x=1); c.subtract(Counter(x=3))
print(c)                  # => Counter({'x': -2})  (сохранено)
```

**Q5. Зачем `namedtuple`, если есть обычный класс или `dict`?**

A. `namedtuple` даёт неизменяемость, хешируемость, компактность по памяти (нет `__dict__`, есть `__slots__`-подобное поведение), доступ по имени и по индексу, распаковку и совместимость со всем, что ждёт кортеж. По сравнению с `dict` — меньше памяти, защита от опечаток в ключах, читаемый `repr`. По сравнению с обычным классом — меньше шаблонного кода. Если нужны изменяемость, методы или дефолты с логикой — лучше `typing.NamedTuple` (классовый синтаксис) или `@dataclass`.

**Q6. В чём разница между `namedtuple` из `collections` и `typing.NamedTuple`?**

A. Функциональность хранения та же (оба — подклассы `tuple`). `typing.NamedTuple` даёт классовый синтаксис с аннотациями типов, поддерживает значения по умолчанию, методы, свойства и докстринги естественным образом. `collections.namedtuple` — фабричная функция, дефолты задаются через параметр `defaults`. Для нового кода с типизацией обычно предпочтительнее `typing.NamedTuple`.

**Q7. Что такое `ChainMap` и чем он отличается от `dict.update()` для слияния?**

A. `ChainMap` создаёт **представление** поверх нескольких словарей без копирования: поиск идёт по цепочке, побеждает первый найденный ключ. `{**a, **b}` или `a.update(b)` создают **новый объединённый** словарь (копию). Отличия: (1) `ChainMap` не дублирует данные — изменения в исходных словарях видны; (2) записи в `ChainMap` идут только в первый словарь; (3) дешевле по памяти при больших словарях; (4) можно динамически добавлять слои через `new_child`. Типичный кейс — приоритетные конфигурации.

**Q8. Зачем нужны `UserDict`/`UserList`/`UserString` вместо наследования от `dict`/`list`/`str`?**

A. При прямом наследовании от встроенных C-типов их собственные методы (`__init__`, `update`, операции конкатенации и т.п.) реализованы на C и **не вызывают** ваши переопределённые `__setitem__`/`__getitem__`. Поэтому переопределение поведения получается неполным и непредсказуемым. `UserDict`/`UserList`/`UserString` хранят данные в атрибуте `.data` и реализованы на чистом Python так, что все операции проходят через переопределяемые методы — наследование ведёт себя единообразно. Если же нужно только добавить методы (без переопределения базового поведения), часто хватает прямого наследования или композиции.

**Q9. Как реализовать простой LRU-кэш на `OrderedDict`?**

A. Используют `move_to_end` при доступе (помечает ключ как недавно использованный) и `popitem(last=False)` при переполнении (удаляет самый старый — с начала).

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)        # отметить как недавно использованный
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # удалить самый старый (с начала)

lru = LRUCache(2)
lru.put(1, "a"); lru.put(2, "b")
print(lru.get(1))     # => a   (1 стал свежим)
lru.put(3, "c")       # переполнение -> вытесняется 2 (самый старый)
print(lru.get(2))     # => -1  (вытеснен)
print(lru.get(3))     # => c
```

**Q10. Что вернёт `Counter` для отсутствующего ключа и почему это удобно?**

A. `0`, а не `KeyError`. Это позволяет писать `c[key] += 1` без предварительной проверки наличия ключа. Но важно понимать: чтение отсутствующего ключа **не создаёт** запись в `Counter` (в отличие от `defaultdict`), хотя присваивание (`c[key] += 1`) создаст.

```python
c = Counter()
print(c["x"])        # => 0
print(dict(c))       # => {}   (ключ 'x' не создан простым чтением)
c["x"] += 1
print(dict(c))       # => {'x': 1}
```

**Q11. Как `deque(maxlen=N)` помогает реализовать скользящее окно или буфер последних событий?**

A. При добавлении в полную `deque` элемент автоматически вытесняется с противоположного конца — получается кольцевой буфер фиксированного размера за O(1) на операцию. Это идеально для «последних N логов», скользящего среднего, ограниченной истории.

```python
log = deque(maxlen=3)
for event in ["a", "b", "c", "d"]:
    log.append(event)
print(list(log))     # => ['b', 'c', 'd']  (хранятся только последние 3)
```

**Q12. Почему `Counter.most_common()` удобнее ручной сортировки, и как он сортирует при равных частотах?**

A. `most_common(n)` возвращает n пар `(элемент, количество)` по убыванию частоты, используя `heapq.nlargest` (эффективнее полной сортировки при малом n). При **равных** частотах порядок соответствует порядку первого появления элементов (insertion order), что предсказуемо. Ручной аналог `sorted(c.items(), key=lambda kv: -kv[1])` менее читаем и не использует частичную сортировку.

## Подводные камни (gotchas)

```python
# 1. defaultdict: чтение отсутствующего ключа СОЗДАЁТ его
from collections import defaultdict
dd = defaultdict(list)
if dd["maybe"]:          # это обращение создало пустой список!
    pass
print(dict(dd))          # => {'maybe': []}
# Правильно: проверять через 'maybe' in dd
```

```python
# 2. Counter: вычитание оператором '-' молча теряет отрицательные значения
from collections import Counter
print(Counter(a=1) - Counter(a=5))     # => Counter()   (а не {'a': -4})
# Нужны отрицательные -> используйте subtract()
```

```python
# 3. deque.extendleft переворачивает порядок добавляемых элементов
from collections import deque
d = deque([0])
d.extendleft([1, 2, 3])
print(d)     # => deque([3, 2, 1, 0])   а не [1, 2, 3, 0]
```

```python
# 4. namedtuple неизменяем: _replace возвращает НОВЫЙ объект, не меняет старый
from collections import namedtuple
P = namedtuple("P", "x y")
p = P(1, 2)
p._replace(x=9)          # результат проигнорирован!
print(p)                 # => P(x=1, y=2)   (без изменений)
# Правильно: p = p._replace(x=9)
```

```python
# 5. ChainMap: del работает только по первому словарю
from collections import ChainMap
cm = ChainMap({}, {"a": 1})
try:
    del cm["a"]          # 'a' живёт во втором словаре
except KeyError as e:
    print("KeyError:", e)
```

```python
# 6. Прямое наследование от dict не вызывает переопределённый __setitem__
class D(dict):
    def __setitem__(self, k, v):
        super().__setitem__(k, v * 10)
d = D(a=1)               # __init__ НЕ идёт через __setitem__
print(d)                 # => {'a': 1}   (а не {'a': 10})
# Используйте UserDict для корректного поведения
```

```python
# 7. OrderedDict сравнивается с учётом порядка (легко словить неожиданное False)
from collections import OrderedDict
print(OrderedDict(a=1, b=2) == OrderedDict(b=2, a=1))   # => False
```

```python
# 8. defaultdict в repr/pickle и при передаче в функции остаётся defaultdict.
# Иногда нужно «заморозить» его в обычный dict перед сериализацией/сравнением:
dd = defaultdict(int); dd["x"] += 1
print(dict(dd) == {"x": 1})   # => True
```

## Лучшие практики

- Используйте `defaultdict(list)`/`defaultdict(int)`/`defaultdict(set)` для группировки и подсчёта вместо ручных проверок `if key not in d`.
- Для подсчёта частот и топ-N всегда берите `Counter` + `most_common`, а не самописную сортировку.
- Для очередей, стеков с двумя концами и скользящих окон используйте `deque`; задавайте `maxlen` для ограниченных буферов.
- Не используйте `deque` для частого индексного доступа в середину — там `list` эффективнее.
- Для лёгких неизменяемых записей с типами берите `typing.NamedTuple`; если нужна изменяемость или богатое поведение — `@dataclass`.
- Не полагайтесь на `OrderedDict` ради порядка (обычный `dict` его хранит) — берите его только ради `move_to_end`, `popitem(last=False)` или сравнения с учётом порядка.
- Для приоритетных конфигов предпочитайте `ChainMap` слиянию `{**a, **b}`, если важна экономия памяти или динамические слои.
- Для кастомных контейнеров с переопределением поведения наследуйтесь от `UserDict`/`UserList`/`UserString`, а не от `dict`/`list`/`str`.
- Проверяйте наличие ключа в `defaultdict` через `in` или `.get()`, чтобы не создавать «фантомные» записи.
- «Замораживайте» `defaultdict` в обычный `dict` перед сериализацией/возвратом из API, если не хотите неявного создания ключей у потребителя.

## Шпаргалка

```python
from collections import (namedtuple, deque, Counter,
                         OrderedDict, defaultdict, ChainMap,
                         UserDict, UserList, UserString)

# namedtuple ---------------------------------------------------------
P = namedtuple("P", "x y", defaults=[0])     # y по умолчанию 0
p = P(1); p.x; p[0]                          # доступ по имени/индексу
P._fields                                    # ('x', 'y')
p._replace(y=5)                              # НОВЫЙ объект
p._asdict()                                  # {'x': 1, 'y': 0}
P._make([3, 4])                              # из итерируемого
# Альтернатива: class P(typing.NamedTuple): x: int; y: int = 0

# deque --------------------------------------------------------------
d = deque([1, 2], maxlen=3)
d.append(3); d.appendleft(0)                 # O(1) с обоих концов
d.pop(); d.popleft()                         # O(1)
d.rotate(1); d.rotate(-1)                    # циклический сдвиг
# maxlen -> вытеснение с противоположного конца

# Counter ------------------------------------------------------------
c = Counter("aab")                           # {'a':2,'b':1}
c.most_common(1)                             # [('a', 2)]
c.elements()                                 # итератор a a b
c.update(["a"]); c.subtract({"b": 5})        # +/- на месте
a + b; a - b; a & b; a | b                   # сумма/разн(>0)/min/max
c.total()                                    # сумма значений (3.10+)

# OrderedDict --------------------------------------------------------
od = OrderedDict(a=1, b=2)
od.move_to_end("a")                          # в конец
od.move_to_end("a", last=False)              # в начало
od.popitem(last=False)                       # удалить с начала (FIFO)
# сравнение OrderedDict == OrderedDict учитывает порядок

# defaultdict --------------------------------------------------------
dd = defaultdict(list); dd["k"].append(1)    # автосоздание значения
defaultdict(int)                             # счётчик -> 0
defaultdict(set)                             # граф/множества
# dd[key] для отсутствующего -> СОЗДАЁТ ключ

# ChainMap -----------------------------------------------------------
cm = ChainMap(overrides, defaults)           # поиск слева направо
cm.maps                                      # список словарей
cm.new_child({"x": 1})                       # добавить слой в начало
cm.parents                                   # без первого слоя
# запись/удаление -> только первый словарь

# UserDict / UserList / UserString -----------------------------------
class MyDict(UserDict):                       # данные в self.data
    def __setitem__(self, k, v): super().__setitem__(k.upper(), v)
# наследуйтесь от User* при переопределении поведения контейнера
```
