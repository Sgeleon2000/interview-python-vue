# enum — подготовка к собеседованию

## Что это и зачем

`enum` — модуль стандартной библиотеки (с Python 3.4), который позволяет создавать
**перечисления** (enumerations): набор именованных констант, связанных в единый тип.

Зачем это нужно:

- **Читаемость.** `status = OrderStatus.PAID` понятнее, чем `status = 2`.
- **Безопасность.** Нельзя случайно присвоить «магическое» число, которого нет в наборе.
- **Самодокументируемость.** Все допустимые значения собраны в одном месте.
- **Группировка констант.** Вместо разрозненных `RED = 1; GREEN = 2; BLUE = 3` —
  единый тип `Color`, по которому можно итерироваться, проверять принадлежность и т.д.
- **Синглтоны.** Каждый член перечисления существует в единственном экземпляре,
  что позволяет сравнивать через `is`.

```python
from enum import Enum

# Без enum: магические числа, легко ошибиться
def pay(status):
    if status == 2:        # что значит "2"? непонятно
        ...

# С enum: явно и безопасно
class OrderStatus(Enum):
    NEW = 1
    PAID = 2
    SHIPPED = 3

def pay(status: OrderStatus):
    if status is OrderStatus.PAID:   # читаемо и однозначно
        ...
```

Главная идея: **член enum — это не просто значение, а объект-синглтон** со свойствами
`.name` и `.value`. Это отличает enum от обычной группы констант.

## Ключевые концепции

- **Член (member)** — именованная константа перечисления. У него есть `.name` (строка-имя)
  и `.value` (произвольное значение).
- **Синглтоны и identity.** Каждый член создаётся ровно один раз. Правильное сравнение —
  через `is`, а не `==` (хотя `==` для членов того же enum тоже работает по identity).
- **Доступ по имени и по значению.** `Color['RED']` — по имени (как из словаря),
  `Color(1)` — по значению (вызов класса как «фабрики»).
- **Итерация** идёт по членам в порядке объявления; алиасы пропускаются.
- **Алиасы (aliases).** Два члена с одинаковым `value` — второй становится алиасом первого.
- **`auto()`** автоматически генерирует значения.
- **`@unique`** запрещает алиасы (валится с ошибкой при дубликате значения).
- **Иерархия типов:** `Enum` (база), `IntEnum` (члены — это `int`), `StrEnum` (3.11+, члены —
  это `str`), `Flag` (битовые комбинации), `IntFlag` (битовые + это `int`).
- **`EnumMeta` / `EnumType`** — метакласс, который и реализует всю магию: запрет
  переопределения, итерацию, поиск по значению, кэш `_value2member_map_`.
- **`_missing_`** — хук для обработки неизвестных значений при `EnumClass(x)`.

## Основные функции/классы/методы

### Базовый `Enum`

```python
from enum import Enum

class Color(Enum):
    RED = 1
    GREEN = 2
    BLUE = 3

c = Color.RED

# name и value
print(c.name)            # => RED
print(c.value)           # => 1
print(c)                 # => Color.RED
print(repr(c))           # => <Color.RED: 1>

# Доступ по имени (как из словаря) — через []
print(Color['GREEN'])    # => Color.GREEN

# Доступ по значению (вызов класса) — через ()
print(Color(3))          # => Color.BLUE

# Итерация — в порядке объявления
print(list(Color))       # => [<Color.RED: 1>, <Color.GREEN: 2>, <Color.BLUE: 3>]
for color in Color:
    print(color.name, color.value)
# => RED 1
# => GREEN 2
# => BLUE 3

# Сравнение по identity (члены — синглтоны)
print(Color.RED is Color.RED)        # => True
print(Color.RED is Color(1))         # => True
print(Color.RED == Color.RED)        # => True
print(Color.RED == 1)                # => False  (Enum НЕ равен своему value)

# Проверка принадлежности
print(Color.RED in Color)            # => True

# __members__ — упорядоченный маппинг ИМЯ -> член (включая алиасы)
print(list(Color.__members__))       # => ['RED', 'GREEN', 'BLUE']
```

