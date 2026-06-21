# argparse — подготовка к собеседованию

## Что это и зачем

`argparse` — стандартный модуль Python для разбора аргументов командной строки (CLI). Он входит в стандартную библиотеку (доступен с Python 3.2, заменил устаревший `optparse`), поэтому не требует установки зависимостей.

Зачем он нужен:

- **Парсинг `sys.argv`**. Вместо ручного разбора `sys.argv[1:]` вы декларативно описываете аргументы, а модуль сам разбирает строку, проверяет типы и количество значений.
- **Автоматическая генерация справки**. Флаги `-h/--help`, корректное форматирование, секции usage — всё генерируется автоматически.
- **Валидация и приведение типов**. `type=int`, `choices`, `required`, кастомные валидаторы — ошибки пользователя отлавливаются до того, как попадут в бизнес-логику.
- **Понятные сообщения об ошибках**. При некорректном вводе argparse печатает usage, текст ошибки и завершает процесс с кодом возврата 2.
- **Подкоманды**. Можно строить интерфейсы в стиле `git commit`, `git push`, `docker run` через subparsers.

Когда выбрать что-то другое: для крупных CLI с богатым UX часто берут `click` или `typer` (декларативность, цвета, автодополнение), но на собеседовании ожидают знания именно `argparse` как базового инструмента «из коробки».

```python
import argparse

# Минимальный пример: одна программа, один аргумент
parser = argparse.ArgumentParser(description="Пример простого CLI")
parser.add_argument("name", help="Имя пользователя")        # позиционный аргумент
parser.add_argument("--greeting", default="Привет")          # опциональный аргумент

args = parser.parse_args()                                    # читает sys.argv[1:]
print(f"{args.greeting}, {args.name}!")
# Запуск: python app.py Андрей --greeting "Здравствуй"
# Вывод: Здравствуй, Андрей!
```

## Ключевые концепции

### Позиционные vs опциональные аргументы

- **Позиционные** аргументы задаются без префикса (`name`, `path`). Они обязательны (если не управлять через `nargs`), а их порядок важен. Имя позиционного аргумента совпадает с именем атрибута в `Namespace`.
- **Опциональные** аргументы начинаются с `-` (короткая форма) или `--` (длинная форма): `-v`, `--verbose`. По умолчанию они необязательны. Имя атрибута выводится из длинной формы (`--max-count` → `args.max_count`, дефис превращается в подчёркивание).

```python
parser = argparse.ArgumentParser()
parser.add_argument("src")                       # позиционный, обязательный
parser.add_argument("dst")                       # позиционный, обязательный
parser.add_argument("-f", "--force",             # опциональный флаг
                    action="store_true")
parser.add_argument("--max-count", type=int)     # → args.max_count
```

### Namespace

`parse_args()` возвращает объект `argparse.Namespace` — простой контейнер атрибутов. Доступ через точку (`args.src`). Можно превратить в словарь: `vars(args)`.

### Жизненный цикл

1. Создаём `ArgumentParser`.
2. Регистрируем аргументы через `add_argument`.
3. Вызываем `parse_args()` — модуль читает `sys.argv`, приводит типы, валидирует.
4. При ошибке вызывается `parser.error()`, который печатает usage в `stderr` и завершает процесс с кодом 2.

## Основные функции/классы/методы

### ArgumentParser

Конструктор парсера и точка входа в API.

```python
import argparse

parser = argparse.ArgumentParser(
    prog="mytool",                              # имя программы в usage (по умолчанию sys.argv[0])
    description="Описание над списком аргументов",
    epilog="Текст в самом низу справки (например, примеры использования)",
    formatter_class=argparse.RawDescriptionHelpFormatter,  # как форматировать description/epilog
    add_help=True,                              # добавлять ли автоматически -h/--help
    allow_abbrev=True,                          # разрешать сокращения длинных опций (--verb → --verbose)
    fromfile_prefix_chars="@",                  # читать аргументы из файла @args.txt
    argument_default=None,                      # дефолт для ВСЕХ аргументов сразу
)
```

