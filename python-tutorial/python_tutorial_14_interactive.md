# Глава 14. Interactive Input Editing and History Substitution — конспект и вопросы

## О чём глава

Глава посвящена удобствам работы в **интерактивном режиме** интерпретатора Python (REPL): редактированию вводимой строки, истории команд, автодополнению по Tab. Большинство этих возможностей предоставляет библиотека **GNU Readline** (или совместимая) через модули `readline` и `rlcompleter`. Также рассматривается настройка стартового скрипта `PYTHONSTARTUP` и альтернативные интерактивные оболочки (IPython, bpython, ptpython).

Для собеседования тема не топовая, но знание про автодополнение, историю, `PYTHONSTARTUP` и IPython показывает зрелость и удобство в повседневной работе.

---

## Ключевые концепции (с примерами)

### 1. Редактирование строки (line editing)

В современном интерпретаторе Python 3.13+ появился новый интерактивный REPL (на чистом Python), но классически возможности обеспечивает GNU Readline. Доступны привычные сочетания клавиш в стиле Emacs:

```text
Ctrl+A      перейти в начало строки
Ctrl+E      перейти в конец строки
Ctrl+K      удалить от курсора до конца строки (kill)
Ctrl+U      удалить от курсора до начала строки
Ctrl+W      удалить слово слева от курсора
Ctrl+Y      вставить (yank) ранее удалённый текст
Ctrl+L      очистить экран
Ctrl+R      обратный поиск по истории команд
Стрелки     перемещение и навигация по истории
```

### 2. История команд (history substitution)

Введённые строки сохраняются в буфере истории. Стрелки вверх/вниз листают историю; `Ctrl+R` ищет по подстроке. Историю можно сохранять между сессиями в файл.

```python
# В стартовом скрипте (PYTHONSTARTUP) можно сохранять историю на диск
import atexit
import os
import readline

histfile = os.path.join(os.path.expanduser("~"), ".python_history")
try:
    readline.read_history_file(histfile)
    readline.set_history_length(1000)
except FileNotFoundError:
    pass

atexit.register(readline.write_history_file, histfile)
# => история команд переживёт перезапуск интерпретатора
```

> Примечание: начиная с Python 3.4, интерпретатор по умолчанию сам сохраняет историю в `~/.python_history` и включает автодополнение — без отдельного скрипта.

### 3. Автодополнение по Tab (модуль `rlcompleter`)

```python
import rlcompleter
import readline
readline.parse_and_bind("tab: complete")

# Теперь в REPL:
# >>> import os
# >>> os.geten<Tab>   ->   os.getenv
# >>> "".upp<Tab>     ->   "".upper(
```

Дополняются имена переменных, модулей, атрибутов объекта (через `dir()`) и ключевые слова. По одному Tab — если вариант один, он подставляется; по двум Tab — показывается список вариантов.

---

## Подробный разбор по темам

### Модуль `readline`

`readline` — обёртка над библиотекой GNU Readline. Через него управляют:

```python
import readline

readline.parse_and_bind("tab: complete")   # привязка клавиш
readline.set_history_length(1000)           # длина истории
readline.get_current_history_length()       # сколько строк сейчас
readline.read_history_file(path)            # загрузить историю
readline.write_history_file(path)           # сохранить историю
readline.add_history("текст")               # добавить строку вручную
readline.set_completer(func)                # своя функция дополнения
```

Если Readline недоступна (например, на чистой Windows без обёрток), импорт `readline` упадёт с `ImportError` — поэтому в стартовых скриптах его оборачивают в `try/except`.

### Модуль `rlcompleter`

Предоставляет класс `Completer`, реализующий дополнение имён Python для Readline. Достаточно его импортировать и привязать Tab:

```python
import rlcompleter, readline
readline.parse_and_bind("tab: complete")
```

Можно написать собственный completer и установить его через `readline.set_completer()`.

### Стартовый скрипт PYTHONSTARTUP

Переменная окружения `PYTHONSTARTUP` указывает на файл с Python-кодом, который выполняется **перед** показом первого приглашения `>>>` в интерактивном режиме. Это удобно для:

- настройки истории и автодополнения;
- предимпорта часто используемых модулей;
- определения вспомогательных функций.

```bash
# В ~/.bashrc или ~/.zshrc
export PYTHONSTARTUP="$HOME/.pythonrc.py"
```

