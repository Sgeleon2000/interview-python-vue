# operator — подготовка к собеседованию

## Что это и зачем

`operator` — модуль стандартной библиотеки, который предоставляет встроенные операторы Python (`+`, `*`, `<`, доступ по индексу, доступ к атрибутам и т.д.) в виде **обычных функций**. Это полезно, когда нужно передать операцию как аргумент в функцию высшего порядка (`map`, `filter`, `sorted`, `reduce`, `functools.reduce`).

Зачем это нужно:
- **Производительность**: `operator.add` быстрее `lambda a, b: a + b` (реализация на C, нет накладных расходов на вызов Python-функции).
- **Читаемость**: `sorted(data, key=itemgetter('age'))` яснее, чем `sorted(data, key=lambda x: x['age'])`.
- **Функциональный стиль**: операторы становятся объектами первого класса.

```python
import operator

# Вместо lambda — готовая функция
nums = [3, 1, 2]
total = __import__('functools').reduce(operator.add, nums)
print(total)  # 6

# Доступ к полю как функция
from operator import itemgetter
people = [{'name': 'Аня', 'age': 30}, {'name': 'Боря', 'age': 25}]
print(sorted(people, key=itemgetter('age')))
# [{'name': 'Боря', 'age': 25}, {'name': 'Аня', 'age': 30}]
```

## Ключевые концепции

1. **Функция вместо оператора** — каждый оператор Python имеет функциональный аналог: `a + b` -> `operator.add(a, b)`.
2. **Геттеры** (`itemgetter`, `attrgetter`, `methodcaller`) — фабрики, возвращающие вызываемые объекты для извлечения данных.
3. **Ключевые функции (key functions)** — `sorted`, `min`, `max`, `groupby` принимают `key=`, и геттеры идеально подходят на эту роль.
4. **Производительность и picklability** — операторные функции быстрее и сериализуемы (в отличие от lambda).

## Основные функции/классы/методы

### itemgetter — извлечение по индексу/ключу

**`itemgetter(item)` / `itemgetter(*items)`** — возвращает вызываемый объект, который извлекает элемент(ы) по индексу или ключу (через `obj[item]`).

```python
from operator import itemgetter

# По индексу (для списков/кортежей)
get_second = itemgetter(1)
print(get_second(['a', 'b', 'c']))  # 'b'

# По ключу (для словарей)
get_name = itemgetter('name')
print(get_name({'name': 'Аня', 'age': 30}))  # 'Аня'

# Несколько элементов сразу -> возвращает КОРТЕЖ
get_first_last = itemgetter(0, -1)
print(get_first_last('Python'))  # ('P', 'n')

# Сортировка списка кортежей по второму элементу
data = [('яблоко', 3), ('банан', 1), ('вишня', 2)]
print(sorted(data, key=itemgetter(1)))
# [('банан', 1), ('вишня', 2), ('яблоко', 3)]

# Сортировка по нескольким полям (сначала по 1-му, потом по 2-му)
records = [('Иванов', 30), ('Петров', 25), ('Иванов', 22)]
print(sorted(records, key=itemgetter(0, 1)))
# [('Иванов', 22), ('Иванов', 30), ('Петров', 25)]
```

### attrgetter — извлечение атрибутов

**`attrgetter(attr)` / `attrgetter(*attrs)`** — возвращает вызываемый объект, извлекающий атрибут(ы) (через `obj.attr`). Поддерживает вложенные атрибуты через точку.

```python
from operator import attrgetter

class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age
    def __repr__(self):
        return f'{self.name}({self.age})'

people = [Person('Аня', 30), Person('Боря', 25), Person('Вася', 30)]

# Сортировка по атрибуту age
print(sorted(people, key=attrgetter('age')))
# [Боря(25), Аня(30), Вася(30)]

# Несколько атрибутов -> кортеж. Сортировка по age, затем по name
print(sorted(people, key=attrgetter('age', 'name')))
# [Боря(25), Аня(30), Вася(30)]

# Вложенные атрибуты через точку
class Order:
    def __init__(self, person):
        self.customer = person
orders = [Order(Person('Зоя', 40)), Order(Person('Аня', 20))]
print(sorted(orders, key=attrgetter('customer.name')))  # сортировка по customer.name
```

