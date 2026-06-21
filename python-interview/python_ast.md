# ast — подготовка к собеседованию

## Что это и зачем

Модуль `ast` (Abstract Syntax Tree — абстрактное синтаксическое дерево) предоставляет доступ к синтаксической структуре исходного кода Python в виде дерева объектов. Когда интерпретатор обрабатывает код, он проходит несколько стадий:

```
исходный код (текст)
   → токенизация (tokenize)
   → парсинг (построение AST)        ← здесь работает модуль ast
   → компиляция в байткод (модуль dis показывает результат)
   → выполнение в виртуальной машине CPython
```

`ast` даёт программный доступ к промежуточному представлению — дереву, которое описывает структуру программы независимо от её текстового написания (отступы, скобки, комментарии уже отброшены).

**Зачем нужен на практике:**
- Статический анализ кода (linters: `flake8`, `pylint`, форматтеры, `bandit` для поиска уязвимостей).
- Трансформация кода / метапрограммирование (макросы, автоматическая инструментация, перезапись выражений).
- Безопасное вычисление литералов без `eval` (`ast.literal_eval`).
- Извлечение метаданных: список импортов, докстрингов, сигнатур функций без выполнения кода.
- Инструменты вроде `pytest` (переписывание assert для подробных сообщений об ошибках), `mypy`, `black`, `isort`.

## Ключевые концепции

1. **Узлы (Node).** Дерево состоит из узлов — экземпляров классов, наследующих `ast.AST`. Примеры: `Module`, `FunctionDef`, `BinOp`, `Call`, `Name`, `Constant`, `Assign`.

2. **Поля узла.** У каждого типа узла есть:
   - `_fields` — кортеж имён дочерних полей (например, у `BinOp` это `('left', 'op', 'right')`).
   - `_attributes` — служебные атрибуты позиции: `lineno`, `col_offset`, `end_lineno`, `end_col_offset`.

3. **Типы узлов по ролям:**
   - **Выражения (expr)** — вычисляются в значение: `BinOp`, `Call`, `Name`, `Constant`, `Compare`.
   - **Инструкции (stmt)** — действия: `Assign`, `If`, `For`, `FunctionDef`, `Return`, `Import`.
   - **Контексты (expr_context)** — `Load`, `Store`, `Del` — определяют, читается переменная или ей присваивают.
   - **Операторы** — `Add`, `Sub`, `Mult`, `Eq`, `And` и т.п.

4. **Обход дерева.** `ast.walk` (рекурсивный обход всех узлов в произвольном порядке), `ast.NodeVisitor` (паттерн «Посетитель»), `ast.NodeTransformer` (посетитель, способный заменять узлы).

5. **Корень дерева.** Результат `ast.parse(...)` по умолчанию — узел `Module` со списком инструкций в поле `body`.

## Основные функции/классы/методы

### ast.parse — построение дерева

```python
import ast

source = """
def add(a, b):
    return a + b
"""

tree = ast.parse(source)               # mode='exec' по умолчанию
print(type(tree))                      # <class 'ast.Module'>
print(ast.dump(tree, indent=4))        # читаемое текстовое представление дерева
```

Параметр `mode`:
- `'exec'` — модуль (несколько инструкций), результат `Module`.
- `'eval'` — одно выражение, результат `Expression`.
- `'single'` — одна интерактивная инструкция (как в REPL), результат `Interactive`.

```python
# Разбор одного выражения
expr_tree = ast.parse("1 + 2 * 3", mode="eval")
print(ast.dump(expr_tree))
# Expression(body=BinOp(left=Constant(value=1),
#            op=Add(),
#            right=BinOp(left=Constant(2), op=Mult(), right=Constant(3))))
```

### ast.dump — отладочный вывод дерева

```python
tree = ast.parse("x = [i for i in range(3)]")
# indent делает вывод многострочным и читаемым (Python 3.9+)
print(ast.dump(tree, indent=2, include_attributes=False))
```

### ast.walk — простой рекурсивный обход

```python
import ast

source = "import os, sys\nfrom collections import OrderedDict"
tree = ast.parse(source)

# Соберём все импортируемые имена модулей
imported = []
for node in ast.walk(tree):              # обходит ВСЕ узлы дерева
    if isinstance(node, ast.Import):
        for alias in node.names:
            imported.append(alias.name)
    elif isinstance(node, ast.ImportFrom):
        imported.append(node.module)

print(imported)                          # ['os', 'sys', 'collections']
```

