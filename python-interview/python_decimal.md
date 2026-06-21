# decimal — подготовка к собеседованию

## Что это и зачем

`decimal` — модуль стандартной библиотеки для **десятичной арифметики с фиксированной и плавающей точкой**, с контролируемой точностью и корректным округлением. Главный класс — `Decimal`.

Зачем нужен:
- **Деньги и финансы.** `float` (двоичный IEEE 754) не может точно представить `0.1`, поэтому для денег он недопустим. `Decimal('0.1')` хранит точное десятичное значение.
- **Контролируемая точность.** Можно задать число значащих цифр (`precision`) и режим округления (`ROUND_*`).
- **Воспроизводимость и соответствие стандарту.** Реализует стандарт IBM General Decimal Arithmetic — те же правила, что в банковских системах и SQL `DECIMAL`.
- **Точные равенства.** `Decimal('0.1') + Decimal('0.2') == Decimal('0.3')` — это `True`.

## Ключевые концепции

- **Контекст (`Context`).** Глобальное (или потоковое) состояние, задающее точность (`prec`), режим округления (`rounding`), ловушки исключений (`traps`). Доступен через `getcontext()`.
- **Точность = значащие цифры, а не знаки после запятой.** `prec=28` (по умолчанию) означает 28 значащих цифр, а не 28 знаков после точки.
- **Создавать из строки, а не из float!** `Decimal(0.1)` унаследует ошибку float (`0.1000000000000000055...`), а `Decimal('0.1')` — точное значение.
- **Quantize для фиксированных знаков.** Округление до N знаков после запятой делается через `.quantize()`.
- **Неизменяемость.** Объекты `Decimal` immutable.

## Основные функции/классы/методы

### Создание Decimal

```python
from decimal import Decimal

# Из строки — ПРАВИЛЬНО (точное значение)
print(Decimal('0.1'))        # 0.1

# Из float — ОПАСНО (наследует двоичную ошибку)
print(Decimal(0.1))          # 0.1000000000000000055511151231257827021181583404541015625

# Из int — точно
print(Decimal(10))           # 10

# Из кортежа (знак, цифры, экспонента)
print(Decimal((0, (3, 1, 4), -2)))  # 3.14  (0=плюс, цифры 3,1,4, exp=-2)

# Точное сравнение
print(Decimal('0.1') + Decimal('0.2') == Decimal('0.3'))  # True
print(0.1 + 0.2 == 0.3)                                    # False (float!)
```

### Контекст и точность

```python
from decimal import Decimal, getcontext

ctx = getcontext()
print(ctx.prec)        # 28 — число значащих цифр по умолчанию

getcontext().prec = 6  # установить 6 значащих цифр
print(Decimal(1) / Decimal(7))   # 0.142857  (6 значащих цифр)

getcontext().prec = 28           # вернуть обратно
print(Decimal(1) / Decimal(7))   # 0.1428571428571428571428571429
```

### localcontext — локальное изменение контекста

```python
from decimal import Decimal, localcontext

# Временно меняем точность только внутри блока with
with localcontext() as ctx:
    ctx.prec = 4
    print(Decimal(1) / Decimal(3))   # 0.3333

# За пределами блока — прежняя точность (28)
print(Decimal(1) / Decimal(3))       # 0.3333333333333333333333333333
```

### Округление: quantize и режимы ROUND_*

```python
from decimal import Decimal, ROUND_HALF_UP, ROUND_HALF_EVEN, ROUND_DOWN, ROUND_UP, ROUND_CEILING, ROUND_FLOOR

x = Decimal('2.675')

# Округление до 2 знаков после запятой
print(x.quantize(Decimal('0.01'), rounding=ROUND_HALF_UP))    # 2.68 (как в школе)
print(x.quantize(Decimal('0.01'), rounding=ROUND_HALF_EVEN))  # 2.67 (банковское!)

# Сравним режимы на 0.5
for r, name in [(ROUND_HALF_UP, 'HALF_UP'), (ROUND_HALF_EVEN, 'HALF_EVEN'),
                (ROUND_DOWN, 'DOWN'), (ROUND_UP, 'UP'),
                (ROUND_CEILING, 'CEILING'), (ROUND_FLOOR, 'FLOOR')]:
    val = Decimal('0.5').quantize(Decimal('1'), rounding=r)
    print(f'{name:12} 0.5 -> {val}')
# HALF_UP      0.5 -> 1   (от нуля при .5)
# HALF_EVEN    0.5 -> 0   (к чётному при .5)
# DOWN         0.5 -> 0   (к нулю)
# UP           0.5 -> 1   (от нуля)
# CEILING      0.5 -> 1   (к +inf)
# FLOOR        0.5 -> 0   (к -inf)

# ROUND_HALF_EVEN ("банковское округление") — режим по умолчанию,
# минимизирует систематическое смещение при массовых округлениях
```