Важно: обычный `Enum` **не** равен своему значению (`Color.RED == 1` → `False`).
Это сознательное ограничение для безопасности типов.

### `IntEnum`

Члены `IntEnum` — это полноценные `int`, поэтому они **сравниваются и взаимодействуют с int**.

```python
from enum import IntEnum

class Priority(IntEnum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3

# Член — это int, поэтому сравнения с числами работают
print(Priority.HIGH == 3)            # => True
print(Priority.HIGH > Priority.LOW)  # => True
print(Priority.MEDIUM + 10)          # => 12   (ведёт себя как int)
print(isinstance(Priority.LOW, int)) # => True

# Можно использовать там, где ожидается int
print(sorted([Priority.HIGH, Priority.LOW, Priority.MEDIUM]))
# => [<Priority.LOW: 1>, <Priority.MEDIUM: 2>, <Priority.HIGH: 3>]

# Полезно для совместимости (например, HTTP-статусы)
import http
print(http.HTTPStatus.OK == 200)     # => True (HTTPStatus — это IntEnum)
```

Плюс: удобная совместимость со старым кодом, который оперирует числами.
Минус: «протекание» — член нечаянно сравнивается с любым int (см. gotchas).

### `StrEnum` (Python 3.11+)

Члены `StrEnum` — это полноценные `str`. До 3.11 этого добивались миксином `class X(str, Enum)`.

```python
from enum import StrEnum   # Python 3.11+

class Env(StrEnum):
    DEV = "dev"
    PROD = "prod"

print(Env.PROD == "prod")            # => True
print(isinstance(Env.PROD, str))     # => True
print(Env.PROD.upper())              # => PROD  (строковые методы работают)
print(f"Running in {Env.PROD}")      # => Running in prod

# До 3.11 делали так (str + Enum миксин):
from enum import Enum
class EnvOld(str, Enum):
    DEV = "dev"
    PROD = "prod"

print(EnvOld.PROD == "prod")         # => True
# Но есть нюанс форматирования:
print(f"{EnvOld.PROD}")              # => EnvOld.PROD  (в 3.11 поведение __str__ изменили)
print(f"{Env.PROD}")                 # => prod  (StrEnum.__str__ == str.__str__)
```

Зачем `StrEnum`: значения, которые должны напрямую сериализоваться в строку (JSON, БД,
заголовки, имена полей), и при этом оставаться перечислением.

Почему раньше делали `str + Enum`: чтобы член можно было использовать как строку
напрямую (в шаблонах, ключах словарей, при сравнении со строками из БД/API).
`StrEnum` стандартизировал этот паттерн и поправил `__str__`.

### `Flag` и `IntFlag` (битовые комбинации)

`Flag` поддерживает побитовые операции `|`, `&`, `^`, `~`. Значения членов должны быть
степенями двойки (с `auto()` это автоматически).

```python
from enum import Flag, auto

class Permission(Flag):
    READ = auto()     # 1
    WRITE = auto()    # 2
    EXECUTE = auto()  # 4

print(Permission.READ.value)     # => 1
print(Permission.WRITE.value)    # => 2
print(Permission.EXECUTE.value)  # => 4

# Комбинация флагов через |
rw = Permission.READ | Permission.WRITE
print(rw)                        # => Permission.READ|WRITE
print(rw.value)                  # => 3

# Проверка вхождения флага через &
print(Permission.READ in rw)     # => True
print(Permission.EXECUTE in rw)  # => False
print(bool(rw & Permission.WRITE))   # => True

# Снятие флага: XOR или AND с инверсией
print(rw & ~Permission.WRITE)    # => Permission.READ
print(rw ^ Permission.READ)      # => Permission.WRITE

# Итерация по составному флагу (Python 3.11+) выдаёт отдельные взведённые биты
for p in rw:
    print(p)
# => Permission.READ
# => Permission.WRITE

# Пустой флаг (ничего не взведено)
none = Permission(0)
print(none)                      # => Permission(0)  (в старых версиях Permission.0)
print(bool(none))                # => False
```

