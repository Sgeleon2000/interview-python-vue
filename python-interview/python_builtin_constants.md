# Встроенные константы (Built-in Constants) — подготовка к собеседованию

## Что это и зачем

Встроенные константы (built-in constants) — это объекты, которые всегда доступны в любом месте программы на Python без необходимости их импортировать. Они находятся в пространстве имён `builtins` и являются частью «ядра» языка. К ним относятся: `True`, `False`, `None`, `NotImplemented`, `Ellipsis` (`...`) и `__debug__`.

Эти объекты — не обычные переменные, а **синглтоны** (за исключением `True`/`False`, которые тоже фактически синглтоны соответствующих значений `int`). Они играют ключевую роль в:

- логике и булевых выражениях (`True`, `False`);
- представлении «отсутствия значения» (`None`);
- механизме перегрузки операторов и протоколе сравнения (`NotImplemented`);
- системе аннотаций типов, срезах numpy и заглушках (`Ellipsis`);
- управлении режимом оптимизации интерпретатора и работе `assert` (`__debug__`).

Понимание семантики этих констант — частый вопрос на собеседованиях уровня middle/senior, потому что они напрямую связаны с тем, как Python реализует диспетчеризацию операторов, идентичность объектов (`is` vs `==`), оптимизацию байткода и контракты протоколов данных.

Отдельно стоит группа «констант», добавляемых модулем `site` для удобства интерактивной работы: `quit`, `exit`, `copyright`, `credits`, `license`. Они не являются настоящими языковыми константами и не должны использоваться в production-коде.

## Ключевые концепции

- **Синглтоны.** `None`, `True`, `False`, `NotImplemented`, `Ellipsis` существуют в единственном экземпляре на весь процесс. Сравнивать их корректно через оператор идентичности `is`, а не через `==`.
- **Неизменяемость и защищённость от переприсваивания.** Начиная с Python 3, `True`, `False`, `None`, `__debug__` являются ключевыми словами (keywords): попытка присвоить им значение — синтаксическая ошибка (`SyntaxError`). `NotImplemented` и `Ellipsis` — обычные имена, но переприсваивать их крайне нежелательно.
- **`None` как «ничего».** Тип `NoneType`. Используется как значение по умолчанию, сигнал «значения нет», результат функций без `return`.
- **`NotImplemented` как сигнал диспетчеру операторов.** Это **значение** (а не исключение), которое метод вроде `__eq__`, `__lt__`, `__add__` возвращает, чтобы сказать интерпретатору: «я не умею работать с этим типом, попробуй reflected-метод другого операнда».
- **`Ellipsis` (`...`) как универсальная заглушка/маркер.** Применяется в `typing` (`Callable[..., T]`, `Tuple[int, ...]`), в numpy для срезов (`arr[..., 0]`), как «тело-заглушка» в `.pyi`-стабах и протоколах.
- **`__debug__` и флаг `-O`.** По умолчанию `__debug__ == True`. При запуске интерпретатора с `-O` (или `-OO`) `__debug__` становится `False`, а все инструкции `assert` удаляются из байткода. Это константа времени компиляции.
- **Истинность булевых значений — это `int`.** `True == 1` и `False == 0`, `isinstance(True, int)` истинно. `bool` — подкласс `int`.

## Основные функции/классы/методы

### `None`

`None` — единственный объект типа `NoneType`. Обозначает отсутствие значения.

```python
print(type(None))          # => <class 'NoneType'>
print(None is None)        # => True

# Функция без явного return возвращает None
def f():
    pass

print(f())                 # => None

# Типичное использование: значение по умолчанию для изменяемых аргументов
def append_item(item, target=None):
    if target is None:     # правильно: проверка через is
        target = []        # создаём новый список на каждый вызов
    target.append(item)
    return target

print(append_item(1))      # => [1]
print(append_item(2))      # => [2]  (а не [1, 2]!)
```

`None` всегда «ложен» в булевом контексте:

```python
print(bool(None))          # => False
if not None:
    print("None ложен")    # => None ложен
```

