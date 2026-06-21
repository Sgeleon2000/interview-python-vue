# ABC (абстрактные базовые классы) — подготовка к собеседованию

## Что это и зачем

Модуль `abc` (Abstract Base Classes) — это часть стандартной библиотеки Python, которая
предоставляет инфраструктуру для определения **абстрактных базовых классов**. Абстрактный
класс — это класс, который **нельзя инстанцировать напрямую** и который служит «контрактом»
(интерфейсом) для своих наследников: он описывает, какие методы/свойства обязан реализовать
подкласс, чтобы считаться полноценным.

В Python нет ключевого слова `interface`, как в Java или C#. Роль интерфейсов и абстрактных
классов выполняет именно модуль `abc` совместно с декоратором `@abstractmethod`.

Зачем это нужно:

- **Явный контракт.** Абстрактный класс фиксирует, какие методы должны быть реализованы.
  Если подкласс забыл реализовать абстрактный метод — Python не даст создать его экземпляр,
  и ошибка возникнет рано (`TypeError` при инстанцировании), а не в неожиданном месте кода.
- **Документирование намерений.** Читая абстрактный класс, разработчик сразу видит ожидаемый
  API.
- **Кастомизация `isinstance` / `issubclass`.** Через `register` и `__subclasshook__` можно
  заставить `isinstance(obj, MyABC)` возвращать `True` даже для классов, которые формально не
  наследуются от `MyABC`. На этом построен весь модуль `collections.abc`.
- **Полиморфизм и проверки типов.** Можно проверять «является ли объект итерируемым / хешируемым /
  отображением» одной проверкой `isinstance(x, Iterable)`, не выясняя конкретный тип.

Сравнение подходов к «контрактам» в Python:

| Подход | Когда проверяется | Наследование | Где живёт |
|--------|-------------------|--------------|-----------|
| Утиная типизация (duck typing) | в рантайме при вызове | не требуется | везде |
| ABC (`abc`) | при инстанцировании (для abstractmethod) + `isinstance` | обычно явное (или `register`) | рантайм |
| `typing.Protocol` | статически (mypy) + опционально в рантайме | структурное, без наследования | в основном type-checker |

```python
from abc import ABC, abstractmethod


class Animal(ABC):
    """Абстрактный базовый класс — задаёт контракт для всех животных."""

    @abstractmethod
    def make_sound(self) -> str:
        """Каждое животное обязано уметь издавать звук."""
        ...


class Dog(Animal):
    def make_sound(self) -> str:
        return "Гав"


# Animal()  -> TypeError: Can't instantiate abstract class Animal
#              with abstract method make_sound
print(Dog().make_sound())  # Гав
```

---

## Ключевые концепции

1. **Метакласс `ABCMeta`.** Вся «магия» абстрактности реализована не в классе, а в
   **метаклассе** `ABCMeta`. Именно он:
   - запоминает множество абстрактных методов в атрибуте `__abstractmethods__`;
   - запрещает инстанцирование, если множество не пусто;
   - реализует механизм `register` и вызов `__subclasshook__` при `isinstance`/`issubclass`.

2. **Класс `ABC`.** Это просто удобный «хелпер»: `class ABC(metaclass=ABCMeta)`. Наследоваться
   от `ABC` проще, чем писать `metaclass=ABCMeta` вручную, особенно для тех, кто не знаком с
   метаклассами.

3. **Декоратор `@abstractmethod`.** Помечает метод как абстрактный (ставит флаг
   `__isabstractmethod__ = True`). Сам по себе декоратор работает **только** в связке с
   `ABCMeta` — без метакласса флаг ни на что не влияет.

4. **`__abstractmethods__`** — `frozenset` имён нерреализованных абстрактных методов. Пока он
   непустой — экземпляр создать нельзя.

5. **Виртуальные подклассы (`register`).** Можно «зарегистрировать» произвольный класс как
   подкласс ABC. Тогда `isinstance`/`issubclass` будут это учитывать, **но реальное
   наследование не происходит** (нет наследования атрибутов, нет проверки реализации методов).

6. **`__subclasshook__`** — позволяет переопределить логику `issubclass` (например, считать
   подклассом любой класс, у которого есть нужный метод). Это «структурная» проверка.