`IntFlag` — то же самое, но члены ещё и являются `int`:

```python
from enum import IntFlag, auto

class Access(IntFlag):
    READ = auto()    # 1
    WRITE = auto()   # 2
    EXEC = auto()    # 4

combo = Access.READ | Access.EXEC
print(combo.value)               # => 5
print(combo == 5)                # => True  (это int!)
print(int(combo))                # => 5
print(Access(5))                 # => Access.READ|EXEC  (восстанавливаем из числа)

# Удобно для C-style битовых масок (os.open flags, mmap и т.п.)
```

Разница `Flag` vs `IntFlag`: `IntFlag` свободно смешивается с целыми числами
(полезно для системных вызовов и битовых масок), `Flag` — строже и безопаснее.

### `auto()` и `_generate_next_value_`

`auto()` присваивает следующее значение автоматически. Логику генерации задаёт
`_generate_next_value_(name, start, count, last_values)`.

```python
from enum import Enum, auto

class Color(Enum):
    RED = auto()     # 1
    GREEN = auto()   # 2
    BLUE = auto()    # 3

print([c.value for c in Color])  # => [1, 2, 3]

# Кастомная генерация: значение = имя в нижнем регистре
class Color2(Enum):
    # name      - имя члена ("RED")
    # start     - 1 по умолчанию
    # count     - сколько членов уже определено
    # last_values - список ранее присвоенных значений
    def _generate_next_value_(name, start, count, last_values):
        return name.lower()

    RED = auto()
    GREEN = auto()

print(Color2.RED.value)          # => red
print(Color2.GREEN.value)        # => green

# Важно: _generate_next_value_ должен быть объявлен ДО членов
```

В `Flag`/`IntFlag` `auto()` выдаёт степени двойки (1, 2, 4, 8, ...), а не 1, 2, 3.

### Функциональный API

Enum можно создать «на лету», без `class`-синтаксиса.

```python
from enum import Enum

# Имена через список — значения авто-нумеруются с 1
Animal = Enum('Animal', ['CAT', 'DOG', 'COW'])
print(list(Animal))              # => [<Animal.CAT: 1>, <Animal.DOG: 2>, <Animal.COW: 3>]
print(Animal.DOG.value)          # => 2

# Имена через строку с разделителями (пробел/запятая)
Animal2 = Enum('Animal2', 'CAT DOG COW')
print(Animal2.COW.value)         # => 3

# Явные значения через словарь или список кортежей
Planet = Enum('Planet', {'MERCURY': 1, 'VENUS': 2})
Planet2 = Enum('Planet2', [('A', 10), ('B', 20)])
print(Planet.VENUS.value)        # => 2
print(Planet2.B.value)           # => 20

# Полезно для динамического создания и в metaclass-сценариях
```

### Добавление методов, свойств и `_missing_`

Enum — это обычный класс, поэтому в нём можно объявлять методы, `@property`,
классовые методы, а также переопределять `_missing_`.

```python
from enum import Enum

class Planet(Enum):
    # value = (масса, радиус)
    MERCURY = (3.303e+23, 2.4397e6)
    EARTH   = (5.976e+24, 6.37814e6)

    def __init__(self, mass, radius):
        # Распаковываем кортеж-значение в атрибуты
        self.mass = mass
        self.radius = radius

    @property
    def surface_gravity(self):
        # G * mass / radius**2
        G = 6.67300e-11
        return G * self.mass / (self.radius ** 2)

    def describe(self):
        return f"{self.name}: g = {self.surface_gravity:.2f}"

print(Planet.EARTH.surface_gravity)   # => 9.802652743337129
print(Planet.EARTH.describe())        # => EARTH: g = 9.80
print(Planet.EARTH.value)             # => (5.976e+24, 6378140.0)


# _missing_ — обработка неизвестных значений при ВЫЗОВЕ класса
class Color(Enum):
    RED = 1
    GREEN = 2
    BLUE = 3

    @classmethod
    def _missing_(cls, value):
        # Вызывается, когда Color(value) не нашёл точного совпадения.
        # Например, разрешим поиск по имени без учёта регистра.
        if isinstance(value, str):
            for member in cls:
                if member.name.lower() == value.lower():
                    return member
        # Вернуть None => поднимется ValueError
        return None

print(Color('red'))              # => Color.RED   (нашли через _missing_)
print(Color(2))                  # => Color.GREEN (обычный путь)
try:
    Color('purple')
except ValueError as e:
    print('ValueError:', e)      # => ValueError: 'purple' is not a valid Color
```