`NoneType` нельзя инстанцировать заново — вы всегда получаете тот же объект:

```python
NoneType = type(None)
print(NoneType() is None)  # => True
```

### `True` и `False`

Это синглтоны типа `bool`, который наследуется от `int`.

```python
print(isinstance(True, int))   # => True
print(True + True)             # => 2
print(True == 1)               # => True
print(False == 0)              # => True
print(True is 1)               # => False (разные объекты, хоть и равны по ==)

# bool() приводит к True/False по «истинности» объекта
print(bool([]))                # => False (пустой контейнер)
print(bool([0]))               # => True  (непустой)
print(bool(0))                 # => False
print(bool("0"))               # => True  (непустая строка)
```

С Python 3 `True`/`False` — ключевые слова:

```python
# True = 5   # => SyntaxError: cannot assign to True
```

### `NotImplemented`

`NotImplemented` — специальное **значение**, возвращаемое из методов бинарных операторов и сравнения, когда операция не поддерживается для данной пары типов. Это НЕ исключение `NotImplementedError`.

Механизм работы на примере `__eq__`:

```python
class Money:
    def __init__(self, amount, currency):
        self.amount = amount
        self.currency = currency

    def __eq__(self, other):
        if not isinstance(other, Money):
            # Не знаем, как сравнивать с другим типом —
            # возвращаем NotImplemented, а не False!
            return NotImplemented
        return (self.amount, self.currency) == (other.amount, other.currency)

    def __repr__(self):
        return f"Money({self.amount}, {self.currency!r})"


a = Money(100, "USD")
b = Money(100, "USD")
print(a == b)              # => True
print(a == "что-то")       # => False (Python применил fallback на идентичность)
```

Почему важно возвращать `NotImplemented`, а не `False`? Потому что интерпретатор, получив `NotImplemented` от `a.__eq__(other)`, попробует «отражённый» (reflected) метод `other.__eq__(a)`. Если оба вернут `NotImplemented`, для `==`/`!=` Python применит сравнение по идентичности (`is`). Если вернуть `False` напрямую — вы сломаете эту цепочку и помешаете другому операнду корректно ответить.

То же самое для арифметики (`__add__` / `__radd__`):

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __add__(self, other):
        if isinstance(other, Vector):
            return Vector(self.x + other.x, self.y + other.y)
        return NotImplemented        # не умеем складывать с этим типом

    def __radd__(self, other):
        # вызовется, если other.__add__(self) вернул NotImplemented
        return self.__add__(other)

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"


v1 = Vector(1, 2)
v2 = Vector(3, 4)
print(v1 + v2)             # => Vector(4, 6)

try:
    print(v1 + 5)          # ни __add__, ни __radd__ не справились
except TypeError as e:
    print("TypeError:", e) # => TypeError: unsupported operand type(s)...
```

Если **оба** операнда вернут `NotImplemented` для арифметического оператора, интерпретатор сам поднимет `TypeError` с понятным сообщением — это удобно и единообразно.

Пример с упорядочиванием (`__lt__`):

```python
import functools

@functools.total_ordering
class Version:
    def __init__(self, parts):
        self.parts = tuple(parts)

    def __eq__(self, other):
        if not isinstance(other, Version):
            return NotImplemented
        return self.parts == other.parts

    def __lt__(self, other):
        if not isinstance(other, Version):
            return NotImplemented   # позволит сравнить «зеркально»
        return self.parts < other.parts


print(Version([1, 2]) < Version([1, 3]))   # => True
print(Version([2, 0]) > Version([1, 9]))   # => True (total_ordering выводит из __lt__/__eq__)
```

Важно: с Python 3.9 проверка истинности `bool(NotImplemented)` выдаёт `DeprecationWarning`, а в будущих версиях станет ошибкой. Никогда не используйте `NotImplemented` в булевом контексте.

```python
import warnings
warnings.simplefilter("always")
print(bool(NotImplemented))   # => True, но с DeprecationWarning
```

### `Ellipsis` (`...`)

`Ellipsis` — единственный объект типа `ellipsis`. Литерал `...` эквивалентен имени `Ellipsis`.

```python
print(... is Ellipsis)        # => True
print(type(...))              # => <class 'ellipsis'>
```

Сам по себе `Ellipsis` не несёт встроенной семантики — это «маркер», которому смысл придают библиотеки и соглашения.

**1. В аннотациях типов (`typing`):**

```python
from typing import Callable, Tuple

