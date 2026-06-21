# dis — подготовка к собеседованию

## Что это и зачем

Модуль `dis` (disassembler — дизассемблер) показывает **байткод** CPython — низкоуровневые инструкции, в которые компилируется исходный код Python и которые выполняет виртуальная машина (CPython VM). Это «ассемблер» интерпретатора.

Цепочка обработки кода в CPython:

```
исходник → токены → AST → байткод (code object) → выполнение в VM
                                  ↑
                          dis показывает этот уровень
```

**Зачем нужен на собеседовании и на практике:**
- Понять, как Python на самом деле выполняет код (стековая машина).
- Объяснить производительность: почему один вариант кода быстрее другого, что делает оптимизатор.
- Отладка тонких эффектов: замыкания, `LOAD_FAST` vs `LOAD_GLOBAL`, кэширование констант.
- Понимание устройства функций, генераторов, comprehension'ов.
- Изучение оптимизаций интерпретатора (peephole, в 3.11+ — адаптивные специализирующие инструкции).

## Ключевые концепции

1. **Стековая виртуальная машина.** CPython — стековая машина: инструкции работают со стеком значений. Например, `1 + 2` компилируется в «положи 1», «положи 2», «сложи верхние два, положи результат».

2. **Code object.** Скомпилированный код хранится в объекте `code` (`function.__code__`). У него много атрибутов:
   - `co_code` — сам байткод (байты).
   - `co_consts` — константы (числа, строки, вложенные code objects).
   - `co_names` — глобальные/атрибутные имена.
   - `co_varnames` — локальные переменные.
   - `co_argcount`, `co_flags`, `co_stacksize`, `co_firstlineno` и др.

3. **Инструкция (bytecode instruction).** Состоит из опкода (операции) и опционального аргумента (operand). Каждая инструкция в `dis` описана объектом `Instruction` с полями `opname`, `arg`, `argval`, `offset`, `starts_line`, `is_jump_target`.

4. **Типичные опкоды:**
   - `LOAD_CONST` — положить константу на стек.
   - `LOAD_FAST` / `STORE_FAST` — локальная переменная (быстрый доступ по индексу).
   - `LOAD_GLOBAL` — глобальное имя.
   - `LOAD_NAME` / `STORE_NAME` — имя в текущем пространстве (модуль/класс).
   - `BINARY_OP` (3.11+) или `BINARY_ADD`/`BINARY_MULTIPLY` (раньше) — бинарные операции.
   - `CALL` / `CALL_FUNCTION` — вызов.
   - `RETURN_VALUE` — возврат из функции.
   - `POP_JUMP_IF_FALSE`, `JUMP_FORWARD` — управление потоком.

5. **Оптимизации компилятора:** свёртка констант (constant folding), peephole-оптимизации, в 3.11+ — специализирующий адаптивный интерпретатор (PEP 659, inline-кэши, инструкции вроде `LOAD_ATTR_INSTANCE_VALUE`).

## Основные функции/классы/методы

### dis.dis — дизассемблирование

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
# Примерный вывод (Python 3.11):
#   2           0 RESUME                   0
#   3           2 LOAD_FAST                0 (a)
#               4 LOAD_FAST                1 (b)
#               6 BINARY_OP                0 (+)
#              10 RETURN_VALUE
```

Колонки вывода: номер строки исходника, смещение в байтах (offset), имя опкода, аргумент (число), и в скобках — человекочитаемое значение аргумента.

`dis.dis` принимает функцию, метод, класс, code object, строку с кодом или объект-генератор.

```python
import dis

# Дизассемблировать строку
dis.dis("x = 1 + 2")
#   1   0 LOAD_CONST   0 (3)   <-- 1+2 уже свёрнуто компилятором в константу 3!
#       2 STORE_NAME   0 (x)
```

### Свёртка констант (constant folding)

```python
import dis

# Компилятор вычисляет константные выражения на этапе компиляции
dis.dis("y = 60 * 60 * 24")
#   1   0 LOAD_CONST   0 (86400)   <-- посчитано заранее
#       2 STORE_NAME   0 (y)

# Но с переменной свёртки нет:
dis.dis("y = x * 60")
#   1   0 LOAD_NAME    0 (x)
#       2 LOAD_CONST   0 (60)
#       4 BINARY_OP    5 (*)
#       6 STORE_NAME   1 (y)
```

### Сравнение LOAD_FAST и LOAD_GLOBAL — почему локальные быстрее

```python
import dis

GLOBAL = 10

def uses_global():
    return GLOBAL + 1      # LOAD_GLOBAL — поиск по словарю имён

def uses_local():
    local = 10
    return local + 1       # LOAD_FAST — доступ по индексу в массиве, быстрее