## Частые вопросы на собеседовании

**Q1. Чем отличается доступ `Color['RED']` от `Color(1)`?**

`Color['RED']` — доступ **по имени** члена (`__getitem__` метакласса, работает как
словарь по `__members__`). `Color(1)` — доступ **по значению** (вызов класса как
фабрики, ищет в `_value2member_map_`). Несуществующее имя даёт `KeyError`,
несуществующее значение — `ValueError`.

```python
class Color(Enum):
    RED = 1
print(Color['RED'])    # => Color.RED   (по имени)
print(Color(1))        # => Color.RED   (по значению)
# Color['X']  -> KeyError
# Color(99)   -> ValueError
```

**Q2. Что такое алиасы и как они себя ведут при итерации?**

Если двум именам присвоено **одинаковое значение**, второе становится **алиасом**
первого (а не отдельным членом). Алиасы:
- НЕ появляются при итерации `for x in Enum` и в `list(Enum)`;
- ПОЯВЛЯЮТСЯ в `__members__`;
- указывают на тот же объект-синглтон.

```python
class Color(Enum):
    RED = 1
    CRIMSON = 1   # алиас для RED (то же значение)
    GREEN = 2

print(list(Color))                   # => [<Color.RED: 1>, <Color.GREEN: 2>]  (CRIMSON нет)
print(Color.CRIMSON is Color.RED)    # => True
print(list(Color.__members__))       # => ['RED', 'CRIMSON', 'GREEN']  (CRIMSON есть)
print(Color['CRIMSON'])              # => Color.RED
```

**Q3. Как запретить алиасы?**

Декоратором `@unique` из модуля `enum`. Он проверяет уникальность значений на этапе
определения класса и поднимает `ValueError` при дубликате.

```python
from enum import Enum, unique

@unique
class Color(Enum):
    RED = 1
    GREEN = 2
    # CRIMSON = 1   # => ValueError: duplicate values found: CRIMSON -> RED
```

**Q4. В чём опасность `IntEnum` по сравнению с `Enum`?**

Члены `IntEnum` — это `int`, поэтому они неявно сравниваются с любыми числами и
с членами других `IntEnum`. Это может «протечь» и спрятать ошибку: член одного
перечисления случайно окажется равным числу или члену совсем другого перечисления.

```python
from enum import Enum, IntEnum

class A(IntEnum):
    X = 1
class B(IntEnum):
    Y = 1

print(A.X == B.Y)     # => True   (оба == 1, опасное «случайное» равенство!)
print(A.X == 1)       # => True

# С обычным Enum таких сюрпризов нет:
class C(Enum):
    X = 1
class D(Enum):
    Y = 1
print(C.X == D.Y)     # => False
print(C.X == 1)       # => False
```

Вывод: используйте `IntEnum` только когда совместимость с int действительно нужна.
По умолчанию предпочитайте обычный `Enum`.

**Q5. Почему до 3.11 писали `class X(str, Enum)` и что изменил `StrEnum`?**

Чтобы член enum можно было использовать как настоящую строку (в сравнениях,
ключах словарей, сериализации). `str + Enum` давал это через миксин. `StrEnum`
(3.11+) стандартизировал паттерн и, главное, сделал `__str__` равным строковому
представлению значения. Аналогично появилась рекомендация использовать `StrEnum`
вместо ручного миксина.