Полезные `formatter_class`:

- `argparse.HelpFormatter` — по умолчанию; «схлопывает» переносы строк в description/epilog.
- `argparse.RawDescriptionHelpFormatter` — сохраняет переносы строк в description/epilog (но не в help аргументов).
- `argparse.RawTextHelpFormatter` — сохраняет переносы строк везде, включая help каждого аргумента.
- `argparse.ArgumentDefaultsHelpFormatter` — автоматически дописывает `(default: ...)` к каждому help.
- `argparse.MetavarTypeHelpFormatter` — использует имя типа как metavar.

### add_argument

Главный метод. Самые важные параметры:

```python
parser.add_argument(
    "-n", "--name",          # имя(имена) флага или имя позиционного аргумента
    action="store",          # что делать со значением (см. ниже)
    nargs=None,              # сколько значений ожидать (см. ниже)
    const=None,              # значение для store_const/append_const и nargs='?'
    default=None,            # значение, если аргумент не передан
    type=str,                # функция приведения типа / валидатор
    choices=None,            # допустимые значения (список/range/любой контейнер)
    required=False,          # обязателен ли опциональный аргумент
    help="текст справки",
    metavar="NAME",          # как аргумент отображается в usage/help
    dest="name",             # имя атрибута в Namespace
)
```

### Типы аргументов (`type`)

`type` — это любой вызываемый объект, принимающий строку и возвращающий значение. Если он бросает `ValueError` или `TypeError`, argparse покажет понятную ошибку.

```python
parser.add_argument("--count", type=int)        # "10" → 10
parser.add_argument("--ratio", type=float)      # "0.5" → 0.5
parser.add_argument("--path", type=pathlib.Path)# "/tmp" → Path('/tmp')

# Кастомный тип = обычная функция
def positive_int(value: str) -> int:
    ivalue = int(value)
    if ivalue <= 0:
        # argparse перехватит и красиво выведет
        raise argparse.ArgumentTypeError(f"{value!r} должно быть > 0")
    return ivalue

parser.add_argument("--workers", type=positive_int)

# Открытие файла через встроенный FileType
parser.add_argument("--log", type=argparse.FileType("w", encoding="utf-8"))
```

> Важно: `type` НЕ применяется к значению `default`, если оно уже не строка. Если `default="10"` (строка) и аргумент опущен — `type` к нему применится; если `default=10` (int) — нет.

### nargs — количество значений

```python
parser.add_argument("--coords", nargs=2, type=float)   # ровно 2 значения → [x, y]
parser.add_argument("--name", nargs="?", const="C",     # 0 или 1; если флаг без значения → const,
                    default="D")                         #          если флаг вообще не передан → default
parser.add_argument("files", nargs="*")                 # 0 или больше → список (даже пустой)
parser.add_argument("files", nargs="+")                 # 1 или больше → список, иначе ошибка
parser.add_argument("cmd", nargs=argparse.REMAINDER)    # ВСЁ оставшееся как есть (включая флаги)
```

Семантика `nargs`:

- `N` (число) — ровно N значений, всегда список.
- `'?'` — ноль или одно значение. Тонкость: для опционального аргумента `const` используется когда флаг указан БЕЗ значения; `default` — когда флаг вообще не указан.
- `'*'` — любое число значений (список, возможно пустой).
- `'+'` — хотя бы одно значение, иначе ошибка.
- `argparse.REMAINDER` — забирает все оставшиеся аргументы дословно, не пытаясь интерпретировать `-`/`--`. Удобно для проброса (`mytool exec -- ls -la`). Считается «хрупким», в новом коде иногда заменяют на `nargs='*'` после `--`.

### action — действия

```python
parser.add_argument("--verbose", action="store_true")      # флаг → True/False (default False)
parser.add_argument("--quiet", action="store_false")       # флаг → False/True (default True)
parser.add_argument("--mode", action="store_const",        # хранит const, если флаг указан
                    const="fast", default="slow")
parser.add_argument("--tag", action="append")              # накапливает значения в список: --tag a --tag b → ['a','b']
parser.add_argument("--feat", action="append_const",       # добавляет const в общий dest
                    const="x", dest="features")
parser.add_argument("-v", action="count", default=0)       # -vvv → 3 (счётчик)
parser.add_argument("--version", action="version",         # печатает версию и выходит
                    version="%(prog)s 2.1.0")
parser.add_argument("--ext", action="extend", nargs="+")   # extend (3.8+): расширяет список
```

