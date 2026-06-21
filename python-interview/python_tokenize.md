# tokenize / keyword / token — подготовка к собеседованию

## Что это и зачем

Эти три модуля относятся к **лексическому уровню** обработки Python-кода — самой первой стадии, когда исходный текст разбивается на токены (лексемы), ещё до построения AST.

```
исходный код (текст)
   → ТОКЕНИЗАЦИЯ (tokenize)        ← поток токенов
   → парсинг (ast)
   → байткод (dis)
   → выполнение
```

- **`tokenize`** — разбивает исходный код Python на поток токенов (числа, строки, имена, операторы, отступы, переводы строк). Это лексер Python, доступный из самого Python.
- **`token`** — содержит числовые константы типов токенов (`NAME`, `NUMBER`, `OP`, `NEWLINE`, `INDENT`, ...) и вспомогательные структуры.
- **`keyword`** — список зарезервированных слов Python (`if`, `for`, `def`, `class`, ...) и «мягких» ключевых слов (`match`, `case`, `_`, `type`).

**Зачем нужны:**
- Написание линтеров, форматтеров, подсветки синтаксиса, инструментов рефакторинга, которым важны комментарии и точное расположение токенов (AST их теряет).
- Безопасный анализ кода без выполнения.
- Проверка, является ли строка валидным идентификатором / не зарезервированным словом (генераторы кода, ORM, валидация имён).
- Подсчёт метрик, обфускация, минификация.

## Ключевые концепции

1. **Токен** — минимальная значимая единица: имя (`NAME`), число (`NUMBER`), строковый литерал (`STRING`), оператор/разделитель (`OP`), отступ (`INDENT`/`DEDENT`), конец логической строки (`NEWLINE`), пустая физическая строка (`NL`), комментарий (`COMMENT`), конец файла (`ENDMARKER`).

2. **`TokenInfo`** — именованный кортеж, который выдаёт `tokenize`: `(type, string, start, end, line)`, где `start`/`end` — пары `(строка, столбец)`.

3. **Значимость отступов.** В отличие от многих языков, в Python отступы — это токены `INDENT`/`DEDENT`. Токенайзер сам отслеживает уровни вложенности.

4. **`NEWLINE` vs `NL`.** `NEWLINE` завершает логическую инструкцию; `NL` — незначащий перевод строки (пустые строки, переносы внутри скобок). Это важно различать при анализе.

5. **Ключевые слова (keywords)** — зарезервированы и не могут быть именами переменных (`keyword.kwlist`). **Мягкие ключевые слова (soft keywords)** — контекстно-зависимы: `match`, `case`, `_`, `type` (`keyword.softkwlist`); ими формально можно назвать переменную, но в определённых конструкциях они особенные.

6. **Обратимость.** `tokenize.untokenize` способен восстановить исходный код из токенов (приблизительно/точно при полной информации) — основа инструментов трансформации, сохраняющих комментарии.

## Основные функции/классы/методы

### tokenize.tokenize / generate_tokens — поток токенов

```python
import tokenize
import io

source = "x = 10  # переменная\ndef f():\n    return x + 1\n"

# generate_tokens работает со строковым потоком (str), читает построчно
tokens = tokenize.generate_tokens(io.StringIO(source).readline)
for tok in tokens:
    # tok — это TokenInfo(type, string, start, end, line)
    print(f"{tokenize.tok_name[tok.type]:<12} {tok.string!r:<15} "
          f"начало={tok.start} конец={tok.end}")
```

Примерный вывод (фрагмент):

```
NAME         'x'             начало=(1, 0) конец=(1, 1)
OP           '='             начало=(1, 2) конец=(1, 3)
NUMBER       '10'            начало=(1, 4) конец=(1, 6)
COMMENT      '# переменная'  начало=(1, 8) ...
NEWLINE      '\n'            ...
NAME         'def'           ...
...
INDENT       '    '          ...
...
DEDENT       ''              ...
ENDMARKER    ''              ...
```

Разница между функциями:
- `tokenize.tokenize(readline)` — работает с **байтовым** потоком (`readline` возвращает `bytes`), первым выдаёт токен `ENCODING`.
- `tokenize.generate_tokens(readline)` — работает со **строковым** потоком (`str`), без токена `ENCODING`. Удобнее для строк в памяти.

### Токенизация файла

```python
import tokenize

# Корректно определяет кодировку из PEP 263 (# -*- coding: ... -*-)
with tokenize.open("example.py") as f:
    for tok in tokenize.generate_tokens(f.readline):
        if tok.type == tokenize.COMMENT:
            print("Комментарий:", tok.string)
```

### Практика: подсчёт комментариев и строк кода

```python
import tokenize, io

def count_comments(source: str) -> int:
    """Считает строки-комментарии — AST бы их не увидел."""
    count = 0
    for tok in tokenize.generate_tokens(io.StringIO(source).readline):
        if tok.type == tokenize.COMMENT:
            count += 1
    return count

src = "a = 1  # один\n# два\nb = 2\n"
print(count_comments(src))            # 2
```