```python
from enum import Enum, StrEnum
class Old(str, Enum):
    A = "a"
class New(StrEnum):
    A = "a"
print(f"{Old.A}")   # => Old.A   (в 3.11+ __str__ от Enum)
print(f"{New.A}")   # => a       (StrEnum.__str__ == str.__str__)
print(Old.A == "a", New.A == "a")  # => True True
```

**Q6. Почему `auto()` в `Flag` даёт степени двойки, а в `Enum` — 1,2,3?**

Потому что `Flag._generate_next_value_` переопределён: он выдаёт следующую свободную
степень двойки, чтобы каждый флаг занимал отдельный бит. У `Enum` дефолтный
`_generate_next_value_` инкрементирует целое.

```python
from enum import Enum, Flag, auto
class E(Enum):
    A = auto(); B = auto(); C = auto()
class F(Flag):
    A = auto(); B = auto(); C = auto()
print([m.value for m in E])   # => [1, 2, 3]
print([m.value for m in F])   # => [1, 2, 4]
```

**Q7. Как обработать неизвестное/невалидное значение при создании члена?**

Через хук-classmethod `_missing_(cls, value)`. Он вызывается, когда `EnumClass(value)`
не нашёл совпадения. Вернёте член — он станет результатом; вернёте `None` (или ничего) —
поднимется `ValueError`. Удобно для нормализации входных данных (регистр, синонимы,
fallback на UNKNOWN).

```python
class Status(Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"
    UNKNOWN = "unknown"

    @classmethod
    def _missing_(cls, value):
        return cls.UNKNOWN     # любое неизвестное значение -> UNKNOWN

print(Status("active"))   # => Status.ACTIVE
print(Status("banana"))   # => Status.UNKNOWN
```

**Q8. Можно ли наследоваться от enum, у которого уже есть члены?**

Нет. Если в базовом enum **есть хотя бы один член**, наследование запрещено
(`TypeError`). Это защищает инвариант «фиксированный набор констант».
Наследовать можно только enum **без членов** (так подмешивают методы/миксины) —
паттерн «базовый enum с поведением, наследники с данными».

```python
from enum import Enum
class Color(Enum):
    RED = 1
# class More(Color):   # => TypeError: Color: cannot extend <enum 'Color'>
#     GREEN = 2

# А так можно — база без членов, только поведение:
class Described(Enum):
    def describe(self):
        return f"{self.name}={self.value}"
class Fruit(Described):
    APPLE = 1
    PEAR = 2
print(Fruit.APPLE.describe())   # => APPLE=1
```

**Q9. Как правильно сравнивать члены enum: `is` или `==`?**

Для обычного `Enum` корректны оба (члены — синглтоны, `==` сводится к identity), но
**предпочтителен `is`** — он явно выражает намерение и не сравнивает значения.
Сравнивать член с «сырым» значением (`Color.RED == 1`) для обычного Enum нельзя —
вернёт `False`. Для `IntEnum`/`StrEnum` сравнение со значением работает, но именно
поэтому появляются ловушки (см. Q4).

```python
class Color(Enum):
    RED = 1
print(Color.RED is Color.RED)   # => True  (рекомендуется)
print(Color.RED == Color.RED)   # => True
print(Color.RED is Color(1))    # => True  (синглтон)
```

**Q10. Что такое `EnumMeta` / `EnumType` и зачем он?**

Это метакласс всех перечислений. Именно он реализует «магию»: собирает члены,
строит `_value2member_map_` и `__members__`, делает итерацию, обеспечивает
синглтоны, запрещает переопределение членов и наследование от enum с членами,
поддерживает `EnumClass(value)` и `EnumClass[name]`. В 3.11 `EnumMeta`
переименован в `EnumType` (старое имя оставлено как алиас).

```python
from enum import Enum, EnumType   # EnumType == EnumMeta (3.11+)
class Color(Enum):
    RED = 1
print(type(Color))                # => <class 'enum.EnumType'>
print(type(Color) is EnumType)    # => True
```