### methodcaller — вызов метода

**`methodcaller(name, *args, **kwargs)`** — возвращает вызываемый объект, который вызывает у переданного объекта метод `name` с заданными аргументами (через `obj.name(*args, **kwargs)`).

```python
from operator import methodcaller

# Вызвать метод upper() у каждой строки
words = ['python', 'java', 'go']
print(list(map(methodcaller('upper'), words)))  # ['PYTHON', 'JAVA', 'GO']

# Метод с аргументами
texts = ['a-b-c', 'x-y']
print(list(map(methodcaller('split', '-'), texts)))  # [['a','b','c'], ['x','y']]

# Сортировка строк по количеству вхождений 'a'
data = ['banana', 'apple', 'cherry']
print(sorted(data, key=methodcaller('count', 'a')))
# ['cherry', 'apple', 'banana']  (0, 1, 3 вхождения 'a')

# methodcaller('m', x) эквивалентен lambda obj: obj.m(x)
```

### Операторные функции: арифметика, сравнение, логика

`operator` содержит функции для большинства операторов Python.

```python
import operator
from functools import reduce

# --- Арифметические ---
print(operator.add(3, 4))       # 7    (3 + 4)
print(operator.sub(10, 3))      # 7    (10 - 3)
print(operator.mul(3, 4))       # 12   (3 * 4)
print(operator.truediv(7, 2))   # 3.5  (7 / 2)
print(operator.floordiv(7, 2))  # 3    (7 // 2)
print(operator.mod(7, 3))       # 1    (7 % 3)
print(operator.pow(2, 10))      # 1024 (2 ** 10)
print(operator.neg(5))          # -5   (-5)

# --- Сравнения ---
print(operator.lt(1, 2))  # True   (1 < 2)
print(operator.le(2, 2))  # True   (2 <= 2)
print(operator.eq(2, 2))  # True   (2 == 2)
print(operator.ne(1, 2))  # True   (1 != 2)
print(operator.gt(3, 2))  # True   (3 > 2)
print(operator.ge(2, 3))  # False  (2 >= 3)

# --- Логические/прочие ---
print(operator.not_(False))       # True
print(operator.truth([]))         # False  (bool([]))
print(operator.contains([1,2], 2))# True   (2 in [1,2])  ВНИМАНИЕ: порядок (контейнер, элемент)
print(operator.concat([1], [2]))  # [1, 2] (конкатенация последовательностей)
print(operator.getitem([10,20], 1))  # 20  ([10,20][1])

# --- Битовые ---
print(operator.and_(6, 3))  # 2  (6 & 3)
print(operator.or_(4, 1))   # 5  (4 | 1)
print(operator.xor(5, 1))   # 4  (5 ^ 1)
```

### Использование с sorted / map / reduce / filter

```python
import operator
from operator import itemgetter, attrgetter, methodcaller
from functools import reduce

# reduce + операторная функция (произведение)
print(reduce(operator.mul, [1, 2, 3, 4], 1))  # 24

# map + operator: попарное сложение двух списков
a, b = [1, 2, 3], [10, 20, 30]
print(list(map(operator.add, a, b)))  # [11, 22, 33]

# sorted + itemgetter: топ по значению
scores = {'Аня': 90, 'Боря': 75, 'Вася': 88}
top = sorted(scores.items(), key=itemgetter(1), reverse=True)
print(top)  # [('Аня', 90), ('Вася', 88), ('Боря', 75)]

# max/min с key
print(max(scores.items(), key=itemgetter(1)))  # ('Аня', 90)

# filter + методколлер (оставить непустые после strip)
lines = ['  hello ', '   ', 'world']
print(list(filter(methodcaller('strip'), lines)))  # ['  hello ', 'world']
```

