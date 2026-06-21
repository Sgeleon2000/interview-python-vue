# Модуль `string` и форматирование строк — подготовка к собеседованию

## Что это и зачем

Модуль `string` — это часть стандартной библиотеки Python, которая содержит:

- **Строковые константы** (`ascii_letters`, `digits`, `punctuation` и т.д.) — удобные наборы символов для генерации, валидации, фильтрации.
- **Класс `Template`** — простой и безопасный способ подстановки переменных в строки (на основе `$`), особенно полезен для работы с пользовательским вводом и локализацией.
- **Класс `Formatter`** — низкоуровневый движок форматирования, лежащий в основе метода `str.format()`; его можно расширять для кастомного синтаксиса.

Помимо самого модуля `string`, тема «форматирование строк» включает три ключевых механизма языка:

1. `str.format()` и format spec mini-language;
2. f-strings (форматированные строковые литералы, PEP 498, начиная с Python 3.6);
3. устаревший `%`-оператор (printf-style).

На собеседовании важно понимать различия между этими подходами, знать mini-language спецификаторов формата и уметь объяснить, когда какой инструмент уместен.

## Ключевые концепции

- **f-strings — самый быстрый и читаемый способ.** Выражение вычисляется во время выполнения, вставляется в литерал. Это синтаксис уровня компилятора, а не функция.
- **`str.format()` — гибкий, но многословный.** Подходит, когда шаблон строки приходит извне (из конфига, БД), потому что f-string нельзя «отложить».
- **`Template` — безопасный.** Не выполняет произвольный код и форматирование, только подстановку имён. Идеален для шаблонов от пользователей.
- **`%`-форматирование — легаси.** Всё ещё встречается (особенно в логировании), но новый код пишут на f-strings/`format`.
- **Format Spec Mini-Language** — единый «язык» спецификаторов (`:>10.2f`), который работает и в `format()`, и в f-strings, и в `format()`-функции.

## Основные функции/классы/методы

### Строковые константы

```python
import string

print(string.ascii_lowercase)  # abcdefghijklmnopqrstuvwxyz
print(string.ascii_uppercase)  # ABCDEFGHIJKLMNOPQRSTUVWXYZ
print(string.ascii_letters)    # abc...XYZ (lower + upper)
print(string.digits)           # 0123456789
print(string.hexdigits)        # 0123456789abcdefABCDEF
print(string.octdigits)        # 01234567
print(string.punctuation)      # !"#$%&'()*+,-./:;<=>?@[\]^_`{|}~
print(string.whitespace)       # ' \t\n\r\x0b\x0c' (пробельные символы)
print(string.printable)        # digits + letters + punctuation + whitespace
```

Практический пример — генерация надёжного пароля:

```python
import string
import secrets  # криптостойкий генератор, НЕ random

alphabet = string.ascii_letters + string.digits + string.punctuation
password = ''.join(secrets.choice(alphabet) for _ in range(16))
print(password)  # например: 'gT4$kL9!mZ2@pW7q'
```

Валидация — проверим, что строка состоит только из «безопасных» символов:

```python
import string

allowed = set(string.ascii_letters + string.digits + '_')
def is_valid_identifier_chars(s: str) -> bool:
    return bool(s) and set(s) <= allowed

print(is_valid_identifier_chars("user_42"))  # True
print(is_valid_identifier_chars("user-42"))  # False (дефис не разрешён)
```

### Класс `string.Template`

`Template` использует синтаксис `$identifier` или `${identifier}`. Методы:

- `substitute(mapping, **kwargs)` — бросает `KeyError`, если переменная не найдена;
- `safe_substitute(...)` — оставляет неизвестные плейсхолдеры как есть, не падает.

```python
from string import Template

t = Template("Привет, $name! Тебе $age лет.")

# substitute требует все переменные
print(t.substitute(name="Аня", age=30))
# Привет, Аня! Тебе 30 лет.

# ${} нужны, когда после имени идёт буква/цифра
t2 = Template("Файл: ${name}_backup.txt")
print(t2.substitute(name="db"))  # Файл: db_backup.txt

# $$ — экранированный знак доллара
t3 = Template("Цена: $$$price")
print(t3.substitute(price=100))  # Цена: $100

# safe_substitute не падает на отсутствующих ключах
t4 = Template("Привет, $name из $city!")
print(t4.safe_substitute(name="Иван"))  # Привет, Иван из $city!
```

Подстановка из словаря и кастомизация разделителя:

```python
from string import Template

