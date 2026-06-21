# array — подготовка к собеседованию

## Что это и зачем

`array` — это модуль стандартной библиотеки Python, предоставляющий класс `array.array`: **типизированный массив фиксированного типа элементов**, который хранит числовые данные (или символы Unicode) в виде плотного непрерывного блока C-данных в памяти.

Главная идея: в отличие от обычного `list`, который хранит **указатели на Python-объекты** (каждое целое — это полноценный объект `int` со своим заголовком), `array.array` хранит **сырые машинные значения** того же типа подряд, как массив в языке C.

```python
import array

# Массив 32-битных знаковых целых
a = array.array('i', [1, 2, 3, 4, 5])
print(a)            # => array('i', [1, 2, 3, 4, 5])
print(a.typecode)   # => i
print(a.itemsize)   # => 4  (4 байта на элемент)
```

Зачем это нужно:

- **Экономия памяти.** Миллион чисел в `list` занимает в разы больше памяти, чем в `array`, потому что list хранит указатели + сами объекты int (а int в CPython — это ~28 байт). В `array('i', ...)` каждое число занимает ровно 4 байта.
- **Плотное (cache-friendly) хранение.** Данные лежат подряд в памяти, что улучшает локальность кеша процессора при последовательном обходе.
- **Бинарная сериализация.** Удобно читать/писать массивы чисел в бинарные файлы (`tofile`/`fromfile`) и преобразовывать в `bytes` (`tobytes`/`frombytes`).
- **Интеграция с C-кодом и буферами.** `array` поддерживает buffer protocol, поэтому совместим с `memoryview`, `struct`, сокетами, файлами и многими C-расширениями.

Когда **не** использовать `array`: если нужны векторные/матричные вычисления (сложение массивов, broadcasting, линейная алгебра) — берите `numpy`. Если нужен разнородный контейнер — `list`. Если работаете с чистыми байтами — `bytes`/`bytearray`.

## Ключевые концепции

**1. Один тип на весь массив.** При создании задаётся `typecode` — однобуквенный код типа. Все элементы строго этого типа; попытка положить значение вне диапазона или неподходящего типа вызывает исключение.

```python
import array

a = array.array('b', [1, 2, 3])   # signed char, диапазон -128..127
a.append(127)                      # ok
try:
    a.append(128)                  # выход за диапазон
except OverflowError as e:
    print("OverflowError:", e)     # => OverflowError: signed char is greater than maximum

try:
    a.append(3.14)                 # неправильный тип для целочисленного кода
except TypeError as e:
    print("TypeError:", e)         # => TypeError: 'float' object cannot be interpreted as an integer
```

**2. itemsize и буфер.** Каждый элемент занимает `a.itemsize` байт. Весь массив — это непрерывный буфер длиной `len(a) * a.itemsize` байт.

```python
import array

a = array.array('d', [1.0, 2.0, 3.0])   # double, 8 байт
print(a.itemsize)        # => 8
print(len(a))            # => 3
addr, count = a.buffer_info()
print(count)             # => 3  (число элементов)
print(count * a.itemsize)  # => 24  (всего байт)
```

**3. Платформозависимые vs фиксированные размеры.** Размеры кодов `i`, `l`, `L` исторически зависят от платформы/компилятора (модель данных C). Например, `l` (long) — 8 байт на типичном 64-битном Linux/macOS, но 4 байта на Windows. Коды `q`/`Q` (long long) гарантированно 8 байт. Всегда проверяйте `itemsize`, если важна точная разрядность, или используйте `struct` с явными форматами.

**4. Порядок байт (endianness).** `array` хранит данные в **нативном** порядке байт (как на текущей машине). При сериализации в файл/сеть и чтении на другой архитектуре это важно. Метод `byteswap()` переворачивает байты внутри каждого элемента (например, между little-endian и big-endian).

**5. Buffer protocol.** `array` реализует протокол буфера — его можно обернуть в `memoryview`, передать в `struct.unpack_from`, записать в файл/сокет без копирования.

## Основные функции/классы/методы