# Callable с произвольной сигнатурой аргументов, возвращающий int
Handler = Callable[..., int]

# Кортеж переменной длины из однотипных элементов
IntsTuple = Tuple[int, ...]   # (1,), (1, 2, 3), () — все валидны

def apply(fn: Handler, *args) -> int:
    return fn(*args)
```

**2. Как тело-заглушка (часто в `.pyi`-стабах, протоколах, абстракциях):**

```python
from typing import Protocol

class Comparable(Protocol):
    def __lt__(self, other) -> bool: ...   # тело — просто ...

def todo() -> None:
    ...   # заглушка вместо pass; читается как «здесь будет реализация»
```

**3. В numpy для срезов (одно из самых частых практических применений):**

```python
# Псевдокод numpy (требует установленного numpy)
# import numpy as np
# arr = np.zeros((2, 3, 4, 5))
# arr[..., 0]      # эквивалент arr[:, :, :, 0] — Ellipsis заменяет «все промежуточные оси»
# arr[0, ...]      # эквивалент arr[0, :, :, :]
# arr[..., 0, :]   # Ellipsis раскрывается до нужного числа осей
```

`Ellipsis` в индексации numpy означает «столько полных срезов `:`, сколько нужно, чтобы заполнить недостающие измерения». Это позволяет писать обобщённый код, не зная заранее размерность массива.

**4. Как «значение-сентинел» (реже, но встречается):**

```python
_MISSING = ...   # маркер «значение не передано», альтернатива созданию object()

def get(d, key, default=_MISSING):
    if key in d:
        return d[key]
    if default is _MISSING:
        raise KeyError(key)
    return default
```

### `__debug__`

`__debug__` — булева константа, вычисляемая на этапе компиляции. По умолчанию `True`. При запуске Python с флагом `-O` она равна `False`.

```python
print(__debug__)          # => True  (если запущено без -O)

# Инструкция assert эквивалентна:
#   if __debug__:
#       if not <условие>:
#           raise AssertionError(<сообщение>)

def divide(a, b):
    assert b != 0, "Делитель не должен быть нулём"
    return a / b
```

При запуске `python -O script.py`:
- `__debug__` становится `False`;
- все инструкции `assert` физически удаляются из байткода (не просто пропускаются — их там нет);
- удаляются также блоки `if __debug__:`.

```python
# Этот блок исчезнет из байткода при -O
if __debug__:
    print("Дорогая отладочная проверка инвариантов")
```

`-OO` дополнительно к `-O` удаляет docstring-и, уменьшая размер `.pyc`.

`__debug__` нельзя присвоить — это ключевое слово:

```python
# __debug__ = False   # => SyntaxError: cannot assign to __debug__
```

Проверить режим программно можно и так:

```python
import sys
print(sys.flags.optimize)   # 0 без -O, 1 при -O, 2 при -OO
```

### Константы из `site` (кратко)

Модуль `site`, который запускается автоматически при старте интерпретатора, добавляет в `builtins` несколько удобных для интерактивной работы объектов. **Это не настоящие константы языка** и их не должно быть в скриптах.

```python
# Доступны в интерактивной сессии:
# quit       — вызов quit() завершает интерпретатор
# exit       — вызов exit() завершает интерпретатор (то же, что quit)
# copyright  — печатает информацию об авторских правах
# credits    — печатает благодарности участникам
# license    — запускает интерактивный просмотр лицензии