### untokenize — восстановление кода

```python
import tokenize, io

source = "x=1+2\n"
tokens = list(tokenize.generate_tokens(io.StringIO(source).readline))

# Можно модифицировать токены и собрать обратно
restored = tokenize.untokenize(tokens)
print(repr(restored))                 # восстановленный исходник
```

### Модуль token — типы токенов

```python
import token

# Числовые константы типов
print(token.NAME, token.NUMBER, token.OP, token.NEWLINE)

# Имя типа по номеру
print(token.tok_name[token.NAME])     # 'NAME'

# Проверки (3.8+)
print(token.ISTERMINAL(token.NAME))   # терминальный ли символ
print(token.ISEOF(token.ENDMARKER))   # ENDMARKER?

# Сопоставление строки оператора с типом
print(token.EXACT_TOKEN_TYPES["+"])   # тип для '+'
```

### Модуль keyword — ключевые слова

```python
import keyword

# Полный список зарезервированных слов
print(keyword.kwlist)
# ['False', 'None', 'True', 'and', 'as', 'assert', 'async', 'await',
#  'break', 'class', 'continue', 'def', 'del', 'elif', 'else', 'except',
#  'finally', 'for', 'from', 'global', 'if', 'import', 'in', 'is',
#  'lambda', 'nonlocal', 'not', 'or', 'pass', 'raise', 'return', 'try',
#  'while', 'with', 'yield']

# Мягкие ключевые слова (контекстные): match, case, _, type
print(keyword.softkwlist)

# Проверки — частый кейс: валидация генерируемых имён
print(keyword.iskeyword("for"))       # True  — нельзя использовать как имя
print(keyword.iskeyword("match"))     # False — это мягкое ключевое слово
print(keyword.issoftkeyword("match")) # True
print(keyword.iskeyword("name"))      # False
```

### Практика: безопасное имя для генерации кода

```python
import keyword

def safe_identifier(name: str) -> str:
    """Гарантирует валидный неконфликтный идентификатор Python."""
    if not name.isidentifier():
        raise ValueError(f"{name!r} не является идентификатором")
    if keyword.iskeyword(name):
        return name + "_"             # 'class' -> 'class_' (как в dataclasses/attrs)
    return name

print(safe_identifier("class"))       # class_
print(safe_identifier("user_id"))     # user_id
```

### Практика: простой «переименователь» переменных через токены

```python
import tokenize, io

def rename_var(source: str, old: str, new: str) -> str:
    """Заменяет имя переменной, не трогая строки и комментарии."""
    result = []
    tokens = tokenize.generate_tokens(io.StringIO(source).readline)
    for tok in tokens:
        if tok.type == tokenize.NAME and tok.string == old:
            tok = tok._replace(string=new)   # TokenInfo — namedtuple
        result.append(tok)
    return tokenize.untokenize(result)

print(rename_var("total = total + 1  # total\n", "total", "sum_"))
# sum_ = sum_ + 1  # total   <-- внутри комментария не тронуто
```

## Частые вопросы на собеседовании

**Q1. На каком этапе работает токенизация и чем отличается от AST?**
A: Токенизация — первый, лексический этап (текст → поток токенов), до парсинга. AST — уровень выше (структура программы). Токены сохраняют комментарии, точные позиции, отступы; AST их отбрасывает. Для линтеров/форматтеров, которым важны комментарии и форматирование, нужны токены.

**Q2. В чём разница между `tokenize.tokenize` и `generate_tokens`?**
A: `tokenize` работает с байтовым потоком (readline → `bytes`) и выдаёт первым токен `ENCODING`. `generate_tokens` — со строковым (`str`), без `ENCODING`. Для строк в памяти удобнее `generate_tokens(io.StringIO(s).readline)`.

**Q3. Что такое токены INDENT/DEDENT и зачем они?**
A: Поскольку в Python отступы значимы, токенайзер генерирует `INDENT` при увеличении уровня вложенности и `DEDENT` при уменьшении. Это позволяет парсеру понимать блочную структуру без фигурных скобок.

**Q4. Чем отличаются NEWLINE и NL?**
A: `NEWLINE` завершает логическую инструкцию (значимый перевод строки). `NL` — незначащий перевод: пустые строки, переносы строк внутри скобок/скобочных выражений. При анализе кода их нужно различать.

**Q5. Что хранит `TokenInfo`?**
A: Именованный кортеж `(type, string, start, end, line)`: числовой тип токена, его текст, координаты начала и конца `(строка, столбец)` и текст исходной строки. `_replace` позволяет создавать изменённые копии.

