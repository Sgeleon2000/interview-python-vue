# Typing (аннотации типов) — подготовка к собеседованию

## Что это и зачем

Модуль `typing` и сам синтаксис аннотаций типов (PEP 484, 2015 год) позволяют описывать **ожидаемые типы** переменных, аргументов функций, возвращаемых значений, атрибутов классов и т.д. Python остаётся **динамически типизированным** языком, но аннотации дают «слой» статической типизации поверх него.

Главная мысль, которую почти всегда хотят услышать на собеседовании:

> **Аннотации типов в Python НЕ влияют на выполнение программы в рантайме.** Интерпретатор их не проверяет и не приводит типы. Это «подсказки» (type hints) для людей, IDE и **внешних статических анализаторов** (mypy, pyright, и т.п.).

Пример, который это доказывает:

```python
def add(a: int, b: int) -> int:
    return a + b

# Никакой ошибки в рантайме — Python проигнорирует аннотации:
print(add("hello", "world"))  # 'helloworld' — работает!
print(add([1], [2]))          # [1, 2] — тоже работает!
```

Интерпретатор просто складывает то, что пришло. Аннотации `int` — лишь декларация намерения. Ошибку «несоответствие типов» поймает только статический анализатор (`mypy script.py`), но не сам запуск.

### Зачем тогда они нужны

1. **Раннее обнаружение ошибок** — статический анализатор находит несовпадения типов до запуска кода (особенно ценно в больших проектах).
2. **Документация** — сигнатура `def f(x: list[int]) -> dict[str, int]` понятнее любого комментария и не «протухает».
3. **IDE/автодополнение** — PyCharm, VS Code дают точные подсказки, рефакторинг, переходы по коду.
4. **Самодокументируемые контракты** между модулями и командами.
5. **Рантайм-инструменты** (опционально): `pydantic`, `FastAPI`, `dataclasses`, `attrs` **читают** аннотации через интроспекцию и используют их для валидации/сериализации. Но это уже сами библиотеки, а не интерпретатор Python.

### Что аннотации НЕ делают

- НЕ проверяют типы в рантайме.
- НЕ приводят (не кастуют) типы: `x: int = "5"` оставит строку.
- НЕ влияют на производительность исполнения (кроме момента вычисления самих выражений-аннотаций при импорте, если не используется отложенное вычисление).
- НЕ являются обязательными — можно типизировать частично.

---

## Ключевые концепции

### 1. Статическая проверка vs рантайм

| | Статическая проверка | Рантайм |
|---|---|---|
| Кто выполняет | mypy, pyright, IDE | интерпретатор CPython |
| Когда | до запуска (анализ кода) | во время выполнения |
| Что делает с аннотациями | проверяет согласованность | игнорирует (хранит в `__annotations__`) |
| Реакция на ошибку типа | сообщение анализатора | ничего (если код корректен по значениям) |

Аннотации хранятся в специальном атрибуте `__annotations__`:

```python
def f(a: int, b: str = "x") -> bool:
    return True

print(f.__annotations__)
# {'a': <class 'int'>, 'b': <class 'str'>, 'return': <class 'bool'>}

class C:
    x: int
    y: str = "default"

print(C.__annotations__)  # {'x': <class 'int'>, 'y': <class 'str'>}
```

### 2. Эволюция синтаксиса (важно знать версии!)

- **PEP 484** (Python 3.5): сам модуль `typing`, `List`, `Dict`, `Optional`, `Union`...
- **PEP 526** (3.6): аннотации переменных `x: int = 5`.
- **PEP 585** (3.9): дженерики на встроенных типах — `list[int]` вместо `typing.List[int]`.
- **PEP 604** (3.10): оператор `|` для Union — `int | str`, `int | None`.
- **PEP 612** (3.10): `ParamSpec`, `Concatenate`.
- **PEP 613 / 647 / 655** (3.10): `TypeAlias`, `TypeGuard`, `Required/NotRequired`.
- **PEP 673** (3.11): `Self`.
- **PEP 646 / 655** (3.11): variadic generics, `Required/NotRequired` в `typing`.
- **PEP 695** (3.12): новый синтаксис дженериков и `type X = ...`.

### 3. Структурная vs номинальная типизация

- **Номинальная**: совместимость по имени класса/наследованию (`isinstance`, наследование).
- **Структурная** («утиная» типизация на уровне типов): совместимость по форме/набору методов — реализуется через `Protocol`.

---

## Основные функции/классы/методы

### Базовые коллекции: старый и новый синтаксис (PEP 585)

```python
from typing import List, Dict, Tuple, Set  # старый способ (до 3.9)

# Старый стиль (всё ещё работает, но deprecated с 3.9):
def old(a: List[int], b: Dict[str, int], c: Tuple[int, str], d: Set[int]) -> None:
    ...

# Новый стиль (PEP 585, Python 3.9+) — предпочтительный:
def new(a: list[int], b: dict[str, int], c: tuple[int, str], d: set[int]) -> None:
    ...
```

**Tuple — два важных случая:**

