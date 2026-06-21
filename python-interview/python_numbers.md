# numbers — подготовка к собеседованию

## Что это и зачем

`numbers` — модуль стандартной библиотеки, определяющий **абстрактную числовую башню** (numeric tower) через абстрактные базовые классы (ABC). Он задаёт иерархию числовых типов согласно PEP 3141 и описывает, что значит «быть числом» того или иного вида.

Зачем нужен:
- **Полиморфные проверки типов.** `isinstance(x, numbers.Number)` истинно для `int`, `float`, `complex`, `Decimal`, `Fraction` и совместимых пользовательских типов — это лучше, чем перечислять конкретные классы.
- **Контракты для своих числовых классов.** Наследуясь от `numbers.Real`/`Rational`, вы декларируете, какие операции должен поддерживать ваш тип.
- **Документирование уровня абстракции функции** (принимает любое `Real`? любое `Integral`?).

Это в первую очередь инструмент проектирования и проверки типов, а не вычислений.

## Ключевые концепции

Башня (от самого общего к самому конкретному):

```
Number          — вообще число (вершина башни)
  └─ Complex     — комплексные: + - * / , .real, .imag, abs(), conjugate()
       └─ Real   — вещественные: упорядочены (< >), float(), round(), //, %
            └─ Rational — рациональные: .numerator, .denominator
                 └─ Integral — целые: <<, >>, &, |, ^, ~, int()
```

- Чем ниже по башне, тем больше гарантированных операций.
- Каждый уровень — `abc.ABCMeta`-класс; встроенные типы **зарегистрированы** в нём (не наследуются напрямую).
- `bool` ⊂ `int` ⊂ `Integral`; `Fraction` — `Rational`; `float` — `Real`; `complex` — `Complex`; `Decimal` — особый случай (см. gotchas).

## Основные функции/классы/методы

### Проверка принадлежности башне

```python
import numbers
from fractions import Fraction
from decimal import Decimal

# Number — вершина: всё, что число
print(isinstance(5, numbers.Number))            # True
print(isinstance(3.14, numbers.Number))         # True
print(isinstance(2 + 3j, numbers.Number))       # True
print(isinstance(Fraction(1, 2), numbers.Number))  # True
print(isinstance(Decimal('1.5'), numbers.Number))  # True
print(isinstance('5', numbers.Number))          # False (строка не число)
```

### Уровни башни

```python
import numbers
from fractions import Fraction

x = 5
print(isinstance(x, numbers.Integral))   # True  — целое
print(isinstance(x, numbers.Rational))   # True  — целое тоже рационально
print(isinstance(x, numbers.Real))       # True  — и вещественно
print(isinstance(x, numbers.Complex))    # True  — и комплексно (мнимая часть 0)

f = 3.14
print(isinstance(f, numbers.Integral))   # False — float не целое
print(isinstance(f, numbers.Rational))   # False — float НЕ Rational (см. gotcha)
print(isinstance(f, numbers.Real))       # True

c = 2 + 3j
print(isinstance(c, numbers.Real))       # False — комплексное не вещественно
print(isinstance(c, numbers.Complex))    # True

fr = Fraction(1, 3)
print(isinstance(fr, numbers.Rational))  # True  — у Fraction есть num/denom
print(isinstance(fr, numbers.Integral))  # False
```

### bool — это Integral

```python
import numbers

print(isinstance(True, numbers.Integral))  # True — bool наследует int!
print(isinstance(True, int))               # True
print(True + True)                         # 2

# Поэтому проверка "это целое, но не bool" иногда нужна явно:
def is_strict_int(x):
    return isinstance(x, numbers.Integral) and not isinstance(x, bool)

print(is_strict_int(5))     # True
print(is_strict_int(True))  # False
```

### Какие операции гарантирует каждый уровень