7. **Раннее vs позднее обнаружение ошибок.** Абстрактные методы дают ошибку рано (при создании
   объекта). Утиная типизация — поздно (при вызове отсутствующего метода).

---

## Основные функции/классы/методы

### `ABC` и `ABCMeta`

Два эквивалентных способа создать абстрактный класс:

```python
from abc import ABC, ABCMeta, abstractmethod


# Способ 1 — через наследование от ABC (рекомендуется, читабельнее)
class Repository(ABC):
    @abstractmethod
    def get(self, id_: int): ...


# Способ 2 — через явное указание метакласса (нужно, если есть свой метакласс)
class Repository2(metaclass=ABCMeta):
    @abstractmethod
    def get(self, id_: int): ...
```

Когда нужен `metaclass=ABCMeta` вместо `ABC`? Если у класса уже есть собственный метакласс
и нужно его «смешать» с `ABCMeta` (нельзя одновременно наследоваться от `ABC` и иметь другой
несовместимый метакласс — придётся создать общий метакласс-наследник обоих).

### `@abstractmethod`

```python
from abc import ABC, abstractmethod


class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        """Абстрактный метод. Тело можно оставить пустым (...)."""
        ...

    @abstractmethod
    def perimeter(self) -> float:
        ...

    def describe(self) -> str:
        # Обычный (конкретный) метод может опираться на абстрактные.
        return f"Площадь={self.area():.2f}, Периметр={self.perimeter():.2f}"


class Rectangle(Shape):
    def __init__(self, w: float, h: float):
        self.w, self.h = w, h

    def area(self) -> float:
        return self.w * self.h

    def perimeter(self) -> float:
        return 2 * (self.w + self.h)


r = Rectangle(3, 4)
print(r.describe())  # Площадь=12.00, Периметр=14.00
print(Shape.__abstractmethods__)  # frozenset({'area', 'perimeter'})
```

Важно: абстрактный метод **может иметь реализацию**, и подкласс может вызвать её через `super()`:

```python
class Base(ABC):
    @abstractmethod
    def load(self):
        # Полезная общая логика, доступная через super().load()
        print("Базовая загрузка")


class Concrete(Base):
    def load(self):
        super().load()          # вызываем реализацию абстрактного метода
        print("Дополнительная загрузка")


Concrete().load()
# Базовая загрузка
# Дополнительная загрузка
```

### Абстрактные свойства: `@property` + `@abstractmethod`

`abstractproperty` **устарел** (deprecated с Python 3.3). Правильный способ — комбинировать
`@property` и `@abstractmethod`. **Порядок важен:** `@abstractmethod` должен стоять «внутри»,
то есть быть применён последним (ближе к функции).

```python
from abc import ABC, abstractmethod


class Config(ABC):

    @property
    @abstractmethod
    def db_url(self) -> str:
        """Абстрактное read-only свойство."""
        ...

    # Абстрактное свойство с сеттером
    @property
    @abstractmethod
    def timeout(self) -> int:
        ...

    @timeout.setter
    @abstractmethod
    def timeout(self, value: int) -> None:
        ...


class ProdConfig(Config):
    def __init__(self):
        self._timeout = 30

    @property
    def db_url(self) -> str:
        return "postgres://prod"

    @property
    def timeout(self) -> int:
        return self._timeout

    @timeout.setter
    def timeout(self, value: int) -> None:
        self._timeout = value


c = ProdConfig()
print(c.db_url)   # postgres://prod
c.timeout = 60
print(c.timeout)  # 60
```

Устаревший вариант (только для понимания легаси-кода, **не использовать**):

```python
from abc import ABC, abstractproperty


class Old(ABC):
    @abstractproperty          # DeprecationWarning
    def name(self): ...
```

### `abstractclassmethod` и `abstractstaticmethod` — устаревшие

Эти декораторы тоже **устарели** (deprecated с 3.3). Правильный способ — комбинировать
`@classmethod` / `@staticmethod` с `@abstractmethod`. И снова **`@abstractmethod` ставится
последним** (ближайшим к функции).