dis.dis(uses_global)       # увидите LOAD_GLOBAL
dis.dis(uses_local)        # увидите LOAD_FAST
# Это объясняет частую микрооптимизацию: кэширование глобалов в локальные.
```

### dis.Bytecode — объектный интерфейс

```python
import dis

def f(x):
    if x > 0:
        return "positive"
    return "non-positive"

bc = dis.Bytecode(f)
print(bc.codeobj.co_name)             # f
print(bc.dis())                       # строка с дизассемблированием

# Итерация по инструкциям как по объектам
for instr in bc:
    print(f"{instr.offset:>3} {instr.opname:<20} arg={instr.argval}")
```

### dis.get_instructions — поток инструкций

```python
import dis

def cube(n):
    return n ** 3

for ins in dis.get_instructions(cube):
    print(ins.opname, ins.argval, "| jump_target:", ins.is_jump_target)
```

### Исследование code object

```python
def greet(name, greeting="Hi"):
    msg = f"{greeting}, {name}"
    return msg

code = greet.__code__
print("Имя:           ", code.co_name)          # greet
print("Аргументов:    ", code.co_argcount)       # 2
print("Локальные:     ", code.co_varnames)       # ('name', 'greeting', 'msg')
print("Константы:     ", code.co_consts)
print("Имена (global):", code.co_names)
print("Стек (макс):   ", code.co_stacksize)
print("Первая строка: ", code.co_firstlineno)
print("Флаги:         ", code.co_flags)          # битовая маска (генератор, корутина и т.д.)
```

### Вложенные code objects (функции, comprehension'ы)

```python
import dis

def outer():
    return [i * 2 for i in range(5)]

# В co_consts функции лежит ОТДЕЛЬНЫЙ code object для list comprehension
for const in outer.__code__.co_consts:
    if hasattr(const, "co_name"):
        print("Вложенный code object:", const.co_name)
        dis.dis(const)        # можно дизассемблировать его отдельно
```

### Сравнение производительности через байткод

```python
import dis

# Конкатенация строк в цикле vs join
def with_plus(items):
    s = ""
    for x in items:
        s += x          # каждый += создаёт новую строку
    return s

def with_join(items):
    return "".join(items)

dis.dis(with_plus)        # цикл с BINARY_OP и пересозданием строки
dis.dis(with_join)        # один вызов метода join
# Байткод наглядно показывает, почему join эффективнее.
```

### Адаптивные инструкции (Python 3.11+)

```python
import dis

def access(obj):
    return obj.value

# С adaptive=True видны специализированные инструкции после «прогрева»
dis.dis(access, adaptive=True)
# Можно увидеть LOAD_ATTR_INSTANCE_VALUE вместо общего LOAD_ATTR
# (специализирующий интерпретатор, PEP 659)
```

### Разбор флагов кода

```python
import dis

def gen():
    yield 1