### ast.NodeVisitor — паттерн «Посетитель»

`NodeVisitor` обходит дерево и вызывает метод `visit_<ИмяКласса>` для соответствующих узлов. Если метода нет — вызывается `generic_visit`, который рекурсивно обходит детей.

```python
import ast

class FunctionCollector(ast.NodeVisitor):
    """Собирает имена всех определённых функций и их аргументы."""

    def __init__(self):
        self.functions = []

    def visit_FunctionDef(self, node):
        args = [a.arg for a in node.args.args]
        self.functions.append((node.name, args, node.lineno))
        # ВАЖНО: чтобы зайти внутрь функции (вложенные def), нужно вызвать generic_visit
        self.generic_visit(node)

    # асинхронные функции — отдельный тип узла
    visit_AsyncFunctionDef = visit_FunctionDef


source = """
def outer(a, b):
    def inner(c):
        return c
    return inner

async def fetch(url):
    pass
"""

collector = FunctionCollector()
collector.visit(ast.parse(source))
for name, args, line in collector.functions:
    print(f"строка {line}: {name}({', '.join(args)})")
# строка 2: outer(a, b)
# строка 3: inner(c)
# строка 7: fetch(url)
```

### ast.NodeTransformer — изменение дерева

`NodeTransformer` похож на `NodeVisitor`, но метод `visit_*` должен **вернуть узел** (новый, тот же или `None` для удаления). Это позволяет переписывать код.

```python
import ast

class ConstantFolder(ast.NodeTransformer):
    """Сворачивает константные арифметические выражения: 2 + 3 -> 5."""

    def visit_BinOp(self, node):
        # сначала рекурсивно обрабатываем детей (вдруг там тоже свёртка)
        self.generic_visit(node)
        if (isinstance(node.left, ast.Constant)
                and isinstance(node.right, ast.Constant)
                and isinstance(node.op, (ast.Add, ast.Sub, ast.Mult))):
            left, right = node.left.value, node.right.value
            if isinstance(node.op, ast.Add):
                result = left + right
            elif isinstance(node.op, ast.Sub):
                result = left - right
            else:
                result = left * right
            return ast.copy_location(ast.Constant(value=result), node)
        return node


tree = ast.parse("y = 2 + 3 * 4", mode="exec")
new_tree = ConstantFolder().visit(tree)
ast.fix_missing_locations(new_tree)      # проставить lineno/col_offset новым узлам

# Скомпилируем и выполним изменённое дерево
code = compile(new_tree, filename="<ast>", mode="exec")
ns = {}
exec(code, ns)
print(ns["y"])                           # 14  (3*4=12, 2+12=14)
print(ast.dump(new_tree))                # видно Constant(value=14)
```

### ast.literal_eval — безопасная альтернатива eval

`literal_eval` вычисляет **только литералы и их безопасные комбинации**: строки, числа, кортежи, списки, словари, множества, булевы значения, `None`. Никаких вызовов функций, импортов, доступа к атрибутам — поэтому это безопасно для недоверенного ввода.

```python
import ast

# Безопасно: разбор строки в структуру данных
data = ast.literal_eval("{'name': 'Alice', 'tags': ['a', 'b'], 'age': 30}")
print(data["tags"])                      # ['a', 'b']

print(ast.literal_eval("(1, 2, 3)"))     # (1, 2, 3)
print(ast.literal_eval("0xFF"))          # 255

# Опасный код просто не выполнится — будет ValueError
try:
    ast.literal_eval("__import__('os').system('rm -rf /')")
except ValueError as e:
    print("Заблокировано:", e)           # malformed node or string

# Сравните с eval — НИКОГДА не используйте на недоверенных данных:
# eval("__import__('os').system('...')")   # выполнит произвольный код!
```

Чем `literal_eval` отличается от `json.loads`: понимает Python-синтаксис (одинарные кавычки, кортежи, `None`/`True`, множества), но НЕ парсит JSON-специфику (`true`/`null`). Для JSON используйте `json`, для Python-литералов — `literal_eval`.

### compile — компиляция дерева в байткод

```python
import ast

tree = ast.parse("result = sum(range(10))")
# AST можно скомпилировать так же, как строку
code_obj = compile(tree, filename="<generated>", mode="exec")

namespace = {}
exec(code_obj, namespace)
print(namespace["result"])               # 45
```