**Q11. Как добавить вычисляемое свойство к члену enum?**

Через обычный `@property`. Часто значением делают кортеж и распаковывают его в
`__init__`, а свойство считает производные данные.

```python
class Shirt(Enum):
    SMALL = ("S", 36)
    LARGE = ("L", 44)
    def __init__(self, code, chest):
        self.code = code
        self.chest = chest
    @property
    def label(self):
        return f"{self.code} ({self.chest}cm)"
print(Shirt.LARGE.label)   # => L (44cm)
print(Shirt.LARGE.code)    # => L
```

**Q12. Как проверить, что значение/флаг входит в составной `Flag`?**

Через оператор `in` (с 3.12 он работает и для проверки одиночного флага в составном),
либо через побитовое `&`. Итерация по составному флагу (3.11+) перечисляет
взведённые биты.

```python
from enum import Flag, auto
class P(Flag):
    R = auto(); W = auto(); X = auto()
rw = P.R | P.W
print(P.R in rw)            # => True
print(P.X in rw)            # => False
print(bool(rw & P.W))      # => True
print(list(rw))            # => [<P.R: 1>, <P.W: 2>]
```

## Подводные камни (gotchas)

- **`Enum.RED == 1` → `False`.** Обычный Enum не равен своему значению. Сравнивайте
  члены с членами, либо берите `.value` явно. Только `IntEnum`/`StrEnum` равны
  своему значению.

- **`IntEnum` «протекает».** `A.X == B.Y` может оказаться `True`, если значения
  совпадают. И `A.X == 1` тоже `True`. Это удобно, но скрывает баги — выбирайте
  обычный `Enum`, если совместимость с int не нужна.

- **Алиасы тихо «съедаются».** Повтор значения не создаёт новый член, а делает алиас.
  Если вы этого не ждали — член «пропадает» из итерации. Ставьте `@unique`, если
  алиасы не нужны.

- **`auto()` в `Flag` ≠ `auto()` в `Enum`.** В Flag это степени двойки. Если вручную
  присвоить флагу не-степень двойки (например, `3`), он станет составным/алиасом, а
  не самостоятельным битом.

- **Порядок `_generate_next_value_`.** Метод должен быть объявлен **до** первого члена,
  иначе на ранние члены он не повлияет.

- **Нельзя переопределять члены.** Попытка присвоить `Color.RED = ...` или объявить
  два метода с именем существующего члена даёт `TypeError`/`AttributeError`.

- **Нельзя наследовать enum с членами.** `TypeError`. Подмешивайте поведение через
  enum-базу без членов.

- **`f"{member}"` менялся между версиями.** Для `IntEnum`/`IntFlag` в 3.11 `__str__`/
  `__format__` поправили: формат теперь от `int`. Для `str+Enum` миксина `__str__`
  стал `Enum.__str__` (а у `StrEnum` — строковый). Не полагайтесь на конкретный вид
  строки между версиями — используйте `.value` явно.

- **`_missing_`, вернувший не-член.** Если вернуть что-то, что не является членом
  данного enum, поднимется ошибка. Возвращайте член класса или `None`.

- **Изменяемые значения.** `value` может быть кортежем/объектом, но члены — синглтоны;
  не храните в них изменяемое разделяемое состояние, это запутает логику.

- **`pickle` и алиасы/функциональный API.** Для пиклинга enum, созданного
  функциональным API, иногда нужно явно указать `module=...`/`qualname=...`, иначе
  pickle не найдёт класс.

## Лучшие практики

- **По умолчанию используйте обычный `Enum`.** Берите `IntEnum`/`StrEnum` только когда
  действительно нужна прозрачная совместимость с `int`/`str` (внешние API, БД, протоколы).

- **Сравнивайте через `is`** для обычных enum — это явно и безопасно.

- **Ставьте `@unique`,** если алиасы не предусмотрены намеренно. Это ловит опечатки в
  значениях на этапе импорта.