# CO_GENERATOR — этот код является генераторной функцией
flags = gen.__code__.co_flags
print(dis.show_code(gen))             # печатает полную сводку по code object
# В выводе будет строка Flags: ... GENERATOR ...
```

## Частые вопросы на собеседовании

**Q1. Что такое байткод и где он хранится?**
A: Байткод — низкоуровневые инструкции для виртуальной машины CPython, в которые компилируется исходный код. Хранится в code object (`func.__code__.co_code`), кэшируется на диске в `.pyc` файлах (каталог `__pycache__`).

**Q2. Какая архитектура у виртуальной машины CPython?**
A: Стековая. Инструкции кладут значения на стек и снимают их (в отличие от регистровых VM). Например, сложение снимает два верхних значения и кладёт сумму.

**Q3. Почему `LOAD_FAST` быстрее `LOAD_GLOBAL`?**
A: `LOAD_FAST` обращается к локальной переменной по индексу в массиве слотов (O(1), без хеширования). `LOAD_GLOBAL` ищет имя в словаре глобалов, а при промахе — в builtins. Поэтому кэширование глобалов в локальные ускоряет горячие циклы.

**Q4. Что такое свёртка констант? Покажите на примере.**
A: Компилятор заранее вычисляет константные выражения. `dis.dis("60*60*24")` покажет `LOAD_CONST 86400` — умножение не выполняется в рантайме. Работает только когда все операнды — литералы.

**Q5. Что хранится в co_consts, co_names, co_varnames?**
A: `co_consts` — константы (числа, строки, `None`, вложенные code objects). `co_names` — имена глобалов и атрибутов. `co_varnames` — имена локальных переменных (включая аргументы).

**Q6. Зачем нужен `dis` на практике?**
A: Понять, как выполняется код; объяснить производительность; разобраться в эффектах замыканий/генераторов; изучить оптимизации интерпретатора; отладить неочевидное поведение.

**Q7. Как устроены list comprehension на уровне байткода?**
A: В CPython до 3.12 comprehension компилируется в отдельный вложенный code object (отдельная область видимости), лежащий в `co_consts`. В 3.12 inline-оптимизация изменила это (PEP 709) — comprehensions встраиваются без отдельной функции.

**Q8. Что изменилось в байткоде в Python 3.11?**
A: Появился специализирующий адаптивный интерпретатор (PEP 659): «горячие» инструкции на лету заменяются специализированными версиями (`LOAD_ATTR_INSTANCE_VALUE`, `BINARY_OP_ADD_INT` и т.п.) с inline-кэшами. Добавлена `RESUME`, обобщён `BINARY_OP`. Это дало значимый прирост скорости.

**Q9. Стабилен ли байткод между версиями Python?**
A: Нет. Набор опкодов и их семантика меняются между минорными версиями. Поэтому `.pyc` версионируются (magic number), а код, разбирающий байткод, привязан к версии.

**Q10. Как посмотреть флаги code object и что они значат?**
A: `func.__code__.co_flags` — битовая маска. Биты сообщают, является ли функция генератором (`CO_GENERATOR`), корутиной (`CO_COROUTINE`), использует ли `*args`/`**kwargs` и т.д. Удобно через `dis.show_code(func)`.

**Q11. В чём разница между dis.dis и dis.get_instructions?**
A: `dis.dis` печатает человекочитаемый дизассемблер. `dis.get_instructions` возвращает итератор объектов `Instruction` для программного анализа (offset, opname, argval, is_jump_target).

## Подводные камни (gotchas)

- **Вывод `dis` зависит от версии Python.** Не заучивайте конкретные опкоды — между 3.8, 3.11, 3.12 они отличаются (`BINARY_ADD` → `BINARY_OP`).
- **Свёртка констант не всегда срабатывает.** Большие результаты или операции с не-литералами не сворачиваются. Раньше большие строки/последовательности компилятор намеренно не сворачивал, чтобы не раздувать `.pyc`.
- **`is` с маленькими int / короткими строками** — следствие кэширования/интернирования констант, видного в `co_consts`; не полагайтесь на `is` для сравнения значений.
- **`dis` функции с adaptive=False** показывает «исходные» инструкции; специализированные появляются только после реального исполнения и с `adaptive=True`.
- **Comprehension как отдельный scope** (до 3.12) объясняет, почему переменная цикла не «течёт» наружу — это отдельный code object.
- **Не оптимизируйте по байткоду вслепую.** Меньше инструкций не всегда = быстрее; профилируйте (`timeit`, `cProfile`).

## Лучшие практики

- Используйте `dis` как инструмент понимания и обучения, а не для микрооптимизаций без замеров.
- Для программного анализа берите `dis.get_instructions` / `dis.Bytecode`, а не парсинг текста.
- Подтверждайте гипотезы о производительности через `timeit`/`cProfile`, а `dis` — для объяснения «почему».
- Помните о версии Python — фиксируйте интерпретатор при сравнении байткода.
- Для изучения внутренностей сочетайте `dis` с `ast` (уровень выше) и `co_*` атрибутами code object.

## Шпаргалка

```python
import dis

dis.dis(obj)                 # дизассемблировать функцию/код/строку/класс
dis.dis("x = 1 + 2")         # из строки исходника
dis.show_code(func)          # сводка по code object (флаги, consts, и т.д.)
dis.dis(func, adaptive=True) # специализированные инструкции (3.11+)

# Объектный интерфейс
bc = dis.Bytecode(func)
for ins in bc: ...           # Instruction: opname, arg, argval, offset
for ins in dis.get_instructions(func): ...

# Code object
c = func.__code__
c.co_code        # сам байткод (bytes)
c.co_consts      # константы (+ вложенные code objects)
c.co_names       # глобальные/атрибутные имена
c.co_varnames    # локальные переменные
c.co_argcount    # число позиционных аргументов
c.co_flags       # флаги (генератор, корутина, *args ...)
c.co_stacksize   # макс. глубина стека
c.co_firstlineno # первая строка

# Ключевые опкоды
# LOAD_CONST, LOAD_FAST (локал.), LOAD_GLOBAL, LOAD_NAME, STORE_*,
# BINARY_OP, COMPARE_OP, CALL, RETURN_VALUE,
# POP_JUMP_IF_FALSE, JUMP_FORWARD, FOR_ITER, RESUME

# Идея: LOAD_FAST > LOAD_GLOBAL по скорости; константы сворачиваются на компиляции
```