- `store` (по умолчанию) — просто сохранить значение.
- `store_true`/`store_false` — булевы флаги, не требуют значения.
- `store_const` — сохранить фиксированный `const`.
- `append` — добавлять каждое вхождение в список.
- `append_const` — добавлять `const` в общий `dest` (несколько флагов пишут в один список).
- `count` — считать число вхождений (классический `-v -v` или `-vv`).
- `version` — спецдействие: печатает `version` и завершает процесс.

### choices, metavar, dest

```python
parser.add_argument("--level",
                    choices=["debug", "info", "warning", "error"])  # ограничение значений
parser.add_argument("--port", type=int, choices=range(1, 65536))    # choices может быть range
parser.add_argument("--input", metavar="ФАЙЛ")                       # имя в usage/help
parser.add_argument("-x", dest="x_axis")                            # переопределить имя атрибута
```

- `choices` — любой контейнер, поддерживающий `in`. Проверка происходит ПОСЛЕ применения `type`.
- `metavar` — как значение отображается в справке (косметика, не влияет на `dest`).
- `dest` — под каким именем результат попадёт в `Namespace`.

### parse_args vs parse_known_args

```python
args = parser.parse_args()                      # неизвестный аргумент → ошибка и выход
args, unknown = parser.parse_known_args()       # неизвестные складываются в список unknown
# unknown полезен для проброса аргументов в другой инструмент/фреймворк
```

`parse_args(args=...)` принимает список вместо `sys.argv` — удобно для тестов:

```python
args = parser.parse_args(["--name", "test", "--count", "5"])
```

### Группы аргументов

```python
# Группа только для красивого отображения в справке
group = parser.add_argument_group("Настройки сети", "Параметры подключения")
group.add_argument("--host", default="localhost")
group.add_argument("--port", type=int, default=8080)

# Взаимоисключающая группа: можно указать максимум один из аргументов
mx = parser.add_mutually_exclusive_group(required=False)
mx.add_argument("--json", action="store_true")
mx.add_argument("--yaml", action="store_true")
# Если required=True — ровно один из них обязателен
```

### subparsers — подкоманды

```python
parser = argparse.ArgumentParser(prog="git")
subparsers = parser.add_subparsers(
    dest="command",          # куда положить имя выбранной команды
    required=True,           # требовать указания команды (3.7+)
    title="команды",
    metavar="COMMAND",
    help="доступные команды",
)

p_commit = subparsers.add_parser("commit", help="Зафиксировать изменения")
p_commit.add_argument("-m", "--message", required=True)
p_commit.set_defaults(func=do_commit)   # привязка обработчика к команде

p_push = subparsers.add_parser("push", help="Отправить на сервер")
p_push.add_argument("remote", nargs="?", default="origin")
p_push.set_defaults(func=do_push)
```

### fromfile_prefix_chars — аргументы из файла

```python
parser = argparse.ArgumentParser(fromfile_prefix_chars="@")
# Содержимое args.txt (по одному аргументу на строку):
#   --name
#   Андрей
#   --count
#   5
args = parser.parse_args(["@args.txt"])   # подставит аргументы из файла
```

## Частые вопросы на собеседовании

**Вопрос:** Чем `argparse` отличается от ручного разбора `sys.argv` и от `optparse`?

**Ответ:** `sys.argv` даёт только сырой список строк — всю валидацию, приведение типов, генерацию справки и обработку ошибок пришлось бы писать вручную. `argparse` всё это делает декларативно. `optparse` — устаревший предшественник: он не поддерживает позиционные аргументы и подкоманды и официально объявлен deprecated; начиная с Python 3.2 рекомендуют `argparse`.

---