```python
from abc import ABC, abstractmethod


class Serializer(ABC):

    @classmethod
    @abstractmethod
    def from_dict(cls, data: dict) -> "Serializer":
        """Абстрактный classmethod — фабрика."""
        ...

    @staticmethod
    @abstractmethod
    def format_name() -> str:
        """Абстрактный staticmethod."""
        ...


class JsonSerializer(Serializer):
    def __init__(self, payload):
        self.payload = payload

    @classmethod
    def from_dict(cls, data: dict) -> "JsonSerializer":
        return cls(data)

    @staticmethod
    def format_name() -> str:
        return "json"


print(JsonSerializer.format_name())          # json
print(JsonSerializer.from_dict({"a": 1}).payload)  # {'a': 1}
```

### `register` — виртуальные подклассы

`SomeABC.register(Cls)` делает `Cls` «виртуальным подклассом» `SomeABC`. После этого
`issubclass(Cls, SomeABC)` и `isinstance(Cls(), SomeABC)` возвращают `True`, **хотя
наследования не было**. `register` можно использовать как декоратор.

```python
from abc import ABC, abstractmethod


class Drawable(ABC):
    @abstractmethod
    def draw(self) -> None: ...


# Сторонний класс, который мы не можем/не хотим менять
class ThirdPartyWidget:
    def draw(self) -> None:
        print("рисуем виджет")


Drawable.register(ThirdPartyWidget)

print(issubclass(ThirdPartyWidget, Drawable))   # True
print(isinstance(ThirdPartyWidget(), Drawable)) # True

# ВАЖНО: register НЕ проверяет наличие метода draw!
class Broken:
    pass

Drawable.register(Broken)
print(isinstance(Broken(), Drawable))  # True (!), хотя draw нет

# register как декоратор:
@Drawable.register
class AnotherWidget:
    def draw(self): print("ещё виджет")
```

Ключевые особенности `register`:

- Виртуальный подкласс **не наследует** ничего от ABC (ни методов, ни атрибутов).
- Виртуальный подкласс **не появляется** в MRO (`__mro__`) и в `Broken.__bases__`.
- `register` **не проверяет**, что класс реально реализует абстрактные методы. Ответственность
  на разработчике.
- ABC с непустым `register` всё равно **сам** не инстанцируется, если у него есть
  abstractmethods.

### `__subclasshook__` — структурная проверка подкласса

Переопределив `__subclasshook__`, можно автоматически считать подклассом любой класс,
удовлетворяющий некоторому условию (например, имеющий нужные методы). Именно так работает,
например, `collections.abc.Hashable`.

```python
from abc import ABC, ABCMeta, abstractmethod


class SupportsClose(ABC):
    @abstractmethod
    def close(self) -> None: ...

    @classmethod
    def __subclasshook__(cls, C):
        # Вызывается при issubclass/isinstance.
        if cls is SupportsClose:
            # Считаем подклассом любой класс, у которого в иерархии есть 'close'
            if any("close" in B.__dict__ for B in C.__mro__):
                return True
        return NotImplemented   # NotImplemented -> обычная логика проверки


class File:
    def close(self): ...


print(issubclass(File, SupportsClose))   # True — хотя File не наследник
print(isinstance(File(), SupportsClose)) # True
```

Возвращаемые значения `__subclasshook__`:

- `True` — `C` считается подклассом;
- `False` — `C` точно **не** подкласс (даже при явном наследовании!);
- `NotImplemented` — продолжить стандартную проверку (по `__mro__` и `register`).

### Примеры из `collections.abc`

Стандартный модуль `collections.abc` — эталонный пример использования `abc`. Он определяет
ABC для протоколов контейнеров: `Iterable`, `Iterator`, `Container`, `Sized`, `Hashable`,
`Callable`, `Sequence`, `MutableSequence`, `Mapping`, `MutableMapping`, `Set` и т.д.

Многие из них имеют **mixin-методы**: реализовав минимальный набор абстрактных методов, вы
бесплатно получаете кучу готовых. Например, реализовав `__getitem__` и `__len__` для `Sequence`,
вы получаете `__contains__`, `__iter__`, `__reversed__`, `index`, `count`.

