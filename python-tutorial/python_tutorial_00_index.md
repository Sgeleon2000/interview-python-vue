# Python Tutorial — оглавление конспектов для подготовки к собеседованию

Учебные конспекты по официальному [Python Tutorial](https://docs.python.org/3/tutorial/index.html) на русском языке. Каждый файл — конспект главы с ключевыми концепциями, разбором тем, вопросами Q/A для собеседования, подводными камнями (gotchas) и шпаргалкой.

---

## Таблица: глава → файл

| № | Глава (оригинал) | Файл | О чём |
|---|------------------|------|-------|
| 1 | Whetting Your Appetite | [python_tutorial_01_appetite.md](./python_tutorial_01_appetite.md) | Зачем нужен Python, его ниша, сильные стороны, сравнение с другими языками |
| 2 | Using the Python Interpreter | [python_tutorial_02_interpreter.md](./python_tutorial_02_interpreter.md) | Запуск интерпретатора, аргументы, кодировка, интерактивный режим |
| 3 | An Informal Introduction to Python | [python_tutorial_03_intro.md](./python_tutorial_03_intro.md) | Числа, строки, списки, первые программы, базовый синтаксис |
| 4 | More Control Flow Tools | [python_tutorial_04_control_flow.md](./python_tutorial_04_control_flow.md) | if/for/while, range, break/continue/else, функции, аргументы, match |
| 5 | Data Structures | [python_tutorial_05_data_structures.md](./python_tutorial_05_data_structures.md) | list, tuple, set, dict, comprehensions, методы коллекций |
| 6 | Modules | [python_tutorial_06_modules.md](./python_tutorial_06_modules.md) | import, пакеты, `__name__`, путь поиска, `__all__` |
| 7 | Input and Output | [python_tutorial_07_input_output.md](./python_tutorial_07_input_output.md) | f-строки, format, чтение/запись файлов, JSON, контекстные менеджеры |
| 8 | Errors and Exceptions | [python_tutorial_08_errors.md](./python_tutorial_08_errors.md) | try/except/else/finally, raise, свои исключения, цепочки, groups |
| 9 | Classes | [python_tutorial_09_classes.md](./python_tutorial_09_classes.md) | ООП, наследование, области видимости, итераторы, генераторы |
| 10 | Brief Tour of the Standard Library | [python_tutorial_10_stdlib.md](./python_tutorial_10_stdlib.md) | os, glob, sys, re, math, urllib, datetime и др. |
| 11 | Standard Library — Part II | [python_tutorial_11_stdlib_part2.md](./python_tutorial_11_stdlib_part2.md) | форматирование, шаблоны, логирование, weakref, очереди, decimal |
| 12 | Virtual Environments and Packages | [python_tutorial_12_venv.md](./python_tutorial_12_venv.md) | venv, pip, requirements.txt, изоляция зависимостей |
| 14 | Interactive Input Editing and History | [python_tutorial_14_interactive.md](./python_tutorial_14_interactive.md) | readline, rlcompleter, Tab-дополнение, PYTHONSTARTUP, IPython |
| 15 | Floating-Point Arithmetic | [python_tutorial_15_floating_point.md](./python_tutorial_15_floating_point.md) | IEEE 754, 0.1+0.2, round, isclose, Decimal/Fraction, nan/inf |

> Глава 13 «What Now?» в оригинале — это лишь список ресурсов для дальнейшего чтения, отдельного конспекта не требует.

---

## Краткое описание каждой главы

- **01. Appetite** — мотивационная глава: место Python среди языков, где он силён (скрипты, веб, данные, автоматизация), почему его выбирают.
- **02. Interpreter** — как запускать Python: интерактивно и из файла, аргументы командной строки, кодировка исходников, переменные окружения.
- **03. Intro** — первое знакомство: арифметика, типы чисел, строки и их методы/срезы, списки. База, на которой строится всё остальное.
- **04. Control Flow** — управление потоком: условия, циклы, `range`, `break/continue/else`, определение функций, виды аргументов (позиционные, именованные, `*args`/`**kwargs`), `match`-конструкция.
- **05. Data Structures** — коллекции: списки (стек/очередь), кортежи, множества, словари, генераторы коллекций (comprehensions), вложенные структуры. Топовая тема для собеседований.
- **06. Modules** — модульность: импорт, пакеты, `if __name__ == "__main__"`, путь поиска модулей, компиляция в `.pyc`.
- **07. Input/Output** — форматирование вывода (f-строки, `str.format`, `repr` vs `str`), работа с файлами, `with`, сериализация в JSON.
- **08. Errors** — исключения: иерархия, `try/except/else/finally`, `raise`, свои классы исключений, цепочки (`raise from`), `ExceptionGroup`. Важно для собеседований.
- **09. Classes** — ООП в Python: классы и экземпляры, атрибуты, наследование (в т. ч. множественное), области видимости (LEGB), итераторы и генераторы. Топовая тема.
- **10. Stdlib** — обзор полезных модулей стандартной библиотеки: ОС, файлы, регулярки, математика, сеть, даты.
- **11. Stdlib Part II** — продвинутая стандартная библиотека: шаблоны, логирование, слабые ссылки, инструменты для производительности.
- **12. Venv** — виртуальные окружения и управление пакетами через pip; изоляция зависимостей проектов.
- **14. Interactive** — удобства REPL: редактирование строк, история, автодополнение, настройка через PYTHONSTARTUP, альтернативные оболочки.
- **15. Floating Point** — арифметика с плавающей точкой, её ограничения и корректная работа с ней. **Частый вопрос на собеседовании.**

---

## Рекомендованный порядок изучения для собеседования

### Этап 1. Фундамент (обязательно знать назубок)
1. **03. Intro** — типы, строки, срезы.
2. **05. Data Structures** — list/tuple/set/dict, comprehensions. ⭐ Спрашивают почти всегда.
3. **04. Control Flow** — функции, виды аргументов, `match`.
4. **09. Classes** — ООП, наследование, генераторы/итераторы. ⭐ Топ-тема.

### Этап 2. Практика и надёжность
5. **08. Errors** — исключения и их обработка. ⭐ Часто спрашивают.
6. **07. Input/Output** — f-строки, файлы, `with`, JSON.
7. **06. Modules** — импорты, пакеты, `__name__`.

### Этап 3. Тонкости, отличающие senior
8. **15. Floating Point** — `0.1+0.2`, `round`, `Decimal`, `nan`. ⭐ Классический «каверзный» вопрос.
9. **10 + 11. Stdlib** — что есть «из коробки», когда не изобретать велосипед.
10. **12. Venv** — окружения, pip (важно для практических задач/онбординга).

### Этап 4. Контекст и удобства
11. **02. Interpreter** и **14. Interactive** — запуск, REPL, инструментарий.
12. **01. Appetite** — общая эрудиция, «почему Python».

> ⭐ — темы с высокой частотой на собеседованиях. Если времени мало: 03 → 05 → 09 → 08 → 15.

---

## Как пользоваться конспектами

Каждый файл-глава имеет единую структуру:

```
# Глава N. Название — конспект и вопросы
## О чём глава
## Ключевые концепции (с примерами кода и выводом # =>)
## Подробный разбор по темам
## Частые вопросы на собеседовании (Q/A)
## Подводные камни (gotchas)
## Шпаргалка
```

Перед собеседованием можно быстро повторить, читая только разделы **«Частые вопросы (Q/A)»**, **«Подводные камни»** и **«Шпаргалка»** в каждом файле.