```python
# ~/.pythonrc.py — выполняется при старте интерактивного Python
import os, sys, json          # часто нужные модули под рукой
print("Готов к работе:", sys.version.split()[0])
# => эти импорты доступны сразу в REPL без повторного import
```

Важно: `PYTHONSTARTUP` работает **только в интерактивном режиме**, не при запуске скриптов (`python script.py`).

### Альтернативные оболочки

| Оболочка | Чем хороша |
|----------|------------|
| **IPython** | Magic-команды (`%timeit`, `%run`, `!ls`), богатое автодополнение, `?`/`??` для справки, история по `_`, `__`, `___`, основа Jupyter |
| **bpython** | Лёгкая, подсветка синтаксиса, инлайн-подсказки сигнатур, автодополнение «на лету» |
| **ptpython** | На базе `prompt_toolkit`: многострочное редактирование, подсветка, конфигурируемость |

```python
# IPython: измерение времени одной строки
%timeit sum(range(1000))
# => 12.3 µs ± 0.2 µs per loop

# Справка по объекту
len?       # короткая справка
str.split??  # исходник/подробности
```

---

## Частые вопросы на собеседовании (Q/A)

**Q: Что делает переменная PYTHONSTARTUP?**
A: Указывает путь к Python-файлу, выполняемому при старте интерактивного интерпретатора (до первого `>>>`). Используется для настройки истории, автодополнения, предимпорта модулей. В неинтерактивном режиме игнорируется.

**Q: Какие модули отвечают за автодополнение и историю в REPL?**
A: `readline` (редактирование строки, история, привязка клавиш) и `rlcompleter` (логика дополнения имён Python). Связка `import rlcompleter, readline; readline.parse_and_bind("tab: complete")`.

**Q: Сохраняется ли история команд между сессиями?**
A: Да, начиная с Python 3.4 интерпретатор по умолчанию пишет историю в `~/.python_history`. Можно настроить вручную через `readline.read_history_file` / `write_history_file` + `atexit`.

**Q: Чем IPython лучше стандартного REPL?**
A: Magic-команды (`%timeit`, `%run`), shell-команды через `!`, мощное автодополнение, `?`/`??` для интроспекции, переменные результатов `_`, удобная история, цветной вывод. Основа для Jupyter.

**Q: Что произойдёт, если readline недоступна?**
A: `import readline` бросит `ImportError`; редактирование/история/Tab-дополнение работать не будут (или ограниченно). Поэтому код в стартовых скриптах оборачивают в `try/except ImportError`.

---

## Подводные камни (gotchas)

- **PYTHONSTARTUP не для скриптов.** Код стартового файла не выполняется при `python my_script.py` — только в интерактивном режиме. Не кладите туда бизнес-логику.
- **readline может отсутствовать.** На «голой» Windows стандартного `readline` нет (исторически использовали `pyreadline`). Всегда оборачивайте импорт в `try/except`.
- **Tab-дополнение и отступы конфликтуют.** Если привязать Tab к дополнению, вставка табов в многострочный ввод может стать менее удобной.
- **Магия IPython — не Python.** `%timeit`, `!ls`, `_` работают только в IPython/Jupyter; в обычном скрипте это синтаксические ошибки.
- **Новый REPL в 3.13.** В Python 3.13 появился новый интерактивный интерпретатор; поведение и горячие клавиши могут немного отличаться от классического readline-REPL.

---

## Шпаргалка

```python
# Включить историю + автодополнение вручную (для PYTHONSTARTUP)
import atexit, os, readline, rlcompleter

readline.parse_and_bind("tab: complete")      # Tab-дополнение
histfile = os.path.expanduser("~/.python_history")
try:
    readline.read_history_file(histfile)
except FileNotFoundError:
    pass
readline.set_history_length(1000)
atexit.register(readline.write_history_file, histfile)
```

```bash
export PYTHONSTARTUP="$HOME/.pythonrc.py"   # активировать стартовый скрипт
pip install ipython bpython ptpython         # удобные альтернативные REPL
```

```text
Ctrl+A / Ctrl+E   начало / конец строки
Ctrl+R            поиск по истории
Стрелки ↑ ↓       листание истории
Tab               автодополнение (rlcompleter)
```

**Главное:** `readline` + `rlcompleter` дают редактирование, историю и Tab-дополнение; `PYTHONSTARTUP` — настройка REPL только в интерактивном режиме; IPython/bpython/ptpython — мощные альтернативы стандартной оболочке.