### Деньги: типичный паттерн

```python
from decimal import Decimal, ROUND_HALF_UP

def money(value) -> Decimal:
    """Привести значение к денежному формату: 2 знака, округление HALF_UP."""
    return Decimal(str(value)).quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)

price = money('19.99')
tax = money(price * Decimal('0.20'))   # 20% налог
print(price)   # 19.99
print(tax)     # 4.00
print(price + tax)  # 23.99

# Важно: str(value) защищает от попадания float-ошибки
print(money(0.1 + 0.2))   # 0.30  (str превратит в '0.30000000000000004', quantize обрежет)
```

### Полезные методы и свойства

```python
from decimal import Decimal

d = Decimal('-3.14000')

print(d.as_tuple())          # DecimalTuple(sign=1, digits=(3,1,4,0,0,0), exponent=-5)
print(d.normalize())         # -3.14  (убирает хвостовые нули)
print(d.is_signed())         # True   (отрицательное)
print(Decimal('1.5').sqrt()) # 1.224744871391589049098642037
print(Decimal('100').ln())   # 4.605170185988091368035982909
print(Decimal('100').log10())# 2
print(Decimal('1.234').adjusted())  # 0 — позиция старшей цифры

# Конвертация
print(float(Decimal('0.1')))  # 0.1 (теряем точность)
print(int(Decimal('3.9')))    # 3   (усечение)
print(str(Decimal('3.14')))   # 3.14
```

### Ловушки исключений (traps) и сигналы

```python
from decimal import Decimal, getcontext, DivisionByZero, InvalidOperation, localcontext

# По умолчанию деление на ноль бросает исключение
try:
    Decimal(1) / Decimal(0)
except DivisionByZero:
    print('Поймали деление на ноль')

# Можно отключить ловушку — тогда вернётся Infinity
with localcontext() as ctx:
    ctx.traps[DivisionByZero] = False
    print(Decimal(1) / Decimal(0))   # Infinity
```

## Частые вопросы на собеседовании

**Q1: Почему нельзя использовать `float` для денег?**
A: `float` использует двоичное представление IEEE 754, в котором `0.1`, `0.2`, `0.01` не представимы точно. Накапливаются ошибки округления: `0.1 + 0.2 == 0.30000000000000004`. Для денег это недопустимо. `Decimal` хранит десятичные значения точно.

**Q2: В чём разница между `Decimal('0.1')` и `Decimal(0.1)`?**
A: `Decimal('0.1')` создаётся из строки и хранит точное значение `0.1`. `Decimal(0.1)` создаётся из float `0.1`, который уже содержит двоичную ошибку, поэтому получится `0.1000000000000000055...`. Всегда создавайте из строки или int.

**Q3: Что такое `prec` в контексте — это знаки после запятой?**
A: Нет, `prec` — это число **значащих цифр** (всего), а не знаков после точки. `prec=3` для `1234.5` даст `1.23E+3`. Для фиксированного числа знаков после запятой используют `.quantize(Decimal('0.01'))`.

**Q4: Что такое банковское округление (ROUND_HALF_EVEN) и зачем оно?**
A: При значении ровно `.5` оно округляет к ближайшему чётному (`0.5→0`, `1.5→2`, `2.5→2`). Это режим по умолчанию в `decimal`. Он устраняет систематическое смещение вверх, которое даёт школьное `ROUND_HALF_UP` при массовых округлениях (важно в финансах и статистике).

**Q5: Как округлить Decimal до 2 знаков после запятой?**
A: `value.quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)`. Шаблон `Decimal('0.01')` задаёт число знаков, а `rounding` — режим.

**Q6: Как изменить точность только для части кода?**
A: Через `with localcontext() as ctx: ctx.prec = N`. Это потокобезопасно и автоматически восстанавливает контекст после блока. Менять `getcontext().prec` напрямую — глобальный и опасный побочный эффект.

**Q7: Сохраняет ли Decimal значащие нули?**
A: Да. `Decimal('1.50')` и `Decimal('1.5')` равны по значению (`==` даёт `True`), но имеют разный "масштаб" (exponent). `Decimal('1.50').normalize()` уберёт хвостовой ноль. Это важно при сериализации.