### Конструктор `array.array(typecode, [initializer])`

```python
import array

# Из списка
a = array.array('i', [10, 20, 30])
print(a)                  # => array('i', [10, 20, 30])

# Пустой массив заданного типа
b = array.array('f')
print(b)                  # => array('f')

# Из строки байт для числового кода (initializer = bytes) — НЕЛЬЗЯ напрямую,
# для байтового заполнения используют frombytes (см. ниже)

# Код 'u' — массив символов Unicode, инициализируется строкой
u = array.array('u', 'привет')
print(u)                  # => array('u', 'привет')
print(u.tounicode())      # => привет

# Из другого array того же кода
c = array.array('i', a)
print(c)                  # => array('i', [10, 20, 30])

# Из любого итерируемого
d = array.array('i', range(5))
print(d)                  # => array('i', [0, 1, 2, 3, 4])
```

### Таблица кодов типов

| Код | C-тип               | Python-тип | Размер (байт) | Диапазон (типично)                  |
|-----|---------------------|------------|---------------|-------------------------------------|
| `b` | signed char         | int        | 1             | -128 .. 127                         |
| `B` | unsigned char       | int        | 1             | 0 .. 255                            |
| `u` | wchar_t / Py_UCS4   | str (1 симв.) | 2 или 4    | символ Unicode                      |
| `h` | signed short        | int        | 2             | -32768 .. 32767                     |
| `H` | unsigned short      | int        | 2             | 0 .. 65535                          |
| `i` | signed int          | int        | 2 или 4       | обычно -2^31 .. 2^31-1              |
| `I` | unsigned int        | int        | 2 или 4       | обычно 0 .. 2^32-1                  |
| `l` | signed long         | int        | 4 или 8       | платформозависимо                   |
| `L` | unsigned long       | int        | 4 или 8       | платформозависимо                   |
| `q` | signed long long    | int        | 8             | -2^63 .. 2^63-1                     |
| `Q` | unsigned long long  | int        | 8             | 0 .. 2^64-1                         |
| `f` | float               | float      | 4             | одинарная точность IEEE 754         |
| `d` | double              | float      | 8             | двойная точность IEEE 754           |

Примечания:
- `u` (Unicode) считается устаревшим начиная с Python 3.3 и помечен на удаление в будущих версиях; для текста используйте `str`. Размер `wchar_t` зависит от сборки (2 на Windows, 4 на Linux/macOS).
- `q`/`Q` доступны всегда (с Python 3.3).
- Точные размеры всегда определяйте через `array.array(code).itemsize`.

```python
import array
# Проверка размеров на текущей платформе
for code in 'bBhHiIlLqQfd':
    print(code, array.array(code).itemsize, end='  |  ')
# Пример вывода (64-bit macOS):
# b 1  |  B 1  |  h 2  |  H 2  |  i 4  |  I 4  |  l 8  |  L 8  |  q 8  |  Q 8  |  f 4  |  d 8  |
```

### Атрибуты

```python
import array
a = array.array('i', [5, 6, 7])
print(a.typecode)    # => i   (код типа массива)
print(a.itemsize)    # => 4   (размер одного элемента в байтах)
```

### Методы изменения/доступа (как у list)

```python
import array
a = array.array('i', [1, 2, 3])

a.append(4)                  # добавить элемент в конец
print(a)                     # => array('i', [1, 2, 3, 4])

a.extend([5, 6])             # добавить несколько (итерируемое того же типа)
print(a)                     # => array('i', [1, 2, 3, 4, 5, 6])

a.insert(0, 99)              # вставить по индексу
print(a)                     # => array('i', [99, 1, 2, 3, 4, 5, 6])

x = a.pop()                  # удалить и вернуть последний (или по индексу)
print(x, a)                  # => 6 array('i', [99, 1, 2, 3, 4, 5])

a.remove(99)                 # удалить первое вхождение значения
print(a)                     # => array('i', [1, 2, 3, 4, 5])

print(a.index(3))            # => 2   (индекс первого вхождения)
print(a.count(2))            # => 1   (число вхождений)

a.reverse()                  # развернуть на месте
print(a)                     # => array('i', [5, 4, 3, 2, 1])

# Поддерживаются срезы, индексация, конкатенация (+), умножение (*)
print(a[1:3])                # => array('i', [4, 3])
print(a + array.array('i', [0]))  # => array('i', [5, 4, 3, 2, 1, 0])
print(array.array('i', [1]) * 3)  # => array('i', [1, 1, 1])
```