```python
from collections.abc import Iterable, Sequence, Hashable, MutableMapping


# Проверки через isinstance — не зная конкретного типа
print(isinstance([1, 2], Iterable))     # True
print(isinstance("abc", Sequence))      # True
print(isinstance(123, Hashable))        # True
print(isinstance({}, MutableMapping))   # True


# Создаём собственную последовательность, получая mixin-методы «в подарок»
class MyRange(Sequence):
    def __init__(self, n): self.n = n

    def __getitem__(self, i):           # обязательный абстрактный
        if 0 <= i < self.n:
            return i * 10
        raise IndexError(i)

    def __len__(self):                  # обязательный абстрактный
        return self.n


r = MyRange(3)
print(list(r))            # [0, 10, 20]  — __iter__ получен бесплатно
print(20 in r)            # True         — __contains__ бесплатно
print(r.index(10))        # 1            — index() бесплатно
print(list(reversed(r)))  # [20, 10, 0]  — __reversed__ бесплатно
```

`Hashable` использует `__subclasshook__`: любой класс с `__hash__` считается его подклассом.

---

## Частые вопросы на собеседовании

**Q1. Что такое абстрактный базовый класс и зачем он нужен?**
A: Класс, который нельзя инстанцировать и который задаёт контракт (набор обязательных
методов/свойств) для наследников. Нужен для явного описания интерфейса, раннего обнаружения
ошибок (нереализованный метод → `TypeError` при создании объекта), документирования API и
кастомизации `isinstance`/`issubclass`.

**Q2. В чём разница между `ABC` и `ABCMeta`?**
A: `ABCMeta` — это метакласс, где реализована вся логика абстрактности. `ABC` — это удобный
базовый класс `class ABC(metaclass=ABCMeta)`. Наследоваться от `ABC` проще, чем указывать
метакласс. `ABCMeta` нужен напрямую, когда у класса уже есть свой метакласс и его надо
скомбинировать.

**Q3. Что произойдёт, если подкласс не реализует абстрактный метод?**
A: Сам класс определится без ошибок, но при попытке создать его экземпляр будет
`TypeError: Can't instantiate abstract class ... with abstract method ...`. Ошибка возникает
при инстанцировании, а не при объявлении класса.

**Q4. Как сделать абстрактное свойство / classmethod / staticmethod?**
A: Комбинировать соответствующий декоратор с `@abstractmethod`, причём `@abstractmethod` должен
быть **самым внутренним** (применяться последним). Например:
`@property` + `@abstractmethod`, `@classmethod` + `@abstractmethod`. Старые `abstractproperty`,
`abstractclassmethod`, `abstractstaticmethod` устарели с Python 3.3.

**Q5. Что такое виртуальный подкласс и метод `register`?**
A: `register` регистрирует произвольный класс как «виртуальный подкласс» ABC. После этого
`isinstance`/`issubclass` считают его подклассом, **но фактического наследования нет**: нет
наследования методов/атрибутов, класс не появляется в `__mro__`, и не проверяется реализация
абстрактных методов. Удобно для сторонних классов, которые нельзя менять.

**Q6. Чем `register` отличается от обычного наследования?**
A: Наследование даёт реальную иерархию: MRO, наследование реализаций, проверку абстрактных
методов при инстанцировании. `register` — только «обещание» о соответствии типу для проверок
`isinstance`/`issubclass`, без какого-либо переноса кода и без валидации.

**Q7. Что такое `__subclasshook__` и зачем он нужен?**
A: Классметод, переопределяющий логику `issubclass`. Позволяет считать класс подклассом по
структуре (например, при наличии нужных методов), а не по наследованию. Возвращает `True`,
`False` или `NotImplemented` (продолжить стандартную проверку). На нём построен, например,
`collections.abc.Hashable`/`Iterable`.

**Q8. Чем абстрактные классы отличаются от утиной типизации?**
A: Утиная типизация ничего не требует заранее: «если крякает как утка — это утка», ошибка
вылезет при вызове отсутствующего метода (поздно). ABC даёт явный контракт и раннюю проверку
(нельзя создать неполный объект), а также возможность централизованной проверки типа через
`isinstance`.