```python
# Кортеж фиксированной длины и разных типов:
point: tuple[int, int] = (10, 20)
record: tuple[str, int, bool] = ("Alice", 30, True)

# Кортеж произвольной длины из однотипных элементов (Ellipsis ...):
numbers: tuple[int, ...] = (1, 2, 3, 4, 5)  # ноль или более int
empty: tuple[int, ...] = ()
```

### Optional и Union

`Optional[X]` — это ровно `Union[X, None]`, то есть «X или None». Это НЕ означает «необязательный аргумент»!

```python
from typing import Optional, Union

# Старый стиль:
def find(name: str) -> Optional[str]:
    ...

def parse(x: Union[int, str]) -> int:
    if isinstance(x, str):
        return int(x)
    return x

# Новый стиль (PEP 604, Python 3.10+):
def find2(name: str) -> str | None:        # == Optional[str]
    ...

def parse2(x: int | str) -> int:           # == Union[int, str]
    ...
```

**Важно:** `Optional[int]` НЕ значит «параметр со значением по умолчанию». Это про значение, которое может быть `None`. Параметр становится необязательным из-за `= значение`, а не из-за `Optional`:

```python
# x обязателен, но может быть None:
def f(x: Optional[int]) -> None: ...
f()         # ОШИБКА: пропущен аргумент
f(None)     # OK

# x необязателен И может быть None:
def g(x: Optional[int] = None) -> None: ...
g()         # OK
```

### Callable

Описывает вызываемые объекты (функции, лямбды, методы):

```python
from typing import Callable

# Callable[[типы_аргументов], тип_возврата]
def apply(func: Callable[[int, int], int], a: int, b: int) -> int:
    return func(a, b)

apply(lambda x, y: x + y, 3, 4)  # 7

# Любые аргументы (сигнатура не важна) — многоточие:
handler: Callable[..., None]

# Функция без аргументов, возвращающая str:
factory: Callable[[], str] = lambda: "hi"
```

### TypeVar — параметры типа (дженерики)

`TypeVar` создаёт переменную типа — «заглушку», которая связывает типы между собой.

```python
from typing import TypeVar

T = TypeVar("T")  # любой тип

# T в аргументе и в возврате — один и тот же тип:
def first(items: list[T]) -> T:
    return items[0]

x = first([1, 2, 3])      # mypy выведет: x — int
y = first(["a", "b"])     # mypy выведет: y — str
```

**`bound` — верхняя граница типа** (тип должен быть подтипом):

```python
from typing import TypeVar

# Tnum — любой тип, являющийся подклассом (int | float):
Tnum = TypeVar("Tnum", bound=float)  # int — подтип float по PEP 484

def double(x: Tnum) -> Tnum:
    return x * 2  # int -> int, float -> float

# Практичный пример с bound по своему классу:
class Animal:
    def name(self) -> str: ...

A = TypeVar("A", bound=Animal)

def label(a: A) -> A:
    print(a.name())  # OK: гарантировано, что метод name есть
    return a
```

**`constraints` — фиксированный набор типов** (только один из перечисленных, без подтипов в виде смеси):

```python
# AnyStr — встроенный TypeVar с constraints:
from typing import AnyStr  # = TypeVar("AnyStr", str, bytes)

def concat(a: AnyStr, b: AnyStr) -> AnyStr:
    return a + b

concat("a", "b")        # OK: оба str -> str
concat(b"a", b"b")      # OK: оба bytes -> bytes
# concat("a", b"b")     # ОШИБКА: нельзя смешивать str и bytes
```

Разница `bound` vs `constraints`:
- `bound=X`: любой подтип X, и результат сохраняет конкретный подтип.
- `constraints=(A, B)`: ровно один из A или B (без объединения подклассов между собой).

**Ковариантность/контравариантность** (для дженерик-классов и протоколов):

```python
from typing import TypeVar

T_co = TypeVar("T_co", covariant=True)      # ковариантный
T_contra = TypeVar("T_contra", contravariant=True)  # контравариантный
T_inv = TypeVar("T_inv")                     # инвариантный (по умолчанию)
```