### `buffer_info()`

Возвращает кортеж `(address, length)`: адрес буфера в памяти и число элементов (не байт!).

```python
import array
a = array.array('d', [1.0, 2.0, 3.0, 4.0])
addr, length = a.buffer_info()
print(length)                       # => 4
total_bytes = length * a.itemsize
print(total_bytes)                  # => 32
print(hex(addr)[:2])                # => 0x  (адрес — целое, печать как hex)
```

### `tobytes()` / `frombytes()` — бинарная сериализация

```python
import array
a = array.array('i', [1, 2, 3])

raw = a.tobytes()            # сериализация в bytes (нативный порядок байт)
print(raw)                   # => b'\x01\x00\x00\x00\x02\x00\x00\x00\x03\x00\x00\x00'
print(len(raw))              # => 12  (3 * 4 байта)

b = array.array('i')
b.frombytes(raw)             # десериализация: длина bytes должна быть кратна itemsize
print(b)                     # => array('i', [1, 2, 3])

# Ошибка, если длина не кратна itemsize:
try:
    array.array('i').frombytes(b'\x01\x02\x03')   # 3 байта, не кратно 4
except ValueError as e:
    print("ValueError:", e)  # => ValueError: bytes length not a multiple of item size
```

### `tolist()` / `fromlist()`

```python
import array
a = array.array('i', [10, 20, 30])
print(a.tolist())            # => [10, 20, 30]   (обычный list)

b = array.array('i')
b.fromlist([1, 2, 3])        # добавить элементы из списка
print(b)                     # => array('i', [1, 2, 3])
# Если хоть один элемент не подходит, fromlist бросит исключение,
# но элементы до ошибки уже будут добавлены — частичное изменение.
```

### `tofile()` / `fromfile()` — бинарные файлы

```python
import array
a = array.array('d', [3.14, 2.71, 1.41])

# Запись в бинарный файл
with open('/tmp/data.bin', 'wb') as f:
    a.tofile(f)

# Чтение: fromfile(f, n) читает РОВНО n элементов
b = array.array('d')
with open('/tmp/data.bin', 'rb') as f:
    b.fromfile(f, 3)
print(b)                     # => array('d', [3.14, 2.71, 1.41])

# Если в файле меньше n элементов — читает сколько есть и бросает EOFError
b2 = array.array('d')
with open('/tmp/data.bin', 'rb') as f:
    try:
        b2.fromfile(f, 100)
    except EOFError as e:
        print("EOFError, прочитано:", len(b2))  # => EOFError, прочитано: 3
```

### `byteswap()` — смена порядка байт

```python
import array
a = array.array('i', [1])
print(a.tobytes())     # => b'\x01\x00\x00\x00'  (little-endian)
a.byteswap()           # перевернуть байты каждого элемента на месте
print(a.tobytes())     # => b'\x00\x00\x00\x01'  (теперь big-endian представление)
print(a[0])            # => 16777216  (0x01000000 как int)
```

### `tounicode()` / `fromunicode()` (только для кода `u`)

```python
import array
u = array.array('u', 'abc')
u.fromunicode('XYZ')         # добавить символы из строки
print(u.tounicode())         # => abcXYZ
```

## Частые вопросы на собеседовании

**Q1: Чем `array.array` отличается от `list` и когда его выбирать?**