### Inplace-операторы (i-версии)

Для расширенного присваивания есть функции с приставкой `i`: `iadd`, `imul` и т.д. Для неизменяемых типов они эквивалентны обычным, для изменяемых — модифицируют на месте.

```python
import operator

lst = [1, 2]
operator.iadd(lst, [3, 4])  # эквивалент lst += [3, 4]
print(lst)  # [1, 2, 3, 4]  — изменён на месте

x = 5
x = operator.iadd(x, 3)  # для int возвращает новое значение
print(x)  # 8
```

## Частые вопросы на собеседовании

**Q1. Зачем нужен модуль `operator`, если есть операторы и lambda?**
A: Чтобы передавать операции как функции в `map/filter/sorted/reduce`. `operator.add` быстрее эквивалентной lambda (реализация на C, нет накладных расходов вызова Python-функции), читаемее и сериализуем (picklable), в отличие от lambda.

**Q2. В чём разница между `itemgetter` и `attrgetter`?**
A: `itemgetter(k)` использует доступ по индексу/ключу `obj[k]` (списки, кортежи, словари). `attrgetter('a')` использует доступ к атрибуту `obj.a` (объекты, экземпляры классов). `attrgetter` ещё поддерживает вложенность через точку: `attrgetter('a.b.c')`.

**Q3. Что возвращает `itemgetter(0, 2)` при вызове?**
A: Кортеж из элементов по индексам 0 и 2. `itemgetter(0, 2)('abcd')` -> `('a', 'c')`. С одним аргументом возвращается одиночное значение, с несколькими — кортеж.

**Q4. Как отсортировать список словарей по полю? А по нескольким полям?**
A: По одному: `sorted(data, key=itemgetter('age'))`. По нескольким: `sorted(data, key=itemgetter('age', 'name'))` — сначала по age, при равенстве по name. Это короче и быстрее lambda.

**Q5. Что делает `methodcaller` и когда он удобен?**
A: `methodcaller('m', args)` возвращает функцию, вызывающую `obj.m(args)`. Удобен в `map`/`sorted`, когда нужно вызвать метод объекта: `map(methodcaller('upper'), words)`. Эквивалент `lambda o: o.m(args)`, но быстрее и picklable.

**Q6. Почему `operator.contains` имеет «обратный» порядок аргументов?**
A: `operator.contains(container, item)` соответствует `item in container`, но аргументы идут в порядке (контейнер, элемент). Это частый источник ошибок: `operator.contains([1,2,3], 2)` -> True.

**Q7. Чем `operator.add` быстрее `lambda a, b: a + b`?**
A: `operator.add` реализована на C и вызывается напрямую, без создания кадра стека Python и интерпретации байт-кода тела lambda. На больших итерациях разница заметна.

**Q8. Как с помощью `operator` посчитать произведение списка?**
A: `from functools import reduce; reduce(operator.mul, nums, 1)`. Начальное значение 1 защищает от пустого списка.

**Q9. Можно ли использовать геттеры `operator` с `groupby`?**
A: Да, и это идиоматично. `itertools.groupby(sorted(data, key=itemgetter('city')), key=itemgetter('city'))`. Один и тот же геттер используется и для сортировки, и для группировки.

**Q10. В чём разница между `truediv` и `floordiv`?**
A: `truediv(7, 2)` -> 3.5 (обычное деление `/`), `floordiv(7, 2)` -> 3 (целочисленное `//`). Аналогично операторам Python.

**Q11. Сериализуются ли (pickle) `itemgetter`/`attrgetter`/lambda?**
A: `itemgetter` и `attrgetter` — picklable (в современном Python), поэтому подходят для multiprocessing. Lambda НЕ сериализуется стандартным pickle, что вызовет ошибку при передаче в процессы.