print(type(exit))   # => <class '_sitebuiltins.Quitter'>
```

Важные нюансы:
- `exit`/`quit` — это объекты класса `Quitter`, а не функции из `sys`. Для завершения программы в скриптах нужно использовать `sys.exit()`, а не `exit()`/`quit()`.
- Если интерпретатор запущен с флагом `-S` (без `site`), этих имён не будет вообще — поэтому код, полагающийся на `exit()`, может упасть с `NameError`.
- `copyright`, `credits`, `license` — экземпляры `_Printer`, печатающие текст при вызове или при отображении в REPL.

## Частые вопросы на собеседовании

**Q: Чем `None` отличается от `False`, `0` и пустой строки?**
A: `None` — это отдельный объект типа `NoneType`, означающий «значения нет». `False`, `0`, `""` — конкретные значения соответствующих типов. Все они «ложны» в булевом контексте (`bool(x) == False`), но это разные объекты: `None == False` → `False`, `None == 0` → `False`. Поэтому проверять «значение не задано» нужно через `is None`, а не через `if not x`, иначе вы спутаете отсутствие значения с валидными «ложными» значениями (0, пустой список и т. д.).

**Q: Почему `None` сравнивают через `is`, а не через `==`?**
A: `None` — синглтон, поэтому проверка идентичности `x is None` всегда корректна, быстрее и не зависит от пользовательского `__eq__`. Если объект переопределил `__eq__` некорректно (например, `__eq__` всегда возвращает `True`), то `x == None` может дать неверный результат, тогда как `x is None` надёжен. PEP 8 прямо предписывает `is None`/`is not None`.

**Q: В чём разница между `NotImplemented` и `NotImplementedError`?**
A: `NotImplemented` — это **значение-синглтон**, которое возвращают из методов операторов/сравнения, чтобы сообщить интерпретатору «попробуй другой операнд». `NotImplementedError` — это **исключение**, которое поднимают (`raise`) в абстрактных методах, означая «этот метод должен быть переопределён в подклассе». Их путают, но семантика противоположная: первое — мягкий сигнал диспетчеру, второе — жёсткая ошибка.

**Q: Что произойдёт, если `__eq__` вернёт `False` вместо `NotImplemented` для неизвестного типа?**
A: Вы сломаете цепочку диспетчеризации. Получив `False`, Python считает, что объект «знает» ответ, и не попробует reflected-метод другого операнда. В результате сравнение `a == b` может дать `False` там, где `b` умел бы корректно сравниться с `a`. Это особенно опасно при сравнении объектов разных, но совместимых типов.

**Q: Как `__debug__` связан с `assert` и флагом `-O`?**
A: `assert` компилируется в код вида `if __debug__: if not cond: raise AssertionError`. По умолчанию `__debug__ == True`. При запуске с `-O` интерпретатор задаёт `__debug__ = False` на этапе компиляции и **полностью удаляет** все `assert` и блоки `if __debug__:` из байткода. Поэтому `assert` нельзя использовать для проверок, критичных для безопасности или бизнес-логики — в production с `-O` они исчезнут.

**Q: Что такое `Ellipsis` и где он реально применяется?**
A: `Ellipsis` (литерал `...`) — синглтон-маркер без встроенной семантики. Применяется: в `typing` — `Callable[..., R]` (любые аргументы), `Tuple[int, ...]` (кортеж переменной длины); в numpy — для срезов `arr[..., 0]` (заменяет нужное число полных осей); как тело-заглушка в `.pyi`-файлах и протоколах вместо `pass`; иногда как уникальный сентинел.

**Q: `True` — это `int`? Докажите.**
A: Да, `bool` — подкласс `int`. `isinstance(True, int)` → `True`, `True == 1` → `True`, `True + True` → `2`, `sum([True, True, False])` → `2`. Это позволяет, например, считать количество истинных условий суммированием булевых значений. Но `True is 1` → `False`: они равны по значению, но это разные объекты.

**Q: Можно ли переприсвоить `None`, `True`, `False`?**
A: В Python 3 — нет, это ключевые слова, попытка даёт `SyntaxError`. В Python 2 `True`/`False` были обычными именами и их можно было переопределить (источник классических багов). `None` был защищён уже в Python 2. `NotImplemented` и `Ellipsis` — не ключевые слова, технически переприсваиваемы, но делать это нельзя.

**Q: Почему `bool(NotImplemented)` выдаёт предупреждение?**
A: С Python 3.9 использование `NotImplemented` в булевом контексте признано ошибочным паттерном (обычно это баг — когда `NotImplemented` случайно возвращён туда, где ждали `bool`). Сейчас выдаётся `DeprecationWarning`, в будущем будет `TypeError`. Если ваш `__bool__`/условие наткнулось на `NotImplemented` — это сигнал ошибки в логике.

**Q: В чём разница между `exit()`/`quit()` и `sys.exit()`?**
A: `exit` и `quit` добавляются модулем `site` (объекты `Quitter`) и предназначены только для интерактивной сессии. В скриптах их может не быть (при запуске с `-S`), поэтому полагаться на них нельзя. Для завершения программы в коде используйте `sys.exit()` (или `raise SystemExit`), который всегда доступен и принимает код выхода.

**Q: Сколько существует объектов `None` в одном процессе?**
A: Ровно один. `None` — синглтон, гарантированно единственный экземпляр `NoneType` на интерпретатор. Поэтому `a is None` — надёжная проверка, а `id(None)` стабилен в пределах процесса. То же относится к `True`, `False`, `NotImplemented`, `Ellipsis`.

**Q: Можно ли использовать `Ellipsis` как ключ словаря или элемент множества?**
A: Да, `Ellipsis` хешируемый и неизменяемый, поэтому годится как ключ/элемент. Иногда его используют как уникальный сентинел-ключ, хотя для надёжности лучше создавать собственный `object()`, чтобы случайно не пересечься с чужим использованием `...`.

## Подводные камни (gotchas)

**1. Сравнение с `None` через `==` вместо `is`.**

```python
class Weird:
    def __eq__(self, other):
        return True        # «равно всему»