### Полезные вспомогательные функции

```python
import ast

tree = ast.parse('"""Docstring модуля."""\ndef f():\n    """doc f"""\n    pass')

# Извлечь докстринг (3.8+)
print(ast.get_docstring(tree))           # Docstring модуля.

# Восстановить исходный код из дерева (3.9+) — обратная операция к parse
print(ast.unparse(tree))

# Скопировать позицию из одного узла в другой при трансформациях
# ast.copy_location(new_node, old_node)
# Проставить недостающие позиции рекурсивно
# ast.fix_missing_locations(tree)

# Получить значение поля по умолчанию
node = ast.parse("x = 1").body[0]
print(ast.dump(node))
```

### Реальный пример: автоматическая инструментация (логирование вызовов)

```python
import ast

class CallLogger(ast.NodeTransformer):
    """Оборачивает каждый вызов f(...) в _log('f'); f(...)."""

    def visit_Call(self, node):
        self.generic_visit(node)
        if isinstance(node.func, ast.Name):
            log_stmt = ast.Expr(
                value=ast.Call(
                    func=ast.Name(id="print", ctx=ast.Load()),
                    args=[ast.Constant(value=f"вызов {node.func.id}")],
                    keywords=[],
                )
            )
            # вернуть выражение, последовательно выполняющее лог и вызов
            return ast.copy_location(
                ast.Tuple(elts=[log_stmt.value, node], ctx=ast.Load()), node
            ).elts[1]   # упрощённо: здесь обычно используют helper
        return node
```

## Частые вопросы на собеседовании

**Q1. На каком этапе обработки кода появляется AST?**
A: После токенизации и до компиляции в байткод. Парсер строит дерево из потока токенов, затем компилятор превращает AST в объект кода (байткод), который выполняет виртуальная машина.

**Q2. Чем отличается NodeVisitor от NodeTransformer?**
A: `NodeVisitor` обходит дерево и ничего не меняет — методы `visit_*` обычно ничего не возвращают (значение игнорируется). `NodeTransformer` предназначен для модификации: метод `visit_*` должен вернуть узел (новый/тот же — заменить, `None` — удалить, список — заменить на несколько). После трансформации нужно вызвать `fix_missing_locations`.

**Q3. Почему `ast.literal_eval` безопаснее `eval`?**
A: `eval` выполняет произвольный код, включая вызовы функций, импорты, доступ к файловой системе. `literal_eval` парсит строку в AST и допускает только узлы-литералы (числа, строки, списки, словари, кортежи, множества, `True`/`False`/`None`), отвергая всё остальное с `ValueError`. Вызовы и обращения к атрибутам невозможны.

**Q4. Зачем в методе visit_* вызывать self.generic_visit(node)?**
A: `generic_visit` рекурсивно обходит дочерние узлы. Если в своём `visit_X` его не вызвать, обход вглубь этого узла прекратится — например, не зайдёте во вложенные функции. По умолчанию (когда нет `visit_X`) `generic_visit` вызывается автоматически.

**Q5. Что такое поля `_fields` и `_attributes` у узла?**
A: `_fields` — имена смысловых дочерних полей (например, у `Assign` это `targets`, `value`). `_attributes` — служебные: `lineno`, `col_offset`, `end_lineno`, `end_col_offset` (позиция в исходнике).

**Q6. Как из AST получить обратно исходный код?**
A: Через `ast.unparse(tree)` (Python 3.9+). До 3.9 использовали сторонние библиотеки (`astor`). При этом форматирование/комментарии не сохраняются — генерируется эквивалентный код.

**Q7. Что вернёт `ast.parse` при разных значениях mode?**
A: `'exec'` → `Module`, `'eval'` → `Expression`, `'single'` → `Interactive`. Для `'eval'` исходник должен содержать ровно одно выражение.

**Q8. Можно ли скомпилировать и выполнить AST напрямую?**
A: Да: `compile(tree, filename, mode)` вернёт code object, который выполняется через `exec`/`eval`. Перед компиляцией модифицированного дерева обязательно `ast.fix_missing_locations(tree)`, иначе будет ошибка из-за отсутствующих `lineno`.