A: `list` — это динамический массив **указателей на произвольные Python-объекты**, элементы могут быть любых типов. `array.array` хранит **сырые числовые значения одного фиксированного типа** подряд, как массив в C. Выбирайте `array`, когда: (1) нужно хранить много однотипных чисел и важна экономия памяти; (2) нужна плотная упаковка для бинарной сериализации (`tobytes`/`tofile`); (3) нужна совместимость с buffer protocol / C-кодом. Выбирайте `list`, если данные разнородны или нужна максимальная гибкость. Если нужны векторные операции — `numpy`.

```python
import array, sys
lst = list(range(1000))
arr = array.array('i', range(1000))
print(sys.getsizeof(lst))   # => ~8056  (только сам список указателей; объекты int отдельно)
print(sys.getsizeof(arr))   # => ~4064  (1000 * 4 байта + заголовок)
# Реальная экономия ещё больше: list дополнительно держит объекты int (~28 байт каждый),
# хотя мелкие int кешируются. На больших уникальных числах разница огромна.
```

**Q2: Что произойдёт, если положить значение вне диапазона типа или другого типа?**

A: Будет исключение. Выход за числовой диапазон кода → `OverflowError`. Несовместимый тип (например, float в целочисленный массив или str) → `TypeError`. Это и есть «типобезопасность» массива — он гарантирует, что все элементы влезают в выбранный C-тип.

```python
import array
a = array.array('B', [])       # unsigned char: 0..255
try:
    a.append(256)
except OverflowError as e:
    print(e)                   # => unsigned byte integer is greater than maximum
try:
    a.append(-1)
except OverflowError as e:
    print(e)                   # => unsigned byte integer is less than minimum
try:
    a.append("x")
except TypeError as e:
    print(e)                   # => 'str' object cannot be interpreted as an integer
```

**Q3: Как `array` экономит память по сравнению с `list`? Покажите на `sys.getsizeof`.**

A: `list` хранит массив C-указателей (по 8 байт на 64-битной системе) на отдельные объекты `int`/`float`, каждый из которых имеет накладные расходы (~28 байт у int). `array` хранит только сами значения нужной разрядности подряд. Для `array('b', ...)` это 1 байт на число вместо 8 (указатель) + 28 (объект).

```python
import array, sys
N = 100_000
lst = [0] * N
arr = array.array('b', bytes(N))   # 1 байт на элемент
print(sys.getsizeof(lst))   # => ~800056   (~8 байт * N + заголовок)
print(sys.getsizeof(arr))   # => ~100064   (~1 байт * N + заголовок)
```

**Q4: Что возвращает `buffer_info()` и как узнать общий размер буфера в байтах?**

A: `buffer_info()` возвращает кортеж `(address, length)`, где `address` — адрес начала буфера в памяти, а `length` — **число элементов** (не байт). Общий размер в байтах = `length * itemsize`. Полезно для низкоуровневой работы (передача в C, проверка локальности). `itemsize` — размер одного элемента.

```python
import array
a = array.array('q', [1, 2, 3])   # long long, 8 байт
addr, n = a.buffer_info()
print(n)                   # => 3
print(n * a.itemsize)      # => 24
```

**Q5: Как сериализовать массив в байты и обратно? Что с порядком байт между машинами?**

A: `tobytes()` отдаёт сырые байты в **нативном** порядке байт, `frombytes()` читает их обратно (длина должна быть кратна `itemsize`). Проблема: на little-endian и big-endian машинах байты разложены по-разному. Если файл/данные передаются между архитектурами, нужно договориться о порядке байт и при необходимости применить `byteswap()`, либо использовать `struct` с явным указанием порядка (`<` little, `>` big, `!` network). `struct` надёжнее для межплатформенного бинарного формата, так как там endianness и размеры заданы явно.

```python
import array, sys
a = array.array('i', [1, 256])
raw = a.tobytes()
print(sys.byteorder)       # => little (на x86/ARM обычно)
print(raw)                 # => b'\x01\x00\x00\x00\x00\x01\x00\x00'

# Получатель на big-endian машине должен перевернуть байты:
b = array.array('i')
b.frombytes(raw)
b.byteswap()               # привести к своему порядку, если порядок не совпал
# Альтернатива через struct с явным порядком:
import struct
packed = struct.pack('<2i', 1, 256)   # явно little-endian
print(packed == raw)       # => True
```