```python
import numbers

# Complex гарантирует: +, -, *, /, ==, abs(), .real, .imag, conjugate()
c = 3 + 4j
print(c.real, c.imag)     # 3.0 4.0
print(abs(c))             # 5.0
print(c.conjugate())      # (3-4j)

# Real добавляет: упорядочивание (<, >), //, %, round(), float(), math.floor/ceil
r = 7.5
print(r // 2, r % 2)      # 3.0 1.5
print(round(r))           # 8

# Rational добавляет: .numerator, .denominator
from fractions import Fraction
fr = Fraction(6, 8)
print(fr.numerator, fr.denominator)  # 3 4

# Integral добавляет: побитовые <<, >>, &, |, ^, ~, int()
i = 12
print(i << 2, i & 5, ~i)  # 48 4 -13
```

### Создание собственного числового типа

```python
import numbers

# Регистрация существующего класса как члена башни (без наследования)
class MyMoney:
    def __init__(self, cents):
        self.cents = cents

# Можно зарегистрировать как Real (виртуальное наследование)
numbers.Real.register(MyMoney)
print(isinstance(MyMoney(100), numbers.Real))    # True
print(issubclass(MyMoney, numbers.Real))         # True
# Но register НЕ заставляет реализовать методы — это лишь обещание!

# Полноценное наследование требует реализовать абстрактные методы:
class Mod7(numbers.Integral):
    def __init__(self, value):
        self.value = value % 7
    def __int__(self): return self.value
    # ... нужно реализовать __add__, __mul__, __abs__, __lshift__,
    #     numerator, denominator и десятки других — иначе TypeError
```

## Частые вопросы на собеседовании

**Q1: Что такое числовая башня (numeric tower)?**
A: Иерархия абстрактных базовых классов из модуля `numbers`, описанная в PEP 3141: `Number → Complex → Real → Rational → Integral`. Каждый нижний уровень — подмножество верхнего с дополнительными гарантированными операциями. Позволяет писать код, полиморфный по «уровню числовости».

**Q2: Как проверить, что объект — вообще число?**
A: `isinstance(x, numbers.Number)`. Это покрывает `int`, `float`, `complex`, `Fraction`, `Decimal` и пользовательские числовые типы, в отличие от перечисления `isinstance(x, (int, float))`.

**Q3: Является ли `float` `Rational`?**
A: Нет. `isinstance(3.14, numbers.Rational)` — `False`. Хотя любой конкретный `float` технически рационален (двоичная дробь), Python намеренно не регистрирует `float` как `Rational`, потому что у него нет точных `.numerator`/`.denominator` в общем контракте. `float` — это `Real`.

**Q4: `bool` — это число? На каком уровне?**
A: Да, `bool` — подкласс `int`, значит `Integral` (и всё выше). `isinstance(True, numbers.Integral)` — `True`, `True + 1 == 2`. Поэтому если нужно отличать настоящие целые от булевых, проверяйте `and not isinstance(x, bool)`.

**Q5: Где находится `Decimal` в башне?**
A: `Decimal` зарегистрирован только как `numbers.Number`, но **НЕ** как `Real`/`Rational`/`Integral`. `isinstance(Decimal('1'), numbers.Real)` — `False`. Это сделано намеренно, так как поведение Decimal (контекст, округление) не вписывается в контракт `Real`.

**Q6: В чём разница между `register` и наследованием от ABC?**
A: `numbers.Real.register(MyClass)` — «виртуальное наследование»: `isinstance`/`issubclass` начинают возвращать `True`, но Python **не проверяет**, реализованы ли методы (это лишь обещание). Прямое наследование `class C(numbers.Real)` требует реализовать все абстрактные методы, иначе `C()` бросит `TypeError`.

**Q7: Какие операции гарантирует `Integral`, но не `Real`?**
A: Побитовые операции (`<<`, `>>`, `&`, `|`, `^`, `~`) и преобразование `int()` без потерь. `Real` гарантирует только арифметику, сравнение, `//`, `%`, `round()`, `float()`.

**Q8: Зачем нужен модуль `numbers` на практике?**
A: Для написания обобщённых числовых библиотек и валидации: функция может объявить «принимаю любое `numbers.Real`» и работать с `int`, `float`, `Fraction` единообразно. Также для дизайна собственных числовых типов с явными контрактами.

**Q9: Чем `numbers.Integral` отличается от `int`?**
A: `int` — конкретный тип. `numbers.Integral` — абстракция «любое целое»: покрывает `int`, `bool` и пользовательские целочисленные типы. Проверка `isinstance(x, numbers.Integral)` шире и гибче, чем `isinstance(x, int)` (хотя для встроенных совпадает, кроме сторонних реализаций).