**Q9. Где AST применяется в реальных инструментах?**
A: `flake8`/`pylint` (анализ), `black` (форматирование на основе AST + токенов), `mypy` (вывод типов), `pytest` (переписывание `assert`), `bandit` (поиск уязвимостей), `coverage`, генераторы кода, ORM с DSL.

**Q10. В чём разница между ast.walk и NodeVisitor?**
A: `ast.walk` — плоский генератор всех узлов без гарантированного порядка и без удобного диспетчеринга. `NodeVisitor` даёт типизированные обработчики `visit_X` и контроль над тем, заходить ли вглубь конкретного узла. Для простого сбора — `walk`, для сложной логики — `NodeVisitor`.

**Q11. Почему `Constant` вместо `Num`/`Str`?**
A: В старых версиях литералы представлялись отдельными узлами `Num`, `Str`, `Bytes`, `NameConstant`. Начиная с 3.8 они объединены в `Constant` (старые признаны устаревшими). Различать тип теперь нужно по `type(node.value)`.

## Подводные камни (gotchas)

- **Забытый `fix_missing_locations`.** После создания новых узлов в `NodeTransformer` без проставленных `lineno`/`col_offset` `compile` упадёт. Используйте `ast.copy_location` или `ast.fix_missing_locations`.
- **Не вызван `generic_visit`.** Обход «застревает» на верхнем узле — вложенные узлы не посещаются.
- **`NodeVisitor` не меняет дерево.** Если в `NodeVisitor` вернуть узел, он игнорируется. Для модификации нужен `NodeTransformer`.
- **`literal_eval` не равен `json.loads`.** Он не примет `true`/`null` (JSON), но примет `True`/`None` (Python). И наоборот.
- **AST зависит от версии Python.** Набор и структура узлов меняются между релизами (walrus `:=`, `match`, позиционные параметры). Код анализа нужно тестировать на целевых версиях.
- **`unparse` теряет форматирование.** Комментарии, конкретные кавычки, пробелы не восстанавливаются — только семантически эквивалентный код.
- **Бесконечная рекурсия в трансформере.** Если `visit_X` возвращает узел того же типа и снова вызывает `visit`, можно зациклиться. Обычно `generic_visit` вызывают до построения нового узла.
- **`literal_eval` всё же может «съесть» память/стек** на очень глубоко вложенных структурах (DoS), хотя произвольный код не выполнит.

## Лучшие практики

- Для безопасного разбора пользовательских литералов — всегда `ast.literal_eval`, никогда `eval`.
- В `NodeVisitor`/`NodeTransformer` не забывайте `self.generic_visit(node)` там, где нужен обход вглубь.
- После трансформаций — `ast.fix_missing_locations(tree)` перед `compile`.
- Для отладки дерева — `ast.dump(tree, indent=2)`.
- Проверяйте тип узла через `isinstance`, а константы — через `isinstance(node.value, ...)`.
- Учитывайте версии Python: используйте `ast.parse(..., feature_version=...)` при необходимости и тестируйте на нужных интерпретаторах.
- Для извлечения метаданных (импорты, докстринги) предпочитайте AST выполнению кода — безопаснее и не имеет побочных эффектов.

## Шпаргалка

```python
import ast

# Парсинг
tree = ast.parse(src)                      # mode='exec' -> Module
ast.parse(expr, mode="eval")               # -> Expression

# Просмотр
ast.dump(tree, indent=2)                   # читаемое дерево
ast.unparse(tree)                          # дерево -> исходный код (3.9+)
ast.get_docstring(node)                    # докстринг модуля/функции/класса

# Обход
for n in ast.walk(tree): ...               # все узлы

class V(ast.NodeVisitor):
    def visit_FunctionDef(self, n):
        self.generic_visit(n)              # зайти вглубь

class T(ast.NodeTransformer):
    def visit_BinOp(self, n):
        self.generic_visit(n)
        return new_node                    # заменить узел
ast.fix_missing_locations(new_tree)        # перед compile

# Безопасное вычисление литералов
ast.literal_eval("{'a': [1, 2]}")          # только литералы, не eval!

# Компиляция и выполнение
code = compile(tree, "<f>", "exec")
exec(code, namespace)

# Узлы: Module, FunctionDef, Assign, Call, Name, Constant, BinOp,
#       If, For, While, Return, Import, ImportFrom, Attribute, Compare
# Атрибуты позиции: lineno, col_offset, end_lineno, end_col_offset
```