**Q6: Чем `array.array` отличается от `bytes`/`bytearray`?**

A: `bytes`/`bytearray` — это последовательности **байтов** (значения 0..255), фактически массив unsigned char. `array.array` может хранить элементы **любого числового типа** (int разной разрядности, float, double), и при индексации возвращает числа этого типа, а не байты. `bytearray` изменяем (как array), `bytes` — нет. Фактически `array('B', ...)` концептуально близок к `bytearray`, но `array` даёт более широкий выбор `typecode`. Для работы именно с байтовыми потоками берите `bytes`/`bytearray`; для типизированных чисел — `array`.

```python
import array
ba = bytearray([1, 2, 3])
print(ba[0], type(ba[0]))          # => 1 <class 'int'>  (всегда 0..255)

af = array.array('f', [1.5, 2.5])
print(af[0], type(af[0]))          # => 1.5 <class 'float'>
print(af.tobytes())                # => b'\x00\x00\xc0?\x00\x00 @'  (4 байта на float)
# bytearray не умеет хранить float как float — только сырые байты
```

**Q7: Когда выбрать `numpy.ndarray` вместо `array.array`?**

A: `array.array` хорош как лёгкий, без зависимостей контейнер типизированных чисел и для бинарного I/O. Но он **не умеет векторных операций**: `a + b` — это конкатенация, а не поэлементное сложение; нет broadcasting, slicing-view с шагом по памяти, математических функций, многомерности. Если нужны вычисления (сложение/умножение массивов, агрегации, линейная алгебра, многомерные данные, производительные операции в C) — берите `numpy`. `numpy` тоже плотно хранит данные и поддерживает buffer protocol, поэтому конвертация дешёвая.

```python
import array
a = array.array('i', [1, 2, 3])
b = array.array('i', [10, 20, 30])
print(a + b)               # => array('i', [1, 2, 3, 10, 20, 30])  -- КОНКАТЕНАЦИЯ!

import numpy as np
na = np.array([1, 2, 3])
nb = np.array([10, 20, 30])
print(na + nb)             # => [11 22 33]  -- поэлементно

# Дешёвая конвертация array -> numpy (часто без копирования)
arr = array.array('d', [1.0, 2.0, 3.0])
nd = np.frombuffer(arr, dtype=np.float64)
print(nd)                  # => [1. 2. 3.]
```

**Q8: Как `array` интегрируется с `memoryview`?**

A: `array` поддерживает buffer protocol, поэтому его можно обернуть в `memoryview` для доступа к данным без копирования. Через memoryview можно читать/писать элементы, делать срезы-views, передавать в `struct`, файлы, сокеты. Изменения через memoryview отражаются в исходном array (и наоборот), пока memoryview жив — менять длину array (append и т.п.) при активном memoryview нельзя (`BufferError`).

```python
import array
a = array.array('i', [1, 2, 3, 4])
mv = memoryview(a)
print(mv[1])               # => 2
mv[1] = 200                # запись отражается в array
print(a)                   # => array('i', [1, 200, 3, 4])
print(mv.format, mv.itemsize)  # => i 4

# Срез-view без копирования
sub = mv[1:3]
print(sub.tolist())        # => [200, 3]

try:
    a.append(5)            # пока memoryview жив — нельзя менять размер
except BufferError as e:
    print("BufferError:", e)  # => BufferError: cannot resize an array that is exporting buffers
```

**Q9: Как прочитать/записать бинарный файл массивом и какие подводные камни?**

A: `tofile(f)` пишет весь массив в открытый бинарный файл; `fromfile(f, n)` читает **ровно n элементов**. Если в файле меньше — читает сколько есть и бросает `EOFError` (но уже прочитанное остаётся в массиве). Чтобы прочитать весь файл, можно узнать число элементов через размер файла / itemsize, либо читать через `frombytes(f.read())`. Файл обязательно открывать в бинарном режиме (`'wb'`/`'rb'`). Endianness сохраняется нативный — для переносимости документируйте порядок байт.