w = Weird()
print(w == None)           # => True  (ловушка!)
print(w is None)           # => False (правильно)
```

Всегда `x is None`, никогда `x == None`.

**2. `if not x:` вместо `if x is None:`.**

```python
def process(items=None):
    # Ошибка: пустой список — валидное значение, но not [] == True
    if not items:          # сработает и для None, и для []
        items = ["default"]
    return items

print(process([]))         # => ['default']  (неожиданно затёрли пустой список!)

# Правильно:
def process2(items=None):
    if items is None:
        items = ["default"]
    return items

print(process2([]))        # => []  (корректно)
```

**3. Возврат `NotImplemented` vs `raise NotImplementedError`.**

```python
# Неверно: в операторе нельзя поднимать NotImplementedError для «не мой тип»
class Bad:
    def __add__(self, other):
        raise NotImplementedError   # сломает fallback на __radd__!

# Неверно: возвращать False из __eq__ для чужого типа
class AlsoBad:
    def __eq__(self, other):
        return False                # мешает reflected-сравнению

# Верно:
class Good:
    def __eq__(self, other):
        if not isinstance(other, Good):
            return NotImplemented
        return True
```

**4. Опора на `assert` для критичных проверок.**

```python
def withdraw(balance, amount):
    assert amount <= balance, "Недостаточно средств"   # ИСЧЕЗНЕТ при python -O!
    return balance - amount

# С флагом -O проверка пропадёт, и баг с овердрафтом пройдёт незаметно.
# Для бизнес-проверок используйте явный if + raise:
def withdraw_safe(balance, amount):
    if amount > balance:
        raise ValueError("Недостаточно средств")
    return balance - amount
```

**5. `bool` как `int` — неожиданная арифметика.**

```python
data = {True: "да", 1: "один"}
print(data)                # => {True: 'один'}  (True и 1 — один ключ!)
print(data[True])          # => один

print(["a", "b"][False])   # => 'a'  (False == 0 как индекс)
```

**6. Изменяемый дефолт через `None`-паттерн обязателен.**

```python
# Классическая ошибка: дефолт вычисляется один раз при определении функции
def bad(item, acc=[]):
    acc.append(item)
    return acc