data = {"host": "localhost", "port": 5432}
print(Template("postgres://$host:$port/db").substitute(data))
# postgres://localhost:5432/db

# Кастомный синтаксис: меняем '$' на '%'
class PercentTemplate(Template):
    delimiter = '%'

pt = PercentTemplate("Значение: %value")
print(pt.substitute(value=42))  # Значение: 42
```

### Класс `string.Formatter`

`Formatter` — расширяемый движок. Полезен, когда нужен собственный синтаксис форматирования или динамический доступ к полям.

```python
from string import Formatter

f = Formatter()

# vformat принимает позиционные и именованные аргументы
result = f.format("{0} + {1} = {result}", 2, 3, result=5)
print(result)  # 2 + 3 = 5

# parse() разбирает строку формата на токены
for literal, field, spec, conversion in Formatter().parse("Имя: {name:>10}!"):
    print(repr(literal), field, repr(spec), conversion)
# 'Имя: ' name '>10' None
# '!' None None None
```

Кастомный `Formatter`, который не падает на отсутствующих ключах:

```python
from string import Formatter

class SafeFormatter(Formatter):
    def get_value(self, key, args, kwargs):
        if isinstance(key, str):
            return kwargs.get(key, f"{{{key}}}")  # вернуть плейсхолдер
        return super().get_value(key, args, kwargs)

sf = SafeFormatter()
print(sf.format("{greeting}, {name}!", greeting="Привет"))
# Привет, {name}!
```

### `str.format()` и метод форматирования

```python
# Позиционные аргументы
print("{} и {}".format("кофе", "чай"))         # кофе и чай
print("{0} и {1} и {0}".format("A", "B"))      # A и B и A

# Именованные аргументы
print("{name} = {value}".format(name="x", value=10))  # x = 10

# Доступ к атрибутам и элементам
point = {"x": 1, "y": 2}
print("{p[x]},{p[y]}".format(p=point))         # 1,2

import datetime
d = datetime.date(2026, 6, 20)
print("Год: {d.year}".format(d=d))             # Год: 2026

# Распаковка
data = {"first": "Иван", "last": "Петров"}
print("{first} {last}".format(**data))         # Иван Петров
```

### f-strings (форматированные литералы)

```python
name = "Мир"
count = 42
pi = 3.14159

print(f"Привет, {name}!")            # Привет, Мир!
print(f"Счёт: {count}")              # Счёт: 42

# Любые выражения внутри {}
print(f"Сумма: {2 + 2}")             # Сумма: 4
print(f"Верхний: {name.upper()}")    # Верхний: МИР

# Спецификаторы формата после ':'
print(f"Pi = {pi:.2f}")              # Pi = 3.14

# '=' для отладки (Python 3.8+): печатает выражение и значение
x = 10
print(f"{x = }")                     # x = 10
print(f"{x * 2 = }")                 # x * 2 = 20

# Вложенные поля: ширина задаётся переменной
width = 8
print(f"{count:{width}d}")           # '      42' (ширина 8)

# Вызовы, тернарники, методы
items = [1, 2, 3]
print(f"Элементов: {len(items)}, пусто: {'да' if not items else 'нет'}")
# Элементы: 3, пусто: нет
```

Важно про кавычки и фигурные скобки:

```python
d = {"key": "значение"}
# В Python < 3.12 нельзя использовать те же кавычки внутри f-string
print(f"{d['key']}")        # значение

# Двойная скобка экранирует одиночную
print(f"{{не выражение}}")  # {не выражение}