**Q12. Как получить вложенный атрибут одним геттером?**
A: `attrgetter('customer.address.city')` — точечная нотация извлечёт `obj.customer.address.city`. `itemgetter` так не умеет (нет вложенности).

## Подводные камни (gotchas)

1. **Порядок аргументов `operator.contains`** — `contains(container, item)`, что соответствует `item in container`. Легко перепутать.

2. **`itemgetter` с несколькими аргументами возвращает кортеж** — даже `itemgetter(0)(x)` (один) даёт скаляр, а `itemgetter(0, 1)(x)` — кортеж. Это меняет тип результата.

3. **`attrgetter` не работает по индексу, `itemgetter` — по атрибутам** — не путайте `obj.attr` и `obj[key]`.

4. **`itemgetter` не поддерживает вложенность** — `itemgetter('a.b')` ищет ключ `'a.b'` буквально, не `obj['a']['b']`.

5. **lambda vs operator при multiprocessing** — lambda не pickle-ится; для параллельной обработки используйте `itemgetter`/`attrgetter`.

6. **`operator.div` не существует в Python 3** — используйте `truediv` или `floordiv`.

7. **`methodcaller` фиксирует аргументы один раз** — переданные при создании args/kwargs используются при каждом вызове.

8. **Inplace-функции (`iadd` и т.д.)** для неизменяемых типов возвращают новый объект — нужно присваивать результат: `x = operator.iadd(x, 1)`.

## Лучшие практики

- Для извлечения полей в `key=` предпочитайте `itemgetter`/`attrgetter` вместо lambda — быстрее, читаемее, picklable.
- Используйте один геттер и для сортировки, и для группировки (`sorted` + `groupby`).
- Для multiprocessing/pickle избегайте lambda — берите функции из `operator`.
- В `reduce` подавайте `operator.add/mul/...` вместо lambda для скорости.
- Сортировка по нескольким полям — `itemgetter(f1, f2)` / `attrgetter(a1, a2)` вместо составной lambda.
- Помните про порядок аргументов специфичных функций (`contains`, `getitem`).
- Для вызова метода в pipeline — `methodcaller`, а не lambda.

## Шпаргалка

```python
import operator
from operator import itemgetter, attrgetter, methodcaller
from functools import reduce

# --- Геттеры ---
itemgetter(1)(seq)            # seq[1]            (индекс/ключ)
itemgetter('k')(d)           # d['k']
itemgetter(0, -1)(seq)       # (seq[0], seq[-1]) — кортеж
attrgetter('age')(obj)       # obj.age
attrgetter('a.b.c')(obj)     # obj.a.b.c         (вложенно)
attrgetter('x', 'y')(obj)    # (obj.x, obj.y)
methodcaller('upper')(s)     # s.upper()
methodcaller('split', ',')(s)# s.split(',')

# --- Сортировка / агрегация ---
sorted(dicts, key=itemgetter('age'))
sorted(dicts, key=itemgetter('age', 'name'))   # по двум полям
sorted(objs,  key=attrgetter('age'))
max(items, key=itemgetter(1))
list(map(methodcaller('strip'), lines))

# --- Операторные функции ---
operator.add(a, b)       # a + b
operator.sub / mul / truediv / floordiv / mod / pow / neg
operator.lt / le / eq / ne / gt / ge        # сравнения
operator.and_ / or_ / xor / not_            # логика/биты
operator.contains(cont, x)  # x in cont   (ВНИМАНИЕ: порядок!)
operator.getitem(seq, i)    # seq[i]
operator.concat(a, b)       # a + b для последовательностей

# --- reduce + operator ---
reduce(operator.add, nums)        # сумма
reduce(operator.mul, nums, 1)     # произведение
list(map(operator.add, a, b))     # поэлементное сложение списков
```