**Q10: Можно ли создать экземпляр `numbers.Real` напрямую?**
A: Нет, это абстрактный класс — `numbers.Real()` бросит `TypeError`. Его можно только наследовать (реализовав абстрактные методы) или регистрировать в нём свои классы.

## Подводные камни (gotchas)

- **`Decimal` НЕ является `Real`.** `isinstance(Decimal('1'), numbers.Real)` → `False`. Только `Number`. Частый сюрприз — проверка `isinstance(x, numbers.Real)` отвергнет Decimal.

- **`float` НЕ является `Rational`.** Несмотря на то, что каждый float — двоичная дробь. Если ждёте `Rational`, float не пройдёт.

- **`bool` проходит как `Integral`.** `isinstance(True, numbers.Integral)` → `True`. Может привести к багам, когда `True` ведёт себя как `1` в числовых контекстах. Фильтруйте явно.

- **`register` не проверяет реализацию (точность контракта).** После `numbers.Real.register(C)` проверки типов проходят, но методов может не быть — при вызове получите `AttributeError` в неожиданном месте. Регистрация — это обещание, не гарантия.

- **Наследование от ABC требует много методов.** `class C(numbers.Integral)` без реализации всех абстрактных методов (`__add__`, `__mul__`, `__lshift__`, `numerator`, `denominator` и т.д.) даст `TypeError` при создании экземпляра.

- **Точность float сохраняется.** `numbers` — про типы, не про значения; он не спасает от ошибок IEEE 754. `isinstance(0.1+0.2, numbers.Real)` истинно, но `0.1+0.2 != 0.3` остаётся в силе.

- **Производительность.** `isinstance` с ABC чуть медленнее, чем с конкретными классами (есть `__instancecheck__`). В горячих циклах это заметно.

- **NumPy-числа.** Скаляры NumPy (`np.int64`, `np.float64`) могут быть, а могут не быть зарегистрированы в башне в зависимости от версии — не полагайтесь на `isinstance(np_value, numbers.Integral)` без проверки.

## Лучшие практики

- Для проверки «это число?» используйте `isinstance(x, numbers.Number)`, а не перечисление конкретных типов.
- Объявляйте требуемый уровень абстракции: принимаете дроби — проверяйте `numbers.Rational`; нужны целые — `numbers.Integral`.
- Помните про `bool`: если нужны «настоящие» целые, добавляйте `and not isinstance(x, bool)`.
- Не ждите, что `Decimal` и `float` впишутся в `Real`/`Rational` так, как кажется интуитивно — проверьте реальное поведение.
- Создавая собственный числовой тип, наследуйтесь от подходящего ABC и реализуйте контракт; `register` используйте только когда уверены в совместимости.
- `numbers` — про дизайн и валидацию типов, а для точности значений применяйте `Decimal`/`Fraction`.

## Шпаргалка

```python
import numbers

# Башня (от общего к конкретному):
# Number > Complex > Real > Rational > Integral

# Проверки
isinstance(x, numbers.Number)     # любое число
isinstance(x, numbers.Complex)    # +,-,*,/, abs, .real, .imag, conjugate
isinstance(x, numbers.Real)       # + упорядочивание, //, %, round, float
isinstance(x, numbers.Rational)   # + .numerator, .denominator
isinstance(x, numbers.Integral)   # + побитовые <<,>>,&,|,^,~, int

# Принадлежность встроенных типов:
# int      -> Integral (и всё выше)
# bool     -> Integral (наследует int!)
# Fraction -> Rational
# float    -> Real     (НЕ Rational!)
# complex  -> Complex  (НЕ Real)
# Decimal  -> Number   (НЕ Real/Rational/Integral!)

# "Настоящее целое" без bool:
isinstance(x, numbers.Integral) and not isinstance(x, bool)

# Регистрация своего типа (виртуальное наследование, без проверки методов):
numbers.Real.register(MyClass)

# Наследование (требует реализации всех абстрактных методов):
class C(numbers.Integral): ...
```