print(bad(1))              # => [1]
print(bad(2))              # => [1, 2]  (тот же список!)

# Решение — None-сентинел:
def good(item, acc=None):
    if acc is None:
        acc = []
    acc.append(item)
    return acc
```

**7. `Ellipsis` как сентинел может пересечься с typing/numpy.**
Если вы используете `...` как «значение не передано», а потом тот же объект придёт из аннотации или numpy-кода, можно получить ложное срабатывание. Для надёжного сентинела безопаснее `_MISSING = object()`.

**8. `NotImplemented` в логическом выражении — скрытый баг.**

```python
result = some_obj.__eq__(other)   # может вернуть NotImplemented
if result:                        # DeprecationWarning, если result is NotImplemented
    ...
# Не вызывайте dunder-методы напрямую; используйте оператор ==, который
# корректно обрабатывает NotImplemented.
```

## Лучшие практики

- Для проверки на «отсутствие значения» используйте `is None` / `is not None`, а не `==` и не «истинность» объекта.
- В методах операторов и сравнения (`__eq__`, `__lt__`, `__add__`, …) возвращайте `NotImplemented` (значение) для неподдерживаемых типов, чтобы работал fallback на reflected-методы. Не возвращайте `False` и не поднимайте `NotImplementedError`.
- Для «метод должен быть переопределён» поднимайте `NotImplementedError` (исключение), обычно вместе с `abc.abstractmethod`.
- Не используйте `assert` для проверок, критичных в production: они исчезают при `-O`. `assert` — для отладочных инвариантов и тестов.
- Не вызывайте dunder-методы сравнения напрямую (`a.__eq__(b)`) — используйте операторы (`a == b`), чтобы интерпретатор сам корректно обработал `NotImplemented`.
- Не полагайтесь на `exit()`/`quit()` в скриптах — используйте `sys.exit()`. `copyright`/`credits`/`license` — только для интерактива.
- Помните, что `bool` — это `int`: будьте осторожны с булевыми значениями в роли ключей словаря и индексов.
- Для уникальных сентинел-маркеров предпочитайте `object()` вместо `...` или `None`, если `None` — валидное значение домена.
- Никогда не используйте `NotImplemented` в булевом контексте — это признак ошибки в логике.

## Шпаргалка

| Константа | Тип | Назначение | Проверка |
|---|---|---|---|
| `None` | `NoneType` | отсутствие значения, дефолты, «нет return» | `x is None` |
| `True` / `False` | `bool` (подкласс `int`) | логика; `True==1`, `False==0` | `x is True` редко нужно |
| `NotImplemented` | `NotImplementedType` | сигнал «операция не поддержана» из dunder-методов → fallback | `return NotImplemented` |
| `Ellipsis` / `...` | `ellipsis` | маркер: typing, numpy-срезы, заглушки | `x is Ellipsis` |
| `__debug__` | `bool` | `True` без `-O`, `False` с `-O`; управляет `assert` | константа компиляции |

- `None`, `True`, `False`, `__debug__` — **ключевые слова**, переприсвоить нельзя (`SyntaxError`).
- `NotImplemented` ≠ `NotImplementedError`: первое — значение для операторов, второе — исключение для абстрактных методов.
- Возврат из `__eq__`/`__add__` для чужого типа: **`return NotImplemented`** (не `False`, не `raise`).
- Если оба операнда вернули `NotImplemented`: для `==`/`!=` → сравнение по `is`; для арифметики → `TypeError`.
- `assert` и блоки `if __debug__:` **удаляются** из байткода при `python -O`. Не используйте `assert` для критичной логики.
- `bool` — подкласс `int`: `isinstance(True, int)` → `True`; `{True: ..., 1: ...}` — один ключ.
- `...` в numpy: `arr[..., i]` = «все промежуточные оси целиком, затем индекс i».
- Все эти объекты — **синглтоны**: сравнивайте через `is`.
- `exit()`/`quit()` — из модуля `site`, только для REPL; в коде — `sys.exit()`.
- Проверка режима оптимизации программно: `sys.flags.optimize` (0 / 1 / 2).