```python
import array, os
a = array.array('h', [10, 20, 30, 40])   # short, 2 байта
with open('/tmp/arr.bin', 'wb') as f:
    a.tofile(f)

size = os.path.getsize('/tmp/arr.bin')
n = size // array.array('h').itemsize
b = array.array('h')
with open('/tmp/arr.bin', 'rb') as f:
    b.fromfile(f, n)
print(b)                   # => array('h', [10, 20, 30, 40])

# Альтернатива — прочитать весь файл и frombytes:
c = array.array('h')
with open('/tmp/arr.bin', 'rb') as f:
    c.frombytes(f.read())
print(c)                   # => array('h', [10, 20, 30, 40])
```

**Q10: Зависит ли размер `i`/`l` от платформы и как писать переносимый код?**

A: Да. Коды `i`/`I` (int), `l`/`L` (long) соответствуют C-типам, чьи размеры зависят от модели данных платформы/компилятора. Классический пример: `l` = 8 байт на 64-битных Linux/macOS (LP64), но 4 байта на 64-битном Windows (LLP64). Гарантированно фиксированы: `b/B`=1, `h/H`=2, `q/Q`=8, `f`=4, `d`=8. Для переносимого бинарного формата либо используйте только гарантированные коды и явный `byteswap`, либо `struct` с форматами фиксированного размера и явным порядком байт. Всегда проверяйте `itemsize` в рантайме, если разрядность критична.

```python
import array
# Безопаснее зафиксировать разрядность:
print(array.array('q').itemsize)   # => 8  (везде)
print(array.array('f').itemsize)   # => 4  (везде)
# А вот это может отличаться:
print(array.array('l').itemsize)   # => 8 на Linux/macOS, 4 на Windows
```

**Q11: Можно ли смешивать типы или менять `typecode` после создания?**

A: Нет. `typecode` фиксируется при создании и неизменяем. Нельзя `extend`/конкатенировать массивы разных кодов — будет `TypeError`. Чтобы «сменить тип», создайте новый массив из данных (через `tolist()` или через сырые байты с reinterpret).

```python
import array
a = array.array('i', [1, 2, 3])
try:
    a.extend(array.array('f', [1.0]))
except TypeError as e:
    print(e)               # => can only extend with array of same kind

# "Смена" типа через значения:
b = array.array('d', a.tolist())
print(b)                   # => array('d', [1.0, 2.0, 3.0])

# Reinterpret тех же байтов как другой тип (битовое перетолкование):
src = array.array('i', [1])
reinterpret = array.array('f')
reinterpret.frombytes(src.tobytes())
print(reinterpret[0])      # => 1.401298464324817e-45  (биты int 1 как float)
```

**Q12: Какова сложность операций у `array`?**

A: Та же, что у `list` для аналогичных операций: индексация и `append` (амортизированно) — O(1); `insert`/`remove`/`pop(i)` в середине — O(n) из-за сдвига; `index`/`count`/`in` — O(n); `extend` — O(k). Дополнительно у array есть очень быстрые bulk-операции `tobytes`/`frombytes`/`tofile`/`fromfile`, работающие с непрерывным буфером (по сути memcpy), что заметно быстрее поэлементной сериализации `list`.

## Подводные камни (gotchas)