**Вопрос:** Как сделать аргумент обязательным? В чём разница для позиционных и опциональных?

**Ответ:** Позиционные аргументы обязательны по умолчанию (если `nargs` не делает их опциональными через `?`/`*`). Опциональные (`--foo`) по умолчанию необязательны; чтобы сделать обязательным, нужно `required=True`. Считается плохой практикой делать `--`-аргумент обязательным (это противоречит ожиданиям «опциональности»), но иногда оправдано ради читаемости usage.

---

**Вопрос:** В чём разница между `store_true`, `store_const` и `count`?

**Ответ:** `store_true` — частный случай булевого флага (по умолчанию `False`, при указании `True`). `store_const` хранит произвольную константу `const` при указании флага. `count` считает число вхождений флага (`-vvv` → 3), что удобно для уровней детализации логов. Аналогично есть `store_false` (по умолчанию `True`).

---

**Вопрос:** Объясните разницу `nargs='?'`, `'*'`, `'+'` и роль `const`/`default` при `'?'`.

**Ответ:** `'?'` — ноль или одно значение; `'*'` — любое число (включая ноль), результат всегда список; `'+'` — минимум одно, иначе ошибка. При `'?'` для опционального аргумента действует тонкая логика: если флаг вообще не передан — берётся `default`; если флаг передан без значения — берётся `const`; если передан со значением — берётся само значение. Это используют, например, для `--color` (без значения = `auto`, отсутствует = `never`).

---

**Вопрос:** Как реализовать кастомную валидацию значения?

**Ответ:** Через `type` передать функцию, принимающую строку и возвращающую преобразованное значение. При невалидном вводе функция должна бросить `argparse.ArgumentTypeError` (или `ValueError`/`TypeError`), и argparse сам выведет аккуратное сообщение и usage. `choices` подходит для перечислимых множеств, но для диапазонов/форматов лучше кастомный `type`. Важно: `choices` проверяется уже после `type`.

---

**Вопрос:** Как организовать подкоманды в стиле `git`? Как вызвать нужный обработчик?

**Ответ:** Через `parser.add_subparsers()`, затем `subparsers.add_parser("name")` для каждой команды со своими аргументами. Распространённый приём — `subparser.set_defaults(func=handler)`, тогда после `args = parser.parse_args()` достаточно вызвать `args.func(args)`. Параметр `dest="command"` сохранит имя выбранной команды, а `required=True` сделает указание команды обязательным.

---

**Вопрос:** Зачем нужен `parse_known_args` и чем он отличается от `parse_args`?

**Ответ:** `parse_args` при встрече неизвестного аргумента печатает ошибку и завершает процесс. `parse_known_args` возвращает кортеж `(namespace, unknown_list)`: распознанные кладёт в `Namespace`, нераспознанные — в список. Это полезно, когда часть аргументов нужно пробросить дальше (в другой парсер, в субпроцесс, во фреймворк вроде PyTorch/Lightning).

---

**Вопрос:** Применяется ли `type` к `default`? Что с этим важно помнить?

**Ответ:** `type` применяется к `default` только если `default` — строка. Если задать `default=10` (int), приведение типа не сработает (значение пройдёт как есть). Поэтому, если используете `type` с кастомной валидацией, либо задавайте `default` уже готовым значением правильного типа, либо учитывайте, что строковый `default` пройдёт через `type`.

---

**Вопрос:** Как протестировать парсер аргументов в unit-тестах?

**Ответ:** Передавать список строк напрямую: `parser.parse_args(["--count", "5"])` вместо чтения `sys.argv`. Для проверки ошибок ловить `SystemExit` (argparse при ошибке вызывает `sys.exit(2)`): `with pytest.raises(SystemExit)`. Можно вынести создание парсера в функцию `build_parser()` и тестировать её изолированно.

---

**Вопрос:** Что такое взаимоисключающие группы и где они полезны?