**Q8: Decimal медленнее float?**
A: Да, значительно (в десятки раз), так как реализован программно, а не аппаратно. Для научных массивных вычислений используют `float`/NumPy, для денег и точности — `Decimal`.

**Q9: Можно ли смешивать Decimal и float в арифметике?**
A: Нет, `Decimal('1') + 0.1` бросит `TypeError`. С `int` смешивать можно. Это намеренно — чтобы не занести float-ошибку в Decimal. Сначала приведите float через `Decimal(str(f))`.

**Q10: Является ли Decimal неизменяемым?**
A: Да, объекты `Decimal` immutable (как `int`, `str`). Любая операция возвращает новый объект. Поэтому их можно использовать как ключи словаря и в множествах.

## Подводные камни (gotchas)

- **Точность float при создании (главное!).** `Decimal(0.1)` тащит за собой `0.1000000000000000055...`. Всегда `Decimal('0.1')` или `Decimal(str(x))`.

- **`prec` — значащие цифры, не дробная часть.** Частая путаница. Для денег управляйте через `quantize`, а не через `prec`.

- **Глобальный контекст влияет на всё.** Изменение `getcontext().prec` действует на весь поток и может неожиданно сломать другой код. Используйте `localcontext`.

- **Нельзя смешивать с float.** `Decimal('1') + 1.0` → `TypeError`. Защита от заражения ошибкой, но новичков удивляет.

- **`==` учитывает значение, но `1.50` и `1.5` разный масштаб.** `Decimal('1.5') == Decimal('1.50')` это `True`, но `repr`/хэш/`as_tuple` различаются. Хэши при этом совпадают (равные значения дают равный хэш).

- **ROUND_HALF_UP не «округление как в школе» для отрицательных интуитивно.** `Decimal('-2.5').quantize(Decimal('1'), ROUND_HALF_UP)` даёт `-3` (от нуля), а не `-2`.

- **Деление по умолчанию ограничено `prec`.** `Decimal(1)/Decimal(3)` обрезается до 28 значащих цифр — это не «бесконечная» точность.

- **`quantize` может бросить `InvalidOperation`**, если результат превышает текущую `prec` по числу цифр в целой части.

## Лучшие практики

- Создавайте `Decimal` из строк: `Decimal('19.99')`, или `Decimal(str(x))` для float.
- Для денег фиксируйте формат через `.quantize(Decimal('0.01'), rounding=...)`.
- Явно выбирайте режим округления: `ROUND_HALF_UP` для «обычного» бизнес-округления, `ROUND_HALF_EVEN` для статистики/банков.
- Меняйте точность через `localcontext`, не трогая глобальный контекст.
- Не смешивайте `Decimal` и `float`; конвертируйте границы данных в `Decimal` как можно раньше.
- Храните деньги в БД как `DECIMAL`/`NUMERIC` и читайте в `Decimal`, а не во `float`.
- Для целых денежных единиц (копейки/центы) иногда удобнее хранить `int` (минимальная единица) — но `Decimal` читабельнее.

## Шпаргалка

```python
from decimal import (Decimal, getcontext, localcontext,
                     ROUND_HALF_UP, ROUND_HALF_EVEN, ROUND_DOWN,
                     ROUND_UP, ROUND_CEILING, ROUND_FLOOR)

# Создание (ВСЕГДА из строки!)
Decimal('0.1')          # точно
Decimal(str(x))         # из float безопасно
Decimal(10)             # из int

# Контекст
getcontext().prec = 28          # значащие цифры (глобально)
with localcontext() as ctx:     # локально, потокобезопасно
    ctx.prec = 6

# Округление до N знаков после запятой
Decimal('2.675').quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)   # 2.68
Decimal('2.675').quantize(Decimal('0.01'), rounding=ROUND_HALF_EVEN) # 2.68

# Режимы:
# ROUND_HALF_UP    — .5 от нуля (школьное)
# ROUND_HALF_EVEN  — .5 к чётному (банковское, по умолчанию)
# ROUND_DOWN/UP    — к нулю / от нуля
# ROUND_FLOOR/CEILING — к -inf / +inf

# Методы
d.normalize()       # убрать хвостовые нули
d.as_tuple()        # (знак, цифры, экспонента)
d.sqrt(), d.ln(), d.log10()
float(d), int(d), str(d)

# Деньги
Decimal(str(x)).quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)
```