**Q9. Чем ABC отличается от `typing.Protocol`?**
A: `Protocol` (PEP 544) — структурная типизация: класс соответствует протоколу, если у него
есть нужные методы/атрибуты, **без явного наследования и без register**. Проверка в основном
статическая (mypy/pyright). ABC — номинальная типизация (нужно наследование или register),
проверка в рантайме. `Protocol` можно сделать `@runtime_checkable` для рантайм-`isinstance`,
но он проверит только наличие имён, не сигнатуры.

**Q10. Можно ли в абстрактном методе писать реализацию? Как её вызвать?**
A: Да. Тело абстрактного метода может содержать полезную общую логику. Подкласс обязан
переопределить метод, но может вызвать базовую реализацию через `super().method()`.

**Q11. Что такое `__abstractmethods__`?**
A: `frozenset` с именами ещё не реализованных абстрактных методов класса. Пока он непустой,
инстанцировать класс нельзя. `ABCMeta` вычисляет его автоматически при создании каждого класса.

**Q12. Какие mixin-методы даёт `collections.abc`?**
A: Многие ABC из `collections.abc` предоставляют производные методы поверх минимального набора
абстрактных. Например, реализовав `__getitem__` и `__len__` у наследника `Sequence`, получаешь
`__iter__`, `__contains__`, `__reversed__`, `index`, `count` автоматически.

**Q13. Может ли абстрактный класс иметь обычные (конкретные) методы и атрибуты?**
A: Да. Абстрактный класс может содержать конкретные методы, атрибуты, `__init__`, и они
наследуются. Абстрактность определяется только наличием хотя бы одного `@abstractmethod`
(или другого абстрактного члена).

**Q14. Можно ли наследоваться от нескольких ABC? Будут ли конфликты метаклассов?**
A: Да, множественное наследование от ABC работает, так как у всех общий метакласс `ABCMeta`.
Конфликт возникает, только если смешать ABC с классом, имеющим иной несовместимый метакласс —
тогда нужен общий метакласс-наследник.

---

## Подводные камни (gotchas)

1. **Неправильный порядок декораторов.** Для абстрактного свойства/classmethod/staticmethod
   `@abstractmethod` должен быть **внутренним** (последним). Перепутанный порядок может «потерять»
   абстрактность, и Python разрешит создать неполный объект.

   ```python
   # ПЛОХО — абстрактность может не сработать корректно
   @abstractmethod
   @property
   def x(self): ...

   # ХОРОШО
   @property
   @abstractmethod
   def x(self): ...
   ```

2. **`@abstractmethod` без `ABCMeta` не работает.** Если класс не использует `ABCMeta`/`ABC`,
   декоратор просто ставит флаг и ни на что не влияет — экземпляр создастся спокойно.

   ```python
   class NotABC:                 # обычный класс, без ABCMeta
       @abstractmethod
       def foo(self): ...
   NotABC()  # OK, никакой ошибки — абстрактность игнорируется
   ```

3. **`register` не проверяет реализацию.** Зарегистрировать можно даже класс без нужных методов.
   `isinstance` вернёт `True`, а вызов метода упадёт с `AttributeError`. `register` — это
   «обещание», а не гарантия.

4. **Виртуальный подкласс не наследует код.** После `register` объект не получает методы ABC.
   Если рассчитывали на mixin-методы (как у `Sequence`) — их не будет; нужно реальное
   наследование.

5. **Забытый абстрактный метод обнаруживается поздно — при инстанцировании.** Класс определится
   без ошибок; `TypeError` прилетит только при `Cls()`. Если объект создаётся в редкой ветке —
   ошибку можно долго не замечать.

6. **`__subclasshook__` может «сломать» явное наследование.** Если хук вернёт `False`,
   `issubclass` даст `False` даже для реального наследника. Поэтому возвращайте `NotImplemented`,
   когда не уверены, а не `False`.

7. **`@runtime_checkable` Protocol проверяет только наличие имён.** `isinstance(x, MyProtocol)`
   не проверяет сигнатуры методов и типы атрибутов — только их присутствие. Это слабее, чем
   полноценная проверка.