# f-string с многострочностью
msg = (
    f"Строка 1 с {name}\n"
    f"Строка 2 с {count}"
)
```

### Format Spec Mini-Language

Полный синтаксис спецификатора:
`[[fill]align][sign][#][0][width][grouping_option][.precision][type]`

```python
n = 1234.5678

# Выравнивание: < влево, > вправо, ^ по центру, = знак слева
print(f"{42:<10}|")    # '42        |'  влево
print(f"{42:>10}|")    # '        42|'  вправо
print(f"{42:^10}|")    # '    42    |'  по центру
print(f"{42:*^10}|")   # '****42****|'  fill='*'

# Знак: + всегда, - только минус, ' ' пробел для положительных
print(f"{42:+}")       # +42
print(f"{-42:+}")      # -42
print(f"{42: }")       # ' 42' (пробел перед положительным)

# Точность для float
print(f"{n:.2f}")      # 1234.57
print(f"{n:10.2f}")    # '   1234.57'
print(f"{n:,.2f}")     # 1,234.57 (разделитель тысяч запятой)
print(f"{n:_.2f}")     # 1_234.57 (разделитель подчёркиванием)

# Системы счисления (для int)
print(f"{255:b}")      # 11111111  двоичная
print(f"{255:o}")      # 377       восьмеричная
print(f"{255:x}")      # ff        шестнадцатеричная (нижний регистр)
print(f"{255:X}")      # FF        верхний регистр
print(f"{255:#x}")     # 0xff      с префиксом
print(f"{255:08b}")    # 00000011  с нулями до ширины 8 -> 11111111

# Проценты и научная нотация
print(f"{0.1234:.1%}") # 12.3%
print(f"{12345:.2e}")  # 1.23e+04

# Строки: точность = обрезка
print(f"{'привет':.3}")  # при (первые 3 символа)
print(f"{'abc':>8}")     # '     abc'
```

Таблица типов:

| Тип | Описание |
|-----|----------|
| `d` | десятичное целое |
| `b` `o` `x` `X` | дв./восьм./шестн. |
| `f` `F` | float с фикс. точкой |
| `e` `E` | научная нотация |
| `g` `G` | общий (выбирает f или e) |
| `%` | проценты (умножает на 100) |
| `s` | строка (по умолчанию) |
| `c` | символ по коду |
| `n` | число с учётом локали |

### Сравнение с `%`-форматированием (legacy)

```python
name, age = "Аня", 30
print("Имя: %s, возраст: %d" % (name, age))   # Имя: Аня, возраст: 30
print("Pi = %.2f" % 3.14159)                   # Pi = 3.14
print("%(name)s — %(age)d" % {"name": name, "age": age})

# Важно для логирования: ленивое форматирование
import logging
log = logging.getLogger(__name__)
log.debug("Пользователь %s сделал %d запросов", name, age)  # форматируется ТОЛЬКО если уровень DEBUG активен
```

## Частые вопросы на собеседовании

**Q1: В чём разница между `str.format()`, f-strings и `%`-форматированием?**
A: f-strings (3.6+) — самые быстрые и читаемые, выражение вычисляется на месте, но шаблон нельзя «отложить». `str.format()` — медленнее, но шаблон можно хранить отдельно (в конфиге, БД). `%` — устаревший printf-стиль, остаётся в логировании ради ленивого форматирования. f-strings предпочтительны в новом коде, кроме случаев с отложенным шаблоном.

**Q2: Когда f-string нельзя использовать и нужен `format()`?**
A: Когда строка-шаблон неизвестна на этапе написания кода — приходит из файла, БД, перевода (i18n). f-string — литерал, его нельзя построить динамически. Тогда хранят `"{name}"` и вызывают `.format(name=...)`.

**Q3: Чем `Template.substitute` отличается от `safe_substitute`?**
A: `substitute` бросает `KeyError`/`ValueError` при отсутствии переменной или неверном синтаксисе. `safe_substitute` не падает — оставляет нераспознанные плейсхолдеры в виде `$name`. Второй безопаснее для частичной подстановки и пользовательского ввода.

**Q4: Почему `string.Template` считается «безопасным» по сравнению с `format`?**
A: `Template` делает только текстовую подстановку имён, не вычисляет выражения, не вызывает методы, не обращается к атрибутам. `str.format()` позволяет `{obj.__class__}` и подобное, что при форматировании пользовательских шаблонов открывает доступ к внутренностям объектов (потенциальная утечка данных). Поэтому для шаблонов от пользователя берут `Template`.

**Q5: Что делает `f"{x=}"`?**
A: Синтаксис самодокументирующих выражений (Python 3.8+). Печатает и само выражение, и его значение: `f"{x=}"` → `"x=10"`. Удобно для отладки. Можно с пробелами: `f"{x = }"` → `"x = 10"`, и со спецификатором: `f"{pi=:.2f}"`.

**Q6: Как вывести знак доллара в `Template`?**
A: Удвоить: `$$`. `Template("$$5").substitute()` → `"$5"`.

**Q7: Что такое format spec mini-language и где он применяется?**
A: Единый синтаксис спецификаторов после `:` — `[[fill]align][sign][#][0][width][,][.prec][type]`. Работает в `str.format()`, f-strings и функции `format()`. Например `:>10.2f` — выравнивание вправо, ширина 10, 2 знака после точки.

**Q8: Как добавить разделитель тысяч?**
A: `f"{1234567:,}"` → `"1,234,567"` (запятая) или `f"{1234567:_}"` → `"1_234_567"` (подчёркивание). Можно комбинировать с точностью: `f"{n:,.2f}"`.

**Q9: Как реализовать собственный формат для своего класса?**
A: Определить метод `__format__(self, spec)`. Он вызывается при `format(obj, spec)` и в f-strings. `spec` — строка спецификатора после `:`.
```python
class Money:
    def __init__(self, amount): self.amount = amount
    def __format__(self, spec):
        return format(self.amount, spec) + " ₽"
print(f"{Money(1234.5):,.2f}")  # 1,234.50 ₽
```

**Q10: Какая разница между `!r`, `!s`, `!a` в форматировании?**
A: Это conversion flags — применяются ДО format spec. `!s` → `str()`, `!r` → `repr()`, `!a` → `ascii()`. Пример: `f"{name!r}"` использует `repr`, добавляя кавычки: `'Аня'`.

**Q11: Производительность: что быстрее?**
A: f-strings обычно быстрее всех, так как разбираются компилятором в эффективный байткод (FORMAT_VALUE/BUILD_STRING). `%` и `.format()` медленнее из-за разбора строки в рантайме. Но для логирования используют `%`-стиль с ленивым форматированием, чтобы не платить за форматирование при выключенном уровне логов.

## Подводные камни (gotchas)

```python
# 1. f-string вычисляется немедленно — нельзя сохранить шаблон
template = "{name}"          # это работает с .format()
# f"{name}"                  # вычислится сразу, name должен существовать

# 2. KeyError при substitute без safe_substitute
from string import Template
try:
    Template("$a $b").substitute(a=1)  # KeyError: 'b'
except KeyError as e:
    print("Нет ключа:", e)

# 3. Логирование: НЕ форматируйте заранее
import logging
log = logging.getLogger()
# Плохо — форматирование происходит ВСЕГДА:
log.debug(f"Дорогой объект: {expensive()}")  # expensive() вызовется
# Хорошо — лениво, только при включённом DEBUG:
log.debug("Дорогой объект: %s", expensive)   # передаём, не вызываем заранее

# 4. Округление .Nf использует банковское округление
print(f"{2.5:.0f}")   # 2 (round half to even!)
print(f"{3.5:.0f}")   # 4
print(f"{0.125:.2f}") # 0.12 (а не 0.13)

# 5. Одинаковые кавычки в f-string до Python 3.12
d = {'k': 1}
# f"{d["k"]}"  # SyntaxError в 3.11, OK в 3.12+
print(f"{d['k']}")  # надёжный способ

# 6. format() и пустые {} нельзя смешивать с номерами
# "{} {0}".format(...)  # ValueError: нельзя смешивать авто- и ручную нумерацию
```

## Лучшие практики

- **Используйте f-strings** для большинства задач — читаемость и скорость.
- **`str.format()`** — только когда шаблон отложен/внешний.
- **`string.Template`** — для шаблонов из недоверенных источников и i18n.
- **В логировании** применяйте `%`-стиль с аргументами (`log.info("%s", x)`), не f-strings — ради ленивого форматирования и совместимости со структурным логированием.
- Для денежных и табличных данных используйте mini-language (`:,.2f`, выравнивание).
- Не злоупотребляйте сложной логикой внутри f-strings — выносите в переменные ради читаемости.
- Для генерации секретов берите `secrets`, а не `random` + `string`.

## Шпаргалка

```python
import string

# Константы
string.ascii_letters   # a-zA-Z
string.digits          # 0-9
string.punctuation     # знаки препинания
string.whitespace      # пробельные

# Template
from string import Template
Template("$x").substitute(x=1)        # подстановка, падает если нет
Template("$x").safe_substitute()      # не падает, оставит $x

# f-strings
f"{x}"          # значение
f"{x!r}"        # repr
f"{x:.2f}"      # 2 знака после точки
f"{x:>10}"      # вправо, ширина 10
f"{x:^10}"      # по центру
f"{x:08.2f}"    # нули, ширина 8, 2 знака
f"{x:,}"        # разделитель тысяч
f"{x:x}"        # hex
f"{x:#b}"       # двоичное с 0b
f"{x:.1%}"      # процент
f"{x=}"         # отладка: 'x=значение'

# format()
"{} {}".format(a, b)            # позиционные
"{0} {1} {0}".format(a, b)      # с индексами
"{name}".format(name=x)         # именованные
"{p.attr}".format(p=obj)        # атрибут
"{d[key]}".format(d=dct)        # элемент

# Spec: [fill]align sign # 0 width , .prec type
# align: < > ^ =   type: d b o x f e g % s c n
```