**Q6. Чем мягкие ключевые слова отличаются от обычных?**
A: Обычные (`keyword.kwlist`) зарезервированы всегда — нельзя использовать как имена. Мягкие (`keyword.softkwlist`: `match`, `case`, `_`, `type`) контекстно-зависимы: они особенные только в конкретных конструкциях, а в остальных местах их можно использовать как обычные идентификаторы (что сохраняет обратную совместимость).

**Q7. Как программно проверить, что строка — валидное имя переменной?**
A: `str.isidentifier()` проверяет синтаксическую форму, а `keyword.iskeyword()` — что это не зарезервированное слово. Оба нужны вместе. Типичный приём при конфликте — добавить подчёркивание (`class` → `class_`).

**Q8. Можно ли восстановить исходный код из токенов?**
A: Да, через `tokenize.untokenize`. При полной информации о токенах восстановление точное; это основа инструментов трансформации, сохраняющих комментарии и форматирование (в отличие от `ast.unparse`).

**Q9. Зачем использовать `tokenize.open` вместо обычного `open`?**
A: `tokenize.open` определяет кодировку файла по PEP 263 (строка `# -*- coding: ... -*-` или BOM) и открывает файл в правильной кодировке, как это делает сам интерпретатор.

**Q10. Где применяется токенизация на практике?**
A: Линтеры (`flake8`, `pycodestyle`), форматтеры (`black`, `autopep8`), `isort`, инструменты подсветки, минификаторы, конвертеры `2to3`, инструменты подсчёта метрик и рефакторинга, которым важны комментарии и точные позиции.

**Q11. Сколько ключевых слов в Python и можно ли это узнать программно?**
A: Точное число зависит от версии; получить актуальный список — `len(keyword.kwlist)`. Не заучивайте число — оно меняется (добавлялись `async`/`await`, `nonlocal` и т.д.).

## Подводные камни (gotchas)

- **`tokenize` vs `generate_tokens`**: подача `str` в `tokenize` (ожидает `bytes`) или наоборот — типичная ошибка. Для строк берите `generate_tokens`.
- **`NL` vs `NEWLINE`**: смешение этих токенов ломает анализаторы, считающие инструкции.
- **`DEDENT` имеет пустую строку** (`tok.string == ''`) и нулевую ширину — не полагайтесь на её текст.
- **`untokenize` чувствителен к корректности позиций**: при ручной правке токенов лучше работать в «5-кортежном» режиме или аккуратно пересчитывать координаты.
- **Мягкие ключевые слова можно случайно использовать как имена** (`match = 1`), и это легально вне конструкции `match`, но запутывает читателя.
- **`str.isidentifier()` пропускает ключевые слова**: `"class".isidentifier()` == `True`, поэтому отдельно нужна `keyword.iskeyword`.
- **Кодировка файла**: чтение исходника обычным `open` без учёта кодировки может исказить не-ASCII; используйте `tokenize.open`.
- **Версионная зависимость**: набор soft keywords (`type` появился с PEP 695) и состав `kwlist` меняются между релизами.

## Лучшие практики

- Для анализа кода с сохранением комментариев/форматирования — токены (`tokenize`), а не AST.
- Для строк в памяти используйте `tokenize.generate_tokens(io.StringIO(src).readline)`.
- Для чтения файлов — `tokenize.open`, чтобы корректно учесть кодировку.
- Валидируйте генерируемые имена связкой `str.isidentifier()` + `keyword.iskeyword()`; конфликты решайте суффиксом `_`.
- Различайте `NEWLINE`/`NL` и помните про `INDENT`/`DEDENT` при разборе структуры.
- При трансформациях используйте `TokenInfo._replace(...)` и `untokenize` для обратимости.
- Учитывайте версию Python для состава ключевых и мягких ключевых слов.

## Шпаргалка

```python
import tokenize, token, keyword, io

# Токенизация строки (str)
for tok in tokenize.generate_tokens(io.StringIO(src).readline):
    tok.type, tok.string, tok.start, tok.end, tok.line   # TokenInfo
    tokenize.tok_name[tok.type]                           # имя типа

# Токенизация байтов / файла
tokenize.tokenize(readline_bytes)        # первым выдаёт ENCODING
with tokenize.open("file.py") as f:      # учитывает кодировку (PEP 263)
    ...

# Восстановление
tokenize.untokenize(tokens)              # токены -> исходный код
tok._replace(string="new")               # TokenInfo — namedtuple

# Типы токенов (модуль token)
token.NAME, token.NUMBER, token.STRING, token.OP, token.COMMENT
token.NEWLINE        # конец логической инструкции
token.NL             # незначащий перевод строки
token.INDENT, token.DEDENT, token.ENDMARKER, token.ENCODING
token.tok_name[token.OP]                 # номер -> имя

# Ключевые слова (модуль keyword)
keyword.kwlist                           # все зарезервированные слова
keyword.softkwlist                       # мягкие: match, case, _, type
keyword.iskeyword("for")                 # True
keyword.issoftkeyword("match")           # True

# Валидное имя:
name.isidentifier() and not keyword.iskeyword(name)
```