```python
import array

# 1. '+' и '*' — это конкатенация и повторение, а НЕ арифметика!
a = array.array('i', [1, 2, 3])
print(a * 2)          # => array('i', [1, 2, 3, 1, 2, 3])   (НЕ [2,4,6])
print(a + a)          # => array('i', [1, 2, 3, 1, 2, 3])   (НЕ поэлементно)

# 2. frombytes требует длину, кратную itemsize, иначе ValueError
try:
    array.array('i').frombytes(b'\x00\x00\x00')   # 3 байта
except ValueError as e:
    print(e)          # => bytes length not a multiple of item size

# 3. fromfile при нехватке данных читает что есть и бросает EOFError
#    (часть данных уже в массиве — не забудьте обработать)

# 4. Нативный порядок байт: tobytes/tofile НЕ переносимы между endianness
#    без явного byteswap или использования struct.

# 5. Нельзя менять размер array, пока экспортируется буфер (memoryview жив) -> BufferError

# 6. Код 'u' (Unicode array) устарел и будет удалён — не используйте для нового кода.

# 7. OverflowError, а не молчаливое усечение, при выходе за диапазон.
b = array.array('B')
try:
    b.append(300)
except OverflowError as e:
    print(e)          # => unsigned byte integer is greater than maximum

# 8. fromlist при ошибке оставляет частично добавленные элементы.
c = array.array('i', [1])
try:
    c.fromlist([2, 3, 'oops'])
except TypeError:
    pass
print(c)              # => array('i', [1, 2, 3])  -- 2 и 3 уже добавились!

# 9. Файлы для tofile/fromfile нужно открывать в бинарном режиме ('wb'/'rb').

# 10. sys.getsizeof(array) показывает размер буфера, но не учитывает,
#     что list дополнительно держит объекты-элементы — реальная экономия
#     array больше, чем разница getsizeof.
```

## Лучшие практики

- **Выбирайте минимально достаточный тип.** Если значения 0..255 — берите `B`, а не `i`. Это экономит память и пропускную способность I/O.
- **Не путайте конкатенацию с арифметикой.** Для векторных вычислений используйте `numpy`, а не `array`.
- **Для межплатформенной сериализации** используйте `struct` с явным порядком байт (`<`, `>`, `!`) и фиксированными форматами, либо документируйте endianness и применяйте `byteswap()`. Не полагайтесь на нативный `tobytes` между разными архитектурами.
- **Проверяйте `itemsize` в рантайме**, если разрядность критична (`i`/`l`/`L` платформозависимы). Предпочитайте `q`/`Q` для гарантированных 8 байт.
- **Для bulk I/O** используйте `tobytes`/`frombytes`/`tofile`/`fromfile` вместо поэлементных циклов — это быстрые memcpy-операции.
- **Используйте `memoryview`** для доступа без копирования при передаче данных в C/struct/сокеты; не меняйте размер массива, пока memoryview активен.
- **Обрабатывайте исключения** `OverflowError`/`TypeError` при заполнении данными из внешних источников и `EOFError`/`ValueError` при десериализации.
- **Не используйте код `u`** в новом коде — он устарел; для текста используйте обычный `str`.
- **Если нужны вычисления, многомерность, broadcasting** — сразу берите `numpy`; конвертация дешёвая через `np.frombuffer`/`np.asarray`.

## Шпаргалка

```python
import array

# Создание
a = array.array('i', [1, 2, 3])     # из итерируемого
e = array.array('d')                # пустой double-массив

# Атрибуты
a.typecode        # 'i'
a.itemsize        # размер элемента в байтах
a.buffer_info()   # (адрес, число_элементов)

# Изменение (как у list)
a.append(x); a.extend(iter); a.insert(i, x)
a.pop([i]); a.remove(x); a.reverse()
a.index(x); a.count(x); len(a); a[i]; a[i:j]

# Конвертации
a.tolist(); a.fromlist([...])
a.tobytes(); a.frombytes(b'...')           # длина кратна itemsize
a.tofile(f); a.fromfile(f, n)              # бинарный режим, n элементов
a.byteswap()                                # смена порядка байт на месте

# Только код 'u' (устаревший)
u = array.array('u', 'txt'); u.tounicode(); u.fromunicode('...')

# Коды типов (фиксированные размеры): b/B=1, h/H=2, q/Q=8, f=4, d=8
# Платформозависимые: i/I (2|4), l/L (4|8)

# Память: array << list для однотипных чисел (sys.getsizeof)
# Арифметика: a + b и a * n — КОНКАТЕНАЦИЯ, не математика -> используйте numpy
# Переносимая сериализация: struct.pack('<...') / '>...' с явным порядком байт
# Без копирования: memoryview(a), np.frombuffer(a, dtype=...)
```