**Ответ:** `add_mutually_exclusive_group()` гарантирует, что из набора аргументов пользователь укажет не более одного (а при `required=True` — ровно один). Пример: `--verbose`/`--quiet` или выбор формата вывода `--json`/`--yaml`/`--xml`. Ограничение: внутри такой группы нельзя иметь обязательные позиционные аргументы, и логика «один из нескольких наборов» (сложные зависимости) ею не выражается — для этого нужна ручная проверка через `parser.error()`.

---

**Вопрос:** Чем `metavar` отличается от `dest`?

**Ответ:** `dest` — это имя атрибута в `Namespace`, под которым доступно значение в коде (`args.dest`). `metavar` — чисто косметика: как аргумент выглядит в тексте usage/help. Они независимы: можно показать `--input ФАЙЛ` (`metavar="ФАЙЛ"`), но в коде обращаться через `args.input`.

---

**Вопрос:** Как обработать аргументы с дефисами в длинном имени, например `--max-count`?

**Ответ:** argparse автоматически конвертирует дефисы в подчёркивания при формировании `dest`: `--max-count` доступен как `args.max_count`. Если нужно иное имя — задайте `dest` явно.

## Подводные камни (gotchas)

- **`type` и `default`**: `type` применяется к `default` только если он строка. Не полагайтесь на приведение нестрокового дефолта.
- **`nargs='*'`/`'+'` всегда дают список**, даже при одном элементе. А вот без `nargs` (одиночное значение) вы получите скаляр, не список — частая причина багов при смене `nargs`.
- **`choices` проверяется после `type`**: для `type=int` указывайте `choices=[1, 2, 3]` (числа), а не `["1", "2", "3"]` (строки), иначе всё всегда будет невалидно.
- **`allow_abbrev=True` по умолчанию**: пользователь может писать `--verb` вместо `--verbose`. Это может сломаться при добавлении новой опции с похожим префиксом (`--version`) — конфликт. В стабильных CLI лучше `allow_abbrev=False`.
- **`store_true` нельзя комбинировать с `type` или `nargs`** — это флаг без значения; попытка приведёт к ошибке конфигурации.
- **`argparse.REMAINDER` хрупок**: он не всегда корректно работает в комбинации с опциональными аргументами, идущими до него, и может «съесть» флаги, предназначенные основному парсеру. По возможности используйте `--` как разделитель и `nargs='*'`.
- **`-h`/`--help` добавляются автоматически**: если вам нужен свой `-h`, отключите `add_help=False`.
- **Дефис в имени → подчёркивание в `dest`**: `--dry-run` это `args.dry_run`.
- **subparsers и `required`**: до Python 3.7 подкоманды по умолчанию НЕ были обязательны, что приводило к тихому падению при `args.func`. Указывайте `required=True` (и/или `dest`) явно и проверяйте, что команда задана.
- **Изменяемый `default`**: не используйте `default=[]`. argparse не делает копию; в сочетании с `action="append"` это может приводить к неожиданному поведению (значения добавляются к общему списку). Используйте `default=None` и обрабатывайте `None` в коде.
- **`parser.error()` завершает процесс** с кодом 2 через `SystemExit`, а не возвращает значение. Учитывайте это в тестах и в коде, который ожидает «вернуть ошибку».
- **`prefix_chars`**: если хотите аргументы вида `/help` (Windows-стиль), нужно задать `prefix_chars="/"`, иначе `/` будет считаться позиционным значением.

## Лучшие практики

- **Выносите построение парсера в функцию** `build_parser() -> argparse.ArgumentParser`. Это упрощает тестирование и переиспользование.
- **Разделяйте парсинг и логику**: `main()` парсит и вызывает функции, которые принимают уже готовые значения, а не `Namespace`. Так логику можно тестировать без CLI.
- **Используйте `set_defaults(func=...)`** для подкоманд — чистый диспетчинг без длинных `if/elif`.
- **Группируйте аргументы** через `add_argument_group` для читаемой справки в больших CLI.
- **Пишите осмысленный `help`** для каждого аргумента и используйте `ArgumentDefaultsHelpFormatter`, чтобы автоматически показывать дефолты.
- **Кастомные типы вместо `choices`** для сложной валидации (диапазоны, форматы, существование файлов).
- **Отключайте `allow_abbrev=False`** в стабильных интерфейсах, чтобы избежать неоднозначностей при развитии CLI.
- **Проверяйте взаимные зависимости** аргументов вручную через `parser.error("...")`, если их нельзя выразить группами.
- **Возвращайте код возврата из `main`**: `sys.exit(main())`, где `main` возвращает int — удобно для скриптов и CI.
- **Тестируйте парсер**, передавая списки в `parse_args([...])` и ловя `SystemExit` на ошибках.