- **Используйте `auto()`,** когда конкретные числа значений не важны — меньше ручных
  ошибок и дубликатов.

- **Кладите поведение в enum.** Методы, `@property`, `__init__` с распаковкой кортежа —
  это делает enum «умным» и убирает разрозненные хелпер-функции.

- **`_missing_` для нормализации входа** (регистр, синонимы) и осознанного fallback
  (например, `UNKNOWN`), вместо россыпи `try/except` по коду.

- **Аннотируйте типы** членами enum (`def f(s: OrderStatus): ...`) — статические
  анализаторы поймают передачу неверных значений.

- **Для битовых масок — `Flag`/`IntFlag`** вместо ручных `1 << n` констант: получаете
  безопасные операции и читаемый `repr`.

- **Не привязывайтесь к строковому виду** (`str(member)`) в логике — он менялся между
  версиями Python; используйте `.name`/`.value`.

## Шпаргалка

```python
from enum import (
    Enum, IntEnum, StrEnum, Flag, IntFlag, auto, unique, EnumType,
)

# --- Объявление ---
class Color(Enum):
    RED = 1
    GREEN = 2
    BLUE = auto()           # 3

# --- Доступ ---
Color.RED                   # член по атрибуту
Color['RED']                # по ИМЕНИ  -> KeyError если нет
Color(1)                    # по ЗНАЧЕНИЮ -> ValueError если нет
Color.RED.name              # 'RED'
Color.RED.value             # 1

# --- Итерация / члены ---
list(Color)                 # члены без алиасов, в порядке объявления
list(Color.__members__)     # имена ВКЛЮЧАЯ алиасы
Color.RED in Color          # проверка принадлежности

# --- Сравнение ---
Color.RED is Color.RED      # True (предпочтительно)
Color.RED == 1              # False (обычный Enum)

# --- Алиасы и уникальность ---
@unique                     # запрет дубликатов значений (ValueError при дубле)
class C(Enum):
    A = 1
    B = 2

# --- IntEnum: член это int ---
class P(IntEnum):
    LOW = 1; HIGH = 3
P.HIGH == 3                 # True;  P.HIGH > P.LOW -> True

# --- StrEnum (3.11+): член это str ---
class Env(StrEnum):
    DEV = "dev"; PROD = "prod"
Env.PROD == "prod"          # True;  f"{Env.PROD}" -> "prod"

# --- Flag / IntFlag: битовые операции ---
class Perm(Flag):
    R = auto(); W = auto(); X = auto()    # 1, 2, 4
rw = Perm.R | Perm.W        # комбинация
Perm.R in rw                # True
rw & ~Perm.W                # снять флаг -> Perm.R
list(rw)                    # [Perm.R, Perm.W]  (3.11+)

# --- Функциональный API ---
Animal = Enum('Animal', ['CAT', 'DOG'])           # авто-значения 1,2
Animal = Enum('Animal', {'CAT': 1, 'DOG': 2})     # явные значения

# --- Методы / свойства / _missing_ ---
class Status(Enum):
    OK = "ok"; UNKNOWN = "unknown"
    @classmethod
    def _missing_(cls, value):     # неизвестное значение -> fallback
        return cls.UNKNOWN
    @property
    def is_ok(self):
        return self is Status.OK

# --- auto() кастомизация ---
class Lower(Enum):
    def _generate_next_value_(name, start, count, last_values):
        return name.lower()        # объявить ДО членов
    A = auto()                     # 'a'

# --- Метакласс ---
type(Color) is EnumType            # True (EnumType == EnumMeta, 3.11+)
```

Ключевые правила за 10 секунд:
1. По умолчанию — `Enum`; `IntEnum`/`StrEnum` только ради совместимости.
2. `[]` — по имени, `()` — по значению.
3. Алиас = повтор значения; `@unique` запрещает.
4. `auto()`: Enum → 1,2,3; Flag → 1,2,4.
5. Сравнивай через `is`; не полагайся на `str(member)`.
6. `_missing_` — для неизвестных значений; нельзя наследовать enum с членами.