8. **Конфликт метаклассов при смешивании.** `class C(ABC, EnumLikeBase)` упадёт с
   `metaclass conflict`, если у `EnumLikeBase` свой метакласс. Решение — общий метакласс,
   наследующий оба.

9. **Абстрактный класс можно случайно «расконсервировать».** Если подкласс реализовал все
   абстрактные методы, но добавил новый `@abstractmethod`, он снова становится абстрактным.
   Это легко упустить.

---

## Лучшие практики

- Наследуйтесь от **`ABC`**, а не указывайте `metaclass=ABCMeta` вручную (читабельнее);
  метакласс — только когда уже есть свой.
- Для абстрактных свойств/classmethod/staticmethod используйте **комбинацию декораторов**, а
  не устаревшие `abstractproperty` / `abstractclassmethod` / `abstractstaticmethod`. Помните:
  `@abstractmethod` — самый внутренний.
- В теле абстрактных методов оставляйте `...` или `raise NotImplementedError`, либо полезную
  общую логику, вызываемую через `super()`.
- Используйте **готовые ABC из `collections.abc`** вместо изобретения своих, когда речь о
  контейнерах/итераторах/отображениях — получите бесплатные mixin-методы и совместимость с
  `isinstance`.
- `register` применяйте осознанно — для сторонних классов, которые нельзя менять; помните, что
  он не валидирует реализацию.
- В `__subclasshook__` возвращайте `NotImplemented` в неопределённых случаях, чтобы не ломать
  стандартную логику и явное наследование.
- Для **структурной** типизации (без наследования), особенно ради статической проверки mypy,
  предпочитайте `typing.Protocol`. ABC — когда нужна номинальная иерархия, общий код (mixin) и
  рантайм-контракт.
- Не злоупотребляйте абстракциями: в простом «питоничном» коде часто достаточно утиной
  типизации. ABC оправданы для библиотек, фреймворков, плагинных архитектур и крупных проектов
  с чёткими контрактами.

---

## Шпаргалка

```python
from abc import ABC, ABCMeta, abstractmethod

# 1) Объявить абстрактный класс
class Base(ABC):                       # или metaclass=ABCMeta
    @abstractmethod
    def do(self): ...                  # обязателен к реализации

# 2) Абстрактное свойство (порядок: property -> abstractmethod)
class C(ABC):
    @property
    @abstractmethod
    def x(self): ...

# 3) Абстрактные classmethod / staticmethod
class D(ABC):
    @classmethod
    @abstractmethod
    def cm(cls): ...
    @staticmethod
    @abstractmethod
    def sm(): ...

# 4) Виртуальный подкласс (без наследования, без проверки)
Base.register(SomeClass)               # или @Base.register над классом

# 5) Структурная проверка подкласса
class E(ABC):
    @classmethod
    def __subclasshook__(cls, C):
        if any("meth" in B.__dict__ for B in C.__mro__):
            return True
        return NotImplemented          # -> стандартная логика
```

Памятка-таблица:

| Что | Как |
|-----|-----|
| Создать ABC | `class X(ABC):` + `@abstractmethod` |
| Абстрактное свойство | `@property` над `@abstractmethod` |
| Абстрактный classmethod | `@classmethod` над `@abstractmethod` |
| Абстрактный staticmethod | `@staticmethod` над `@abstractmethod` |
| Виртуальный подкласс | `X.register(Cls)` |
| Структурная проверка | переопределить `__subclasshook__` |
| Список нерреализованных | `X.__abstractmethods__` (frozenset) |
| Ошибка при создании | `TypeError: Can't instantiate abstract class` |
| Готовые ABC | `collections.abc` (Iterable, Sequence, Mapping, ...) |
| Структурная типизация для mypy | `typing.Protocol` вместо ABC |

Ключевые правила-«мантры»:

- `@abstractmethod` работает **только** с `ABCMeta`/`ABC`.
- `@abstractmethod` — всегда **самый внутренний** декоратор.
- `register` = обещание, **не** проверка и **не** наследование кода.
- Ошибка абстрактности — при **инстанцировании**, не при объявлении.
- ABC — номинально (наследование/register, рантайм); `Protocol` — структурно (mypy).
```