## Полный рабочий пример CLI с подкомандами

```python
#!/usr/bin/env python3
"""Учебный CLI в стиле git: команды add, commit, log."""
import argparse
import pathlib
import sys


# --- Кастомные типы / валидаторы ---------------------------------------------

def positive_int(value: str) -> int:
    """Принимает только положительное целое."""
    ivalue = int(value)
    if ivalue <= 0:
        raise argparse.ArgumentTypeError(f"ожидалось положительное число, получено {value!r}")
    return ivalue


def existing_path(value: str) -> pathlib.Path:
    """Проверяет, что путь существует."""
    path = pathlib.Path(value)
    if not path.exists():
        raise argparse.ArgumentTypeError(f"путь не существует: {value!r}")
    return path


# --- Обработчики команд ------------------------------------------------------

def cmd_add(args: argparse.Namespace) -> int:
    print(f"Добавляю файлы: {[str(p) for p in args.paths]}")
    if args.force:
        print("  (принудительно, force=True)")
    return 0


def cmd_commit(args: argparse.Namespace) -> int:
    print(f"Коммит с сообщением: {args.message!r}")
    if args.amend:
        print("  (изменение последнего коммита)")
    return 0


def cmd_log(args: argparse.Namespace) -> int:
    fmt = "json" if args.json else "yaml" if args.yaml else "text"
    print(f"Показываю последние {args.count} записей в формате {fmt}")
    return 0


# --- Построение парсера ------------------------------------------------------

def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        prog="vcs",
        description="Учебная система контроля версий (демо argparse).",
        epilog="Примеры:\n"
               "  vcs add file1.py file2.py --force\n"
               "  vcs commit -m 'Исправил баг'\n"
               "  vcs log -n 5 --json",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        allow_abbrev=False,
    )
    parser.add_argument("--version", action="version", version="%(prog)s 1.0.0")
    parser.add_argument("-v", "--verbose", action="count", default=0,
                        help="увеличить детализацию (можно повторять: -vvv)")

    subparsers = parser.add_subparsers(
        dest="command", required=True, metavar="COMMAND",
        title="команды", help="доступные команды",
    )

    # add ---------------------------------------------------------------------
    p_add = subparsers.add_parser("add", help="добавить файлы в индекс")
    p_add.add_argument("paths", nargs="+", type=existing_path, metavar="PATH",
                       help="один или несколько путей")
    p_add.add_argument("-f", "--force", action="store_true",
                       help="добавить даже игнорируемые файлы")
    p_add.set_defaults(func=cmd_add)

    # commit ------------------------------------------------------------------
    p_commit = subparsers.add_parser("commit", help="зафиксировать изменения")
    p_commit.add_argument("-m", "--message", required=True, metavar="MSG",
                          help="сообщение коммита")
    p_commit.add_argument("--amend", action="store_true",
                          help="изменить последний коммит")
    p_commit.set_defaults(func=cmd_commit)

    # log ---------------------------------------------------------------------
    p_log = subparsers.add_parser("log", help="показать историю")
    p_log.add_argument("-n", "--count", type=positive_int, default=10,
                       help="сколько записей показать (по умолчанию: 10)")
    fmt_group = p_log.add_mutually_exclusive_group()
    fmt_group.add_argument("--json", action="store_true", help="вывод в JSON")
    fmt_group.add_argument("--yaml", action="store_true", help="вывод в YAML")
    p_log.set_defaults(func=cmd_log)

    return parser


def main(argv: list[str] | None = None) -> int:
    parser = build_parser()
    args = parser.parse_args(argv)         # argv=None → читает sys.argv
    if args.verbose:
        print(f"[debug] уровень детализации: {args.verbose}")
    # Диспетчинг: каждая подкоманда привязала свой func через set_defaults
    return args.func(args)


if __name__ == "__main__":
    sys.exit(main())
```