- **Ковариантность** (`covariant=True`): если `Cat` — подтип `Animal`, то `Producer[Cat]` — подтип `Producer[Animal]`. Подходит для типов «только на выход» (источники). Пример из stdlib — кортежи, неизменяемые контейнеры.
- **Контравариантность** (`contravariant=True`): наоборот, `Consumer[Animal]` — подтип `Consumer[Cat]`. Подходит для «только на вход» (потребители, callback'и принимающие тип).
- **Инвариантность** (по умолчанию): `list[Cat]` НЕ является подтипом `list[Animal]` (потому что в список можно и положить, и взять).

```python
from typing import TypeVar, Generic

T_co = TypeVar("T_co", covariant=True)

class Box(Generic[T_co]):
    def __init__(self, value: T_co) -> None:
        self._value = value
    def get(self) -> T_co:   # только отдаёт — ковариантность безопасна
        return self._value

class Animal: ...
class Cat(Animal): ...

box_cat: Box[Cat] = Box(Cat())
box_animal: Box[Animal] = box_cat  # OK при covariant=True
```

### Generic классы

```python
from typing import Generic, TypeVar

T = TypeVar("T")

class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

    def peek(self) -> T | None:
        return self._items[-1] if self._items else None

stack: Stack[int] = Stack()
stack.push(1)
stack.push(2)
value = stack.pop()   # mypy знает: value — int
# stack.push("x")     # ОШИБКА статической проверки

# Несколько параметров типа:
K = TypeVar("K")
V = TypeVar("V")

class Pair(Generic[K, V]):
    def __init__(self, key: K, value: V) -> None:
        self.key = key
        self.value = value
```

**Новый синтаксис PEP 695 (Python 3.12+)** — без явного `TypeVar`/`Generic`:

```python
# Python 3.12+:
class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []
    def push(self, item: T) -> None:
        self._items.append(item)
    def pop(self) -> T:
        return self._items.pop()

def first[T](items: list[T]) -> T:   # дженерик-функция без TypeVar
    return items[0]
```

### Protocol — структурная типизация

Протокол описывает «форму» объекта (какие методы/атрибуты он должен иметь), без наследования.

```python
from typing import Protocol

class SupportsClose(Protocol):
    def close(self) -> None: ...

def close_it(resource: SupportsClose) -> None:
    resource.close()

class File:
    def close(self) -> None:
        print("closed")
    # File НЕ наследует SupportsClose, но подходит структурно!

close_it(File())  # OK: у File есть метод close()
```

**`runtime_checkable`** — позволяет использовать `isinstance` с протоколом (проверяет только наличие методов, не их сигнатуры):

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Drawable(Protocol):
    def draw(self) -> None: ...

class Circle:
    def draw(self) -> None: ...

print(isinstance(Circle(), Drawable))  # True (в рантайме!)
# Внимание: проверяет только наличие 'draw', но НЕ его сигнатуру/аргументы.
```

### Literal — конкретные значения как тип

```python
from typing import Literal

def set_mode(mode: Literal["r", "w", "a"]) -> None:
    ...

set_mode("r")     # OK
# set_mode("x")   # ОШИБКА: только "r", "w" или "a"

# Литералы можно объединять и для чисел/булевых:
Direction = Literal["north", "south", "east", "west"]
Flag = Literal[0, 1]
def move(d: Direction) -> None: ...
```

### Final — запрет переопределения/переприсваивания

```python
from typing import Final

MAX_SIZE: Final = 100          # тип выведется как int, переприсвоение запрещено
PI: Final[float] = 3.14159

# MAX_SIZE = 200  # ОШИБКА статической проверки: нельзя переприсвоить Final

class Config:
    VERSION: Final = "1.0"     # нельзя переопределить в подклассах

# @final на классе/методе — запрет наследования/переопределения:
from typing import final

@final
class Singleton:        # нельзя наследоваться
    ...

class Base:
    @final
    def critical(self) -> None:  # нельзя переопределить в подклассе
        ...
```

### TypedDict — типизированные словари

Описывает словарь с фиксированным набором ключей и типами значений.

```python
from typing import TypedDict

class Movie(TypedDict):
    title: str
    year: int

m: Movie = {"title": "Inception", "year": 2010}  # OK
# m2: Movie = {"title": "X"}  # ОШИБКА: пропущен ключ 'year'
# m3: Movie = {"title": "X", "year": "2010"}  # ОШИБКА: year должен быть int

# Альтернативный синтаксис (когда ключ — не валидный идентификатор):
Movie2 = TypedDict("Movie2", {"title": str, "release-year": int})
```

**`total=False`** — все ключи становятся необязательными:

```python
class MovieOptional(TypedDict, total=False):
    title: str
    year: int

m: MovieOptional = {}                  # OK: все ключи опциональны
m2: MovieOptional = {"title": "X"}     # OK
```

**`Required` / `NotRequired`** (PEP 655, Python 3.11+ или через `typing_extensions`) — точечное управление обязательностью:

```python
from typing import TypedDict, Required, NotRequired

class User(TypedDict):
    id: int                       # обязательный
    name: str                     # обязательный
    email: NotRequired[str]       # необязательный

class Partial(TypedDict, total=False):
    id: Required[int]             # обязательный, несмотря на total=False
    name: str                     # необязательный
```

### NewType — отдельный «номинальный» тип

Создаёт лёгкий подтип, который статически отличается, но в рантайме это исходный тип.

```python
from typing import NewType

UserId = NewType("UserId", int)
ProductId = NewType("ProductId", int)

def get_user(user_id: UserId) -> str:
    ...

uid = UserId(42)      # в рантайме просто int(42)
get_user(uid)         # OK
# get_user(42)        # ОШИБКА: 42 — это int, а не UserId
# get_user(ProductId(1))  # ОШИБКА: ProductId != UserId

print(type(uid))      # <class 'int'> — в рантайме это int!
```

Польза: защищает от путаницы взаимозаменяемых на вид значений (id пользователя vs id товара) без оверхеда классов.

### Any vs object

```python
from typing import Any

# object — общий супертип. С ним можно делать только то, что есть у object.
def f(x: object) -> None:
    # x.foo()      # ОШИБКА: у object нет метода foo
    print(x)       # OK: __str__ есть у всех
    if isinstance(x, str):
        print(x.upper())  # OK после сужения типа

# Any — «выключает» проверку типов. Любая операция считается допустимой.
def g(x: Any) -> None:
    x.foo()        # OK для mypy (проверка отключена)
    x + 1          # OK
    x[0].bar()     # OK
```

Ключевая разница:
- **`object`** — безопасно: вы ничего не можете делать с объектом, пока не сузите тип. Это «честный» верхний тип.
- **`Any`** — «дыра» в системе типов: совместим с чем угодно в обе стороны, проверка отключается. Используйте осознанно и минимально.

```python
x: Any = get_something()
y: int = x        # OK (Any совместим с int)
z: str = x        # OK (Any совместим с str)

a: object = get_something()
b: int = a        # ОШИБКА: object не совместим с int без cast/проверки
```

### cast — принудительное указание типа анализатору

`cast` ничего не делает в рантайме (просто возвращает значение). Говорит анализатору: «доверься, тут именно такой тип».

```python
from typing import cast

data: object = ["a", "b", "c"]

# Мы точно знаем, что это list[str], но анализатор — нет:
strings = cast(list[str], data)
strings[0].upper()   # теперь mypy доволен

# В рантайме cast(T, x) просто возвращает x без проверок:
# def cast(typ, val): return val
```

Использовать аккуратно — это «обещание» анализатору, которое он не проверяет.

### ClassVar — атрибут класса (не экземпляра)

```python
from typing import ClassVar

class Counter:
    count: ClassVar[int] = 0      # общий для всех экземпляров (атрибут класса)
    name: str                     # атрибут экземпляра

    def __init__(self, name: str) -> None:
        self.name = name
        Counter.count += 1

# ClassVar говорит анализатору: это поле класса, его нельзя
# присваивать через self как атрибут экземпляра.
c = Counter("a")
# c.count = 5  # ОШИБКА (mypy): нельзя присвоить ClassVar через экземпляр
```

Особенно важно в `dataclass`: поля с `ClassVar` НЕ становятся полями датакласса (не попадают в `__init__`).

### Annotated — метаданные к типу

```python
from typing import Annotated

# Annotated[тип, метаданные...] — тип + произвольная доп. информация.
# Для анализатора тип = первый аргумент; остальное читают библиотеки.
Age = Annotated[int, "должно быть >= 0"]

def set_age(age: Age) -> None: ...

# Реальное применение — FastAPI / pydantic:
# def endpoint(q: Annotated[str, Query(max_length=50)]): ...

# Достать метаданные можно через get_type_hints(..., include_extras=True)
```

### overload — несколько сигнатур одной функции

Позволяет описать разные комбинации входов/выходов. Все `@overload`-объявления — только сигнатуры (тело `...`), плюс одна реальная реализация.

```python
from typing import overload

@overload
def process(x: int) -> str: ...
@overload
def process(x: str) -> int: ...

def process(x: int | str) -> int | str:   # реальная реализация
    if isinstance(x, int):
        return str(x)
    return len(x)

a = process(5)      # mypy знает: a — str
b = process("hi")   # mypy знает: b — int
```

### Self (Python 3.11+) — тип «текущего класса»

```python
from typing import Self

class Builder:
    def __init__(self) -> None:
        self._parts: list[str] = []

    def add(self, part: str) -> Self:    # вернёт тип конкретного подкласса
        self._parts.append(part)
        return self

    def build(self) -> str:
        return " ".join(self._parts)

class AdvancedBuilder(Builder):
    def add_special(self) -> Self: ...

# Благодаря Self, add() в подклассе возвращает AdvancedBuilder, а не Builder:
result = AdvancedBuilder().add("x").add_special()  # тип сохраняется
```

До 3.11 это делали через `TypeVar("T", bound="Builder")` и аннотацию `-> T`.

### ParamSpec и Concatenate (Python 3.10+) — сохранение сигнатуры

Нужны для декораторов, чтобы сохранить типы аргументов оборачиваемой функции.

```python
from typing import ParamSpec, TypeVar, Callable
from functools import wraps

P = ParamSpec("P")    # «захватывает» все параметры (args/kwargs)
R = TypeVar("R")      # тип возврата

def logged(func: Callable[P, R]) -> Callable[P, R]:
    @wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        print(f"Вызов {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@logged
def greet(name: str, times: int) -> str:
    return f"Hi {name}" * times

greet("Bob", 2)     # сигнатура сохранена: mypy проверит аргументы
# greet(123)        # ОШИБКА: типы и количество аргументов проверяются
```

**`Concatenate`** — добавить/убрать ведущие параметры:

```python
from typing import Concatenate, ParamSpec, TypeVar, Callable

P = ParamSpec("P")
R = TypeVar("R")

# Декоратор, который сам подставляет первый аргумент (например, request):
def with_request(
    func: Callable[Concatenate[str, P], R]
) -> Callable[P, R]:
    def inner(*args: P.args, **kwargs: P.kwargs) -> R:
        return func("REQUEST", *args, **kwargs)
    return inner
```

### TypeAlias и новый синтаксис `type X = ...`

```python
from typing import TypeAlias, Union

# Старый способ — просто присваивание (анализатор «угадывает», что это алиас):
Vector = list[float]

# Явный псевдоним типа (PEP 613, Python 3.10+):
Matrix: TypeAlias = list[list[float]]
JsonValue: TypeAlias = Union[str, int, float, bool, None, list, dict]

def scale(v: Vector, factor: float) -> Vector:
    return [x * factor for x in v]
```

**Новый синтаксис PEP 695 (Python 3.12+):**

```python
# Python 3.12+:
type Vector = list[float]
type Matrix = list[list[float]]
type Pair[T] = tuple[T, T]        # дженерик-алиас

def make_pair[T](x: T) -> Pair[T]:
    return (x, x)
```

Преимущество `type X = ...`: алиас «ленивый» (отложенно вычисляемый), поддерживает рекурсивные и дженерик-алиасы из коробки.

### type[X] — сам класс как значение (а не экземпляр)

```python
def create(cls: type[int]) -> int:
    return cls()        # вызываем сам класс

create(int)             # OK: передаём класс int, а не экземпляр

class Animal: ...
class Dog(Animal): ...

def factory(cls: type[Animal]) -> Animal:
    return cls()

factory(Dog)            # OK: type[Dog] совместим с type[Animal]
# factory(Dog())        # ОШИБКА: передан экземпляр, а нужен класс
```

`type[X]` — то, что раньше писали как `Type[X]` из `typing`.

### Never / NoReturn — функция никогда не возвращает значение

```python
from typing import NoReturn, Never  # Never — с Python 3.11

def fail(msg: str) -> NoReturn:      # всегда выбрасывает исключение
    raise ValueError(msg)

def infinite() -> NoReturn:          # бесконечный цикл, не возвращается
    while True:
        ...

# Never (3.11+) — «низший» тип (bottom type), полезен для проверки полноты:
def handle(x: int | str) -> str:
    if isinstance(x, int):
        return "int"
    elif isinstance(x, str):
        return "str"
    else:
        # сюда не дойдём; если добавят новый тип в Union — mypy укажет
        assert_never(x)  # x здесь имеет тип Never

from typing import assert_never  # 3.11+
```

`NoReturn` и `Never` семантически близки; `Never` — более общее имя «типа без значений», `NoReturn` — про функции.

### get_type_hints — получить аннотации с разрешением строк

```python
from typing import get_type_hints

class C:
    x: "int"          # аннотация-строка (отложенная)
    y: list[str]

# __annotations__ вернёт строки как есть:
print(C.__annotations__)             # {'x': 'int', 'y': list[str]}

# get_type_hints разрешит строки в реальные объекты типов:
print(get_type_hints(C))             # {'x': <class 'int'>, 'y': list[str]}

# С include_extras=True сохраняет метаданные Annotated:
from typing import Annotated, get_type_hints

def f(a: Annotated[int, "meta"]) -> None: ...
print(get_type_hints(f, include_extras=True))
# {'a': Annotated[int, 'meta'], 'return': <class 'NoneType'>}
```

`get_type_hints` — основной способ для рантайм-инструментов (pydantic и т.п.) читать аннотации, корректно обрабатывая строковые/отложенные аннотации.

### `from __future__ import annotations` — отложенные аннотации (PEP 563)

```python
from __future__ import annotations

class Node:
    def __init__(self, value: int) -> None:
        self.value = value
        self.next: Node | None = None   # ссылка на сам класс без кавычек!

    def link(self, other: Node) -> Node:  # 'Node' ещё не определён до конца тела,
        self.next = other                  # но с future-импортом это работает
        return other
```

Что делает этот импорт:
- **Все аннотации становятся строками** (не вычисляются при определении функции/класса).
- Решает проблему **forward references** (ссылка на ещё не определённый класс) без кавычек.
- Слегка ускоряет импорт модуля (аннотации не вычисляются).
- **Минус**: рантайм-инструменты должны использовать `get_type_hints()` для разрешения строк; прямое чтение `__annotations__` вернёт строки.

```python
from __future__ import annotations

def f(x: int) -> str: ...
print(f.__annotations__)   # {'x': 'int', 'return': 'str'} — СТРОКИ!
```

Без `from __future__ import annotations` forward reference нужно оборачивать в кавычки:

```python
class Tree:
    def add(self, child: "Tree") -> None:  # кавычки обязательны
        ...
```

### mypy — статический анализатор

```bash
# Установка:
pip install mypy

# Проверка файла/пакета:
mypy script.py
mypy mypackage/

# Строгий режим (включает много проверок сразу):
mypy --strict script.py

# Полезные флаги:
mypy --disallow-untyped-defs app.py    # запретить нетипизированные функции
mypy --ignore-missing-imports app.py   # игнорировать отсутствие stubs
mypy --check-untyped-defs app.py       # проверять тела нетипизированных функций
```

Конфигурация в `pyproject.toml`:

```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_unused_ignores = true
warn_return_any = true
disallow_untyped_defs = true
```

Типичные ошибки mypy:

```python
def f(x: int) -> int:
    return x

f("hello")
# error: Argument 1 to "f" has incompatible type "str"; expected "int"

y: int = "abc"
# error: Incompatible types in assignment (expression has type "str",
#        variable has type "int")

def g(x: int | None) -> int:
    return x + 1
# error: Unsupported operand types for + ("None" and "int")
# (нужно сначала проверить на None)
```

`# type: ignore` — подавить конкретную ошибку (лучше с кодом: `# type: ignore[arg-type]`).

### pyright — анализатор от Microsoft (движок Pylance в VS Code)

```bash
npm install -g pyright    # или pip install pyright
pyright script.py
```

Особенности:
- Очень быстрый (написан на TypeScript), хорошая инкрементальная проверка.
- Режимы `basic` и `strict` (в `pyrightconfig.json` или `pyproject.toml`).
- Поддерживает **сужение типов** (narrowing) обычно глубже и точнее, чем mypy.
- По умолчанию встроен в VS Code (Pylance) — даёт подсказки прямо в редакторе.

```json
// pyrightconfig.json
{
  "typeCheckingMode": "strict",
  "pythonVersion": "3.12"
}
```

mypy vs pyright кратко: mypy — «эталонная» реализация PEP, де-факто стандарт в CI; pyright — быстрее и лучше интегрирован в редактор. Часто используют pyright в IDE + mypy в CI.

---

## Частые вопросы на собеседовании

**Q1: Проверяет ли Python аннотации типов в рантайме?**
A: Нет. Интерпретатор хранит аннотации в `__annotations__`, но не проверяет и не приводит типы при выполнении. Проверку делают внешние инструменты (mypy, pyright) ДО запуска. `def f(x: int)` спокойно примет строку в рантайме.

**Q2: В чём разница между `Optional[int]` и аргументом со значением по умолчанию?**
A: `Optional[int]` == `int | None` — это про **значение**, которое может быть None. Необязательность аргумента задаётся `= значение` в сигнатуре, а не `Optional`. `def f(x: Optional[int])` всё равно требует передать аргумент (можно `None`).

**Q3: Чем отличается `Any` от `object`?**
A: `object` — честный супертип: с ним нельзя делать ничего специфичного без сужения типа, проверки работают. `Any` отключает проверку типов («дыра»): совместим с чем угодно в обе стороны, любая операция считается валидной. `object` безопасен, `Any` опасен.

**Q4: Что такое `Protocol` и чем он лучше абстрактных базовых классов?**
A: `Protocol` реализует структурную («утиную») типизацию: объект подходит, если у него есть нужные методы/атрибуты, без явного наследования. ABC требует наследования (номинальная типизация). Protocol удобен для типизации сторонних классов, которые вы не можете изменить.

**Q5: Объясните ковариантность, контравариантность и инвариантность.**
A: Касается совместимости дженериков при наследовании параметров.
- Ковариантность (`covariant=True`): `Box[Cat]` — подтип `Box[Animal]`. Для «только на выход».
- Контравариантность (`contravariant=True`): `Handler[Animal]` — подтип `Handler[Cat]`. Для «только на вход».
- Инвариантность (по умолчанию): `list[Cat]` не совместим с `list[Animal]`, потому что список и читают, и пишут.

**Q6: В чём разница `TypeVar` с `bound` и с `constraints`?**
A: `bound=X` — тип может быть любым подтипом X, конкретный подтип сохраняется в выводе. `constraints=(A, B)` — тип должен быть ровно одним из перечисленных (A или B), без их смешивания. `AnyStr = TypeVar("AnyStr", str, bytes)` — пример constraints: нельзя смешать str и bytes в одном вызове.

**Q7: Что делает `cast` и есть ли у него рантайм-эффект?**
A: `cast(T, value)` сообщает анализатору, что `value` имеет тип `T`. В рантайме просто возвращает значение без проверок и преобразований. Это «обещание» анализатору, которое он принимает на веру.

**Q8: Зачем `from __future__ import annotations`?**
A: Делает все аннотации строками (отложенное вычисление, PEP 563). Решает проблему forward references (ссылка на ещё не объявленный класс) без кавычек, ускоряет импорт. Минус: рантайм-чтение аннотаций требует `get_type_hints()` для разрешения строк.

**Q9: Что такое `TypedDict` и зачем `total=False`, `Required`/`NotRequired`?**
A: `TypedDict` типизирует словарь с известными ключами и типами значений (полезно для JSON/конфигов). `total=False` делает все ключи необязательными. `Required`/`NotRequired` (3.11+) точечно задают обязательность отдельных ключей независимо от `total`.

**Q10: Чем `NewType` отличается от обычного алиаса типа?**
A: Алиас (`UserId = int`) — просто другое имя того же типа, взаимозаменяем с исходным. `NewType("UserId", int)` создаёт **отдельный номинальный тип**: статически нельзя передать обычный `int` туда, где ждут `UserId`. В рантайме `NewType` — это исходный тип (просто функция-идентичность).

**Q11: Зачем нужны `ParamSpec` и `Concatenate`?**
A: Для типизации декораторов с сохранением сигнатуры оборачиваемой функции. `ParamSpec` «захватывает» все параметры (`*args`/`**kwargs`), `Concatenate` позволяет добавить/убрать ведущие параметры. Без них декоратор «стирал» бы типы аргументов в `Callable[..., R]`.

**Q12: Что такое `Self` (3.11) и как делали раньше?**
A: `Self` — тип «текущего класса», корректно работает с наследованием (метод вернёт тип подкласса, а не базового). Полезен для fluent-интерфейсов (`return self`) и фабрик. До 3.11 использовали `TypeVar("T", bound="MyClass")` и возвращали `T`.

**Q13: Чем `NoReturn`/`Never` полезны?**
A: `NoReturn` помечает функцию, которая никогда не возвращает значение (всегда raise или бесконечный цикл) — анализатор учитывает это при анализе потока. `Never` (3.11) — «низший» тип без значений; с `assert_never` обеспечивает **проверку полноты** (exhaustiveness) при разборе Union/перечислений.

**Q14: Как библиотеки вроде pydantic/FastAPI используют аннотации, если рантайм их игнорирует?**
A: Они сами читают аннотации через интроспекцию (`get_type_hints`, `__annotations__`) и реализуют собственную валидацию/сериализацию в рантайме. Это не интерпретатор Python проверяет типы, а код библиотеки, который осознанно интерпретирует подсказки типов.

**Q15: В чём разница mypy и pyright?**
A: mypy — эталонная реализация проверки типов, де-факто стандарт в CI. pyright (Microsoft) — быстрее (TS), глубже сужение типов, встроен в VS Code (Pylance). Частая практика: pyright в редакторе для скорости + mypy в CI как «авторитет».

---

## Подводные камни (gotchas)

**1. Аннотации не валидируют в рантайме.** Нельзя полагаться на них как на защиту от неверных данных:

```python
def f(x: int) -> int:
    return x * 2

f("abc")  # вернёт 'abcabc' — никакой ошибки типа!
```

**2. `Optional` ≠ необязательный аргумент.** Частая путаница (см. Q2).

**3. Изменяемые значения по умолчанию + типы не спасают:**

```python
def add(item: int, lst: list[int] = []) -> list[int]:  # классическая ловушка!
    lst.append(item)
    return lst
# Аннотация list[int] не мешает разделяемому изменяемому дефолту.
# Правильно: lst: list[int] | None = None, и внутри lst = lst or []
```

**4. `list[int]` и подобное в `isinstance` не работает:**

```python
# isinstance(x, list[int])  # TypeError в рантайме!
isinstance(x, list)          # OK — только базовый тип, без параметра
```

**5. Инвариантность списков удивляет:**

```python
def f(items: list[float]) -> None: ...
nums: list[int] = [1, 2, 3]
# f(nums)  # ОШИБКА mypy: list[int] не подтип list[float] (инвариантность)
# Используйте Sequence[float] (ковариантный) для аргументов «только на чтение».
```

**6. `Any` «заражает» код.** Значение `Any` распространяет отключение проверок дальше по цепочке вычислений. Минимизируйте его.

**7. `runtime_checkable` Protocol проверяет только наличие методов, не сигнатуры:**

```python
@runtime_checkable
class P(Protocol):
    def m(self, x: int) -> int: ...

class Bad:
    def m(self): ...   # неверная сигнатура

isinstance(Bad(), P)   # True! проверяется только наличие 'm'
```

**8. Forward reference без кавычек/future-импорта падает:**

```python
class Node:
    next: Node  # NameError при определении класса (Node ещё не готов)!
    # нужно: "Node" или from __future__ import annotations
```

**9. `from __future__ import annotations` ломает наивное чтение аннотаций:**

```python
from __future__ import annotations
def f(x: int) -> None: ...
f.__annotations__["x"]  # это строка 'int', а не класс int!
# Используйте get_type_hints(f).
```

**10. `cast` лжёт безнаказанно.** Неверный `cast` не проверяется и приводит к скрытым ошибкам в рантайме:

```python
x = cast(int, "not a number")  # mypy верит, но x — строка!
x + 1                          # TypeError в рантайме
```

**11. `TypedDict` не проверяется в рантайме при создании:**

```python
m: Movie = {"title": "X", "year": 2020}  # это обычный dict в рантайме
# get_type_hints не валидирует значения — только статическая проверка.
```

**12. Mutable дефолты в `dataclass` и аннотации:** для изменяемых дефолтов нужен `field(default_factory=...)`, аннотация типа сама по себе не решает проблему.

---

## Лучшие практики

1. **Используйте новый синтаксис** (Python 3.9+/3.10+): `list[int]`, `dict[str, int]`, `int | None` вместо `List`, `Dict`, `Optional`. Чище и без лишних импортов.
2. **Типизируйте публичные API полностью** (аргументы и возврат). Внутри функций можно скромнее.
3. **Аргументы — абстрактные типы, возврат — конкретные.** Принимайте `Sequence`/`Iterable`/`Mapping`, возвращайте `list`/`dict`. («Будьте либеральны к входу, строги к выходу».)
4. **Избегайте `Any`.** Если не знаете тип — попробуйте `object` + сужение, дженерик или `Protocol`.
5. **`Protocol` для интерфейсов** сторонних/несвязанных классов вместо навязывания наследования от ABC.
6. **Включайте mypy/pyright в CI**, желательно `--strict` для нового кода. Постепенно ужесточайте для легаси.
7. **`Final` для констант** и `@final` для классов/методов, которые нельзя расширять.
8. **`NewType` для семантически разных id** (UserId vs OrderId), чтобы не перепутать.
9. **`Self` (3.11+)** для fluent-интерфейсов и методов-фабрик вместо `TypeVar(bound=...)`.
10. **`from __future__ import annotations`** в новых модулях — упрощает forward references; помните про `get_type_hints` для рантайм-чтения.
11. **`ParamSpec` для декораторов**, чтобы не терять типы сигнатур.
12. **`assert_never` + `Never`** для проверки полноты разбора Union/Enum — компилятор укажет на забытую ветку.
13. **`# type: ignore[код]`** с конкретным кодом ошибки, а не голый ignore.
14. **Не дублируйте проверки**: для валидации входных ДАННЫХ в рантайме используйте pydantic/attrs/ручные проверки — типы это не заменят.

---

## Шпаргалка

```python
# ── Коллекции (PEP 585, 3.9+) ──────────────────────────────
list[int]                       # список int
dict[str, int]                  # словарь str -> int
tuple[int, str]                 # фикс. кортеж из 2 элементов
tuple[int, ...]                 # кортеж любой длины из int
set[int]                        # множество int
frozenset[str]                  # неизменяемое множество

# ── Union / Optional (PEP 604, 3.10+) ──────────────────────
int | str                       # Union[int, str]
int | None                      # Optional[int]
from typing import Optional, Union   # старый стиль

# ── Callable ───────────────────────────────────────────────
from typing import Callable
Callable[[int, str], bool]      # (int, str) -> bool
Callable[..., int]              # любые аргументы -> int

# ── Дженерики ──────────────────────────────────────────────
from typing import TypeVar, Generic
T = TypeVar("T")
T = TypeVar("T", bound=Base)        # верхняя граница
T = TypeVar("T", int, str)          # constraints
T = TypeVar("T", covariant=True)    # ковариантный
class Box(Generic[T]): ...
# Python 3.12+: class Box[T]: ...   def f[T](x: T) -> T: ...

# ── Протоколы (структурная типизация) ──────────────────────
from typing import Protocol, runtime_checkable
class HasLen(Protocol):
    def __len__(self) -> int: ...
@runtime_checkable                  # для isinstance (только наличие методов)
class Closeable(Protocol):
    def close(self) -> None: ...

# ── Спец-типы ──────────────────────────────────────────────
from typing import Literal, Final, Any, NoReturn, Never, ClassVar, Self
Literal["r", "w", "a"]          # только эти значения
Final                           # запрет переприсваивания
Any                             # отключение проверок (опасно)
NoReturn / Never                # функция не возвращается / bottom type
ClassVar[int]                   # атрибут класса
Self                            # тип текущего класса (3.11+)

# ── TypedDict ──────────────────────────────────────────────
from typing import TypedDict, Required, NotRequired
class TD(TypedDict):
    a: int                      # обязательный
    b: NotRequired[str]         # необязательный
class TD2(TypedDict, total=False):  # все ключи необязательны
    x: Required[int]            # кроме явно Required

# ── Прочее ─────────────────────────────────────────────────
from typing import NewType, cast, Annotated, overload, TypeAlias
UserId = NewType("UserId", int)         # номинальный подтип
cast(list[str], obj)                    # подсказка анализатору (no-op в рантайме)
Annotated[int, "meta"]                  # тип + метаданные
@overload                               # несколько сигнатур
Matrix: TypeAlias = list[list[float]]   # явный алиас (3.10+)
# Python 3.12+:  type Matrix = list[list[float]]
type[int]                               # сам класс, не экземпляр

# ── ParamSpec / Concatenate (3.10+) ────────────────────────
from typing import ParamSpec, Concatenate
P = ParamSpec("P")
Callable[P, R]                          # сохранить сигнатуру (декораторы)
Callable[Concatenate[str, P], R]        # с доп. ведущим аргументом

# ── Forward references / отложенные ────────────────────────
from __future__ import annotations      # все аннотации -> строки (PEP 563)
next: "Node"                            # или кавычки для forward ref
from typing import get_type_hints       # разрешить строки в типы

# ── Инструменты ────────────────────────────────────────────
# mypy script.py            mypy --strict pkg/
# pyright script.py         (strict в pyrightconfig.json)
# # type: ignore[code]      подавить конкретную ошибку
```

**Главное, что нужно помнить:** аннотации типов — это инструмент статического анализа и документации. Интерпретатор Python их НЕ проверяет в рантайме. Безопасность типов обеспечивают mypy/pyright ДО запуска, а валидацию реальных данных — отдельные библиотеки (pydantic/attrs), которые сами читают аннотации.