Пример использования теста (без запуска процесса):

```python
def test_commit_requires_message():
    parser = build_parser()
    # Отсутствует -m → argparse завершит процесс с кодом 2
    import pytest
    with pytest.raises(SystemExit):
        parser.parse_args(["commit"])


def test_log_parsing():
    parser = build_parser()
    args = parser.parse_args(["log", "-n", "3", "--json"])
    assert args.count == 3
    assert args.json is True
    assert args.command == "log"
```

## Шпаргалка

```python
import argparse

# --- Создание парсера ---
p = argparse.ArgumentParser(
    prog="tool", description="...", epilog="...",
    formatter_class=argparse.ArgumentDefaultsHelpFormatter,
    allow_abbrev=False, fromfile_prefix_chars="@",
)

# --- Позиционные ---
p.add_argument("src")                              # обязательный
p.add_argument("files", nargs="*")                 # 0+
p.add_argument("files", nargs="+")                 # 1+

# --- Опциональные ---
p.add_argument("-n", "--name", required=True)      # обязательная опция
p.add_argument("--count", type=int, default=10)    # с типом и дефолтом
p.add_argument("--ratio", type=float)

# --- Флаги (action) ---
p.add_argument("--verbose", action="store_true")   # bool
p.add_argument("-v", action="count", default=0)    # -vvv → 3
p.add_argument("--tag", action="append")           # список
p.add_argument("--version", action="version", version="%(prog)s 1.0")

# --- nargs ---
p.add_argument("--xy", nargs=2, type=float)        # ровно 2
p.add_argument("--c", nargs="?", const="auto", default="off")  # 0 или 1
p.add_argument("rest", nargs=argparse.REMAINDER)   # всё остальное

# --- Ограничения ---
p.add_argument("--lvl", choices=["a", "b", "c"])
p.add_argument("--port", type=int, choices=range(1, 65536))
p.add_argument("--in", dest="input", metavar="ФАЙЛ")

# --- Группы ---
g = p.add_argument_group("Сеть")
mx = p.add_mutually_exclusive_group(required=True)
mx.add_argument("--json", action="store_true")
mx.add_argument("--yaml", action="store_true")

# --- Подкоманды ---
sub = p.add_subparsers(dest="cmd", required=True)
sp = sub.add_parser("run")
sp.add_argument("--fast", action="store_true")
sp.set_defaults(func=handler)

# --- Парсинг ---
args = p.parse_args()                     # из sys.argv
args = p.parse_args(["--name", "x"])      # из списка (тесты)
args, unknown = p.parse_known_args()      # неизвестные → unknown
data = vars(args)                         # Namespace → dict

# --- Кастомный тип ---
def positive(s):
    v = int(s)
    if v <= 0:
        raise argparse.ArgumentTypeError("должно быть > 0")
    return v
p.add_argument("--w", type=positive)
```

| Что нужно | Как сделать |
|-----------|-------------|
| Булев флаг | `action="store_true"` |
| Счётчик `-vvv` | `action="count", default=0` |
| Список значений | `action="append"` или `nargs="+"` |
| Ограничить выбор | `choices=[...]` |
| Привести тип | `type=int` / `type=float` / callable |
| Кастомная валидация | `type=функция`, бросающая `ArgumentTypeError` |
| Подкоманды (git-стиль) | `add_subparsers()` + `add_parser()` + `set_defaults(func=...)` |
| Один из нескольких | `add_mutually_exclusive_group()` |
| Версия | `action="version", version="%(prog)s X.Y"` |
| Имя в справке | `metavar="..."` |
| Имя атрибута | `dest="..."` |
| Игнорировать лишнее | `parse_known_args()` |
| Аргументы из файла | `fromfile_prefix_chars="@"` + `@file.txt` |
