# Warnings — подготовка к собеседованию

## Что это и зачем

Модуль `warnings` — это стандартный механизм Python для выдачи **предупреждений** (warnings). Предупреждение — это сообщение о том, что что-то в коде потенциально проблематично, но **не является фатальной ошибкой**: программа может продолжать работу.

Главные сценарии использования:

- **Оповестить пользователя библиотеки**, что он использует устаревший API, который скоро будет удалён (`DeprecationWarning`, `FutureWarning`, `PendingDeprecationWarning`).
- **Предупредить о сомнительном, но не запрещённом поведении** во время выполнения (`RuntimeWarning`) или компиляции (`SyntaxWarning`).
- **Сообщить об утечке ресурсов** — например, незакрытый файл или сокет (`ResourceWarning`).

Ключевое отличие от исключений: исключение **прерывает** выполнение (если его не поймать), а предупреждение по умолчанию просто **печатается в `stderr`** и выполнение идёт дальше.

```python
import warnings


def old_function():
    # Сообщаем пользователю, что функция устарела.
    # Программа НЕ упадёт — она просто напечатает предупреждение и продолжит.
    warnings.warn(
        "old_function() устарела, используйте new_function()",
        DeprecationWarning,
        stacklevel=2,  # покажет место ВЫЗОВА, а не строку внутри old_function
    )
    return 42


print(old_function())  # выведет предупреждение в stderr, затем 42
```

Зачем это нужно на практике:

- **Эволюция API без поломки кода пользователей.** Вы можете объявить функцию устаревшей сейчас, а удалить через несколько релизов, дав людям время на миграцию.
- **Управляемость.** Предупреждения можно фильтровать: скрывать, показывать один раз, превращать в ошибки — без изменения исходного кода, через фильтры или переменные окружения.
- **Тестируемость.** В тестах легко проверить, что библиотека выдала нужное предупреждение (`catch_warnings`, `pytest.warns`).

## Ключевые концепции

1. **Предупреждение — это объект-исключение.** Каждая категория (`UserWarning`, `DeprecationWarning` и т. д.) — это **класс, наследник `Warning`**, который, в свою очередь, наследник `Exception`. Поэтому предупреждение можно «превратить в ошибку» — тогда оно действительно будет выброшено как исключение.

2. **Фильтры предупреждений** — упорядоченный список правил, которые решают, что делать с каждым предупреждением: показать, скрыть, превратить в ошибку, показать один раз и т. д. Фильтры проверяются по порядку; срабатывает **первый подходящий**.

3. **Действие (action)** фильтра определяет реакцию: `error`, `ignore`, `always`, `default`, `module`, `once`.

4. **Дедупликация.** По умолчанию (`default`) одно и то же предупреждение из одного и того же места показывается **только один раз** — за это отвечает «реестр» уже показанных предупреждений (`__warningregistry__`).

5. **`stacklevel`** — указывает, на какой уровень стека «свалить» вину за предупреждение. Критичен для библиотек: позволяет показать строку кода пользователя, а не строку внутри библиотеки.

6. **Это не logging.** `warnings` предназначен для разработчиков, использующих ваш код. `logging` — для диагностики работы самого приложения в продакшене. Это разные инструменты с разными аудиториями (подробнее ниже).

## Основные функции/классы/методы

### `warnings.warn(message, category=UserWarning, stacklevel=1, source=None)`

Базовый способ выдать предупреждение.

- `message` — строка-сообщение **или** экземпляр класса-предупреждения. Если передать экземпляр, его класс используется как `category`, а аргумент `category` игнорируется.
- `category` — класс предупреждения (по умолчанию `UserWarning`).
- `stacklevel` — на сколько кадров стека «подняться» при определении места предупреждения.
- `source` — объект, ставший причиной (используется для `ResourceWarning` и отслеживания через `tracemalloc`).

```python
import warnings

# 1) Простая строка — категория по умолчанию UserWarning
warnings.warn("Что-то пошло не так, но это не критично")

# 2) Явная категория
warnings.warn("Этот параметр устарел", DeprecationWarning)

# 3) Передача экземпляра предупреждения (категория берётся из класса экземпляра)
warnings.warn(FutureWarning("Поведение изменится в версии 3.0"))
```

### Иерархия категорий предупреждений

```text
Warning  (наследник Exception)
├── UserWarning              # дефолтная категория для warnings.warn()
├── DeprecationWarning       # API устарел; для разработчиков (по умолчанию скрыт)
├── PendingDeprecationWarning# API устареет в будущем (по умолчанию скрыт)
├── SyntaxWarning            # сомнительная синтаксическая конструкция
├── RuntimeWarning           # сомнительное поведение во время выполнения
├── FutureWarning            # API устарел; для КОНЕЧНЫХ пользователей (показывается)
├── ImportWarning            # проблема с импортом (по умолчанию скрыт)
├── UnicodeWarning           # проблемы с Unicode
├── BytesWarning             # проблемы с bytes/bytearray
├── EncodingWarning          # неявная кодировка (по умолчанию скрыт)
└── ResourceWarning          # утечка ресурса (по умолчанию скрыт)
```

Что когда использовать:

```python
import warnings

# Warning — базовый класс. Напрямую обычно не используется,
# но удобен для фильтров: ловит ВСЕ предупреждения.

# UserWarning — общее предупреждение пользователю кода (дефолт).
warnings.warn("Передан подозрительный аргумент")

# DeprecationWarning — устаревший код, целевая аудитория — РАЗРАБОТЧИКИ.
# По умолчанию скрыт (кроме __main__), чтобы не пугать конечных пользователей.
warnings.warn("Метод .foo() устарел", DeprecationWarning, stacklevel=2)

# PendingDeprecationWarning — устареет в будущем, но ещё не сейчас.
# Самая «тихая» категория, по умолчанию скрыта.
warnings.warn("Класс Bar будет помечен устаревшим в 2.0",
              PendingDeprecationWarning, stacklevel=2)

# FutureWarning — как DeprecationWarning, но для КОНЕЧНЫХ пользователей.
# Показывается по умолчанию! Используют, например, pandas/numpy,
# когда МЕНЯЕТСЯ поведение (не удаление API, а смена результата).
warnings.warn("Поведение по умолчанию изменится в следующей версии",
              FutureWarning, stacklevel=2)

# RuntimeWarning — сомнительные операции во время выполнения.
# Например, деление 0.0/0.0 в numpy, или await корутины без её запуска.

# ResourceWarning — утечка ресурса (незакрытый файл/сокет).
# По умолчанию скрыт; включается через -W default или в режиме разработки -X dev.
```

### Формат фильтра и поле `action`

Фильтр — это кортеж из пяти элементов:

```text
(action, message, category, module, lineno)
```

- `action` — что делать (см. таблицу ниже).
- `message` — регулярное выражение (по началу строки сообщения, поиск без учёта регистра через `re.compile`). Пустая строка — совпадает со всем.
- `category` — класс предупреждения (совпадает он и его подклассы).
- `module` — регулярное выражение по **полному имени модуля** (`__name__`), откуда пришло предупреждение.
- `lineno` — номер строки (`0` — любая).

Возможные значения `action`:

| action    | Что делает |
|-----------|------------|
| `"error"`   | Превращает предупреждение в **исключение** (выбрасывается). |
| `"ignore"`  | Полностью **игнорирует** предупреждение (ничего не показывает). |
| `"always"`  | Показывает **всегда**, без дедупликации. |
| `"default"` | Показывает **один раз** для каждого уникального места (комбинация location). Поведение по умолчанию. |
| `"module"`  | Показывает **один раз на модуль** (по первому совпадению в каждом модуле). |
| `"once"`    | Показывает **один раз глобально** для совпадающего предупреждения, независимо от места. |
| `"all"`     | Псевдоним для `"always"` (используется в `-W`/`PYTHONWARNINGS`). |

### `warnings.filterwarnings(action, message="", category=Warning, module="", lineno=0, append=False)`

Добавляет фильтр, где `message` и `module` — **регулярные выражения**.

```python
import warnings

# Превратить ВСЕ DeprecationWarning в ошибки
warnings.filterwarnings("error", category=DeprecationWarning)

# Игнорировать предупреждения, сообщение которых начинается с "deprecated"
warnings.filterwarnings("ignore", message="deprecated")

# Показывать предупреждения только из модуля mylib.legacy
warnings.filterwarnings("always", module=r"mylib\.legacy")

# append=True добавляет фильтр в КОНЕЦ списка (по умолчанию — в начало).
# По умолчанию новый фильтр имеет приоритет (вставляется в начало).
warnings.filterwarnings("ignore", category=ResourceWarning, append=True)
```

### `warnings.simplefilter(action, category=Warning, lineno=0, append=False)`

То же, что `filterwarnings`, но **без regex** — `message` и `module` не настраиваются (всегда «совпадает со всем»). Проще и быстрее.

```python
import warnings

# Превратить ВСЕ предупреждения в ошибки (полезно в CI / тестах)
warnings.simplefilter("error")

# Показывать все предупреждения всегда (отключить дедупликацию)
warnings.simplefilter("always")

# Только для DeprecationWarning — показывать всегда
warnings.simplefilter("always", DeprecationWarning)
```

### `warnings.resetwarnings()`

Сбрасывает **все** фильтры (и те, что добавлены кодом, и те, что заданы через `-W`/`PYTHONWARNINGS`). Список фильтров становится пустым.

```python
import warnings

warnings.resetwarnings()  # очищает warnings.filters целиком
print(warnings.filters)   # []  -> теперь действует только "default"-логика ядра
```

Осторожно: после `resetwarnings()` теряются и системные фильтры — используйте с пониманием.

### `warnings.catch_warnings(record=False)` — контекстный менеджер

Сохраняет текущее состояние модуля (`filters` и `showwarning`) при входе и **восстанавливает** его при выходе. Идеально для тестов: можно временно менять фильтры, не влияя на остальной код.

```python
import warnings


def deprecated_api():
    warnings.warn("устарело", DeprecationWarning, stacklevel=2)


# record=True — перехватывает предупреждения в список вместо вывода
with warnings.catch_warnings(record=True) as caught:
    warnings.simplefilter("always")  # ловим все, без дедупликации
    deprecated_api()

    assert len(caught) == 1
    w = caught[0]
    assert issubclass(w.category, DeprecationWarning)
    assert "устарело" in str(w.message)
    print("Перехвачено:", w.category.__name__, "-", w.message)

# Вне блока with фильтры и showwarning восстановлены автоматически.
```

В Python 3.11+ у `catch_warnings` появились удобные параметры (`action`, `category`, `record` и пр.), позволяющие задать фильтр прямо в конструкторе:

```python
import warnings

with warnings.catch_warnings(action="error"):  # Python 3.11+
    # Внутри блока любое предупреждение станет исключением
    try:
        warnings.warn("станет ошибкой")
    except UserWarning as e:
        print("Поймали как исключение:", e)
```

### `warnings.warn_explicit(message, category, filename, lineno, module=None, registry=None, module_globals=None, source=None)`

Низкоуровневый аналог `warn`, где вы **сами** указываете местоположение (`filename`, `lineno`, `module`). `warn()` под капотом вычисляет эти значения из стека (с учётом `stacklevel`) и вызывает `warn_explicit()`.

Используется, когда вы генерируете предупреждение «от имени» другого места — например, в кодогенераторах, парсерах, линтерах, или когда нужно указать на конкретную строку чужого файла.

```python
import warnings

# Сгенерировать предупреждение, как будто оно из чужого файла config.py, строка 10
warnings.warn_explicit(
    "Подозрительная настройка в конфиге",
    UserWarning,
    filename="config.py",
    lineno=10,
)
```

### `warnings.showwarning(message, category, filename, lineno, file=None, line=None)` и `formatwarning(...)`

- `showwarning` — функция, которая **выводит** предупреждение (можно переопределить, например, чтобы слать в `logging`).
- `formatwarning` — возвращает отформатированную строку предупреждения (можно переопределить, чтобы изменить формат).

```python
import warnings
import logging

logging.basicConfig(level=logging.WARNING)

# Перенаправить ВСЕ предупреждения в logging «штатным» способом:
logging.captureWarnings(True)  # рекомендуемый способ интеграции

warnings.warn("пойдёт в logging через логгер 'py.warnings'")
logging.captureWarnings(False)


# Либо вручную переопределить showwarning:
def custom_showwarning(message, category, filename, lineno, file=None, line=None):
    logging.getLogger("warnings").warning(
        "%s:%s: %s: %s", filename, lineno, category.__name__, message
    )


warnings.showwarning = custom_showwarning
warnings.warn("теперь вывод через custom_showwarning")
```

### Параметр `-W` и переменная окружения `PYTHONWARNINGS`

Позволяют управлять фильтрами **извне**, без правки кода.

Формат одной спецификации: `action:message:category:module:lineno` (все поля кроме `action` опциональны).

```bash
# Превратить все предупреждения в ошибки
python -W error my_script.py

# Игнорировать все предупреждения
python -W ignore my_script.py

# Показывать все DeprecationWarning (которые по умолчанию скрыты)
python -W default::DeprecationWarning my_script.py

# Показывать всегда, конкретное сообщение и модуль:
python -W "always:deprecated:DeprecationWarning:mylib.utils:0" app.py

# Несколько фильтров: повторяем -W (применяются по порядку)
python -W ignore -W error::DeprecationWarning app.py

# Через переменную окружения (фильтры разделяются запятой)
export PYTHONWARNINGS="error::DeprecationWarning,ignore::ResourceWarning"
python app.py
```

Дополнительно: **режим разработчика** `-X dev` (или `PYTHONDEVMODE=1`) включает более строгие фильтры по умолчанию, в т. ч. показывает `DeprecationWarning` и `ResourceWarning`. Очень полезно при разработке.

```bash
python -X dev app.py   # эквивалентно -W default плюс другие проверки
```

### Почему `DeprecationWarning` по умолчанию скрыт

С Python 3.2 `DeprecationWarning` **по умолчанию игнорируется**, КРОМЕ кода, выполняемого напрямую в `__main__`.

Причина: целевая аудитория `DeprecationWarning` — **разработчики**, а не конечные пользователи. Если бы такие предупреждения показывались всем, обычные пользователи приложений видели бы непонятный «шум» из глубин сторонних библиотек, на который не могут повлиять.

Как увидеть скрытые предупреждения:

```bash
python -W default::DeprecationWarning app.py   # включить показ
python -X dev app.py                            # режим разработчика
```

```python
import warnings

# Программно — например, в начале тестов:
warnings.simplefilter("default", DeprecationWarning)
# или строже:
warnings.simplefilter("error", DeprecationWarning)
```

`FutureWarning`, в отличие от `DeprecationWarning`, **показывается по умолчанию** — именно потому, что предназначен конечным пользователям.

### Как сделать предупреждение ошибкой

```python
import warnings

# 1) Глобально — все предупреждения в ошибки
warnings.simplefilter("error")

# 2) Только определённая категория
warnings.filterwarnings("error", category=DeprecationWarning)

# 3) Точечно по сообщению (regex)
warnings.filterwarnings("error", message="устарел.*", category=DeprecationWarning)

try:
    warnings.warn("устарела функция X", DeprecationWarning)
except DeprecationWarning as e:
    print("Перехвачено как исключение:", e)
```

Из командной строки: `python -W error::DeprecationWarning app.py`. Часто используется в CI, чтобы тесты падали при появлении новых deprecation'ов.

### Корректное использование `stacklevel` при написании библиотек

`stacklevel=1` (по умолчанию) указывает на строку **внутри функции, вызвавшей `warn`**. Для библиотек это бесполезно — пользователь увидит строку вашей библиотеки, а не свою.

`stacklevel=2` указывает на **место вызова вашей функции** — то, что нужно пользователю.

```python
# Файл mylib.py
import warnings


def public_api(x):
    # ПЛОХО: пользователь увидит "mylib.py:7: ..." — строку внутри библиотеки
    # warnings.warn("устарело", DeprecationWarning)

    # ХОРОШО: stacklevel=2 покажет строку, ГДЕ пользователь вызвал public_api()
    warnings.warn("public_api устарела", DeprecationWarning, stacklevel=2)
    return x


# Файл user_code.py
# public_api(10)
#   -> user_code.py:N: DeprecationWarning: public_api устарела   (правильно!)
```

Если предупреждение генерируется через несколько уровней обёрток, увеличивайте `stacklevel` соответственно (3, 4 …) или, в Python 3.12+, используйте `skip_file_prefixes` у `warn` для пропуска целых файлов:

```python
import warnings

# Python 3.12+: пропустить кадры из файлов с указанными префиксами,
# чтобы стабильно указывать на код пользователя независимо от глубины обёрток.
def helper():
    warnings.warn(
        "устарело",
        DeprecationWarning,
        skip_file_prefixes=(__file__,),  # пропускаем все кадры этого файла
    )
```

## Частые вопросы на собеседовании

**Q1. В чём разница между `warnings` и исключениями?**
A: Исключение по умолчанию **прерывает** выполнение, если его не поймать. Предупреждение по умолчанию **печатается в `stderr`**, а выполнение продолжается. Но предупреждение — это тоже класс-исключение (наследник `Warning` → `Exception`), и через фильтр `action="error"` его можно превратить в настоящее исключение.

**Q2. Почему `DeprecationWarning` не виден по умолчанию, а `FutureWarning` виден?**
A: `DeprecationWarning` адресован **разработчикам** и по умолчанию скрыт (кроме `__main__`), чтобы не засорять вывод конечным пользователям сторонними предупреждениями. `FutureWarning` адресован **конечным пользователям** (например, смена поведения в pandas/numpy) и поэтому показывается по умолчанию.

**Q3. Что делает `stacklevel` и почему он важен для библиотек?**
A: `stacklevel` указывает, какой кадр стека считать «источником» предупреждения. `stacklevel=1` — строка внутри функции, вызвавшей `warn`. Для библиотек нужно `stacklevel=2` (или больше при обёртках), чтобы предупреждение указывало на **код пользователя**, а не на внутренности библиотеки.

**Q4. Как сделать так, чтобы конкретное предупреждение приводило к падению программы?**
A: Установить фильтр с действием `error`: `warnings.filterwarnings("error", category=DeprecationWarning)` или из CLI `python -W error::DeprecationWarning`. Тогда предупреждение будет выброшено как исключение.

**Q5. Зачем нужен `catch_warnings` и почему его особенно важно использовать в тестах?**
A: `catch_warnings` — контекстный менеджер, который сохраняет и восстанавливает состояние модуля (`filters`, `showwarning`). В тестах он позволяет временно изменить фильтры (например, `simplefilter("always")`) и/или перехватить предупреждения (`record=True`) **без побочных эффектов** на глобальное состояние. Замечание: он **не потокобезопасен**.

**Q6. Почему предупреждение иногда показывается только один раз, а потом «пропадает»?**
A: Действие по умолчанию — `default`, оно дедуплицирует: одно и то же предупреждение из одного и того же места (файл+строка) показывается только при первом срабатывании. За это отвечает реестр `__warningregistry__`. Чтобы видеть всегда — используйте `always`.

**Q7. В чём разница между `filterwarnings` и `simplefilter`?**
A: `filterwarnings` принимает `message` и `module` как **регулярные выражения** — более гибкий. `simplefilter` их не настраивает (совпадает со всем) — проще и без накладных расходов на regex. Оба добавляют фильтр в начало списка (или в конец при `append=True`).

**Q8. Какой порядок применения фильтров? Что важнее — добавленный кодом или заданный через `-W`?**
A: Фильтры проверяются по порядку, срабатывает **первый совпавший**. Новые фильтры (через `filterwarnings`/`simplefilter` без `append`) вставляются **в начало** списка и поэтому имеют приоритет над системными (из `-W`/`PYTHONWARNINGS`), которые лежат в конце.

**Q9. Чем `warnings` отличается от `logging`? Когда что использовать?**
A: `warnings` — для оповещения **разработчиков, использующих ваш код**, о проблемных паттернах (устаревший API, сомнительные операции); обычно одно предупреждение на место, с дедупликацией. `logging` — для диагностики работы **самого приложения** в рантайме (info/debug/error), с уровнями, хендлерами, ротацией. Интегрировать их можно через `logging.captureWarnings(True)`.

**Q10. Что такое `warn_explicit` и когда он нужен?**
A: Низкоуровневая версия `warn`, где вы **вручную** задаёте `filename`, `lineno`, `module`. Нужна, когда предупреждение генерируется «от имени» другого места (кодогенераторы, парсеры, линтеры), и вычисление из стека не подходит. `warn()` сам вызывает `warn_explicit()`, вычислив локацию по `stacklevel`.

**Q11. Как массово увидеть скрытые `DeprecationWarning` при разработке?**
A: Запустить с `python -X dev` (режим разработчика) или `python -W default::DeprecationWarning`. Программно: `warnings.simplefilter("default", DeprecationWarning)` либо строже — `"error"`. В тестах pytest показывает их по умолчанию и имеет `filterwarnings`-маркеры.

**Q12. Что делает `resetwarnings()` и в чём подвох?**
A: Полностью очищает список `warnings.filters`. Подвох: удаляет **и** фильтры, заданные через `-W`/`PYTHONWARNINGS`, не только добавленные кодом. После сброса остаётся только встроенная логика ядра.

**Q13. Можно ли перенаправить предупреждения в логи? Как правильно?**
A: Да. Рекомендуемый способ — `logging.captureWarnings(True)`: предупреждения пойдут через логгер `py.warnings`. Альтернатива — переопределить `warnings.showwarning`. Не забывайте, что дедупликация фильтров всё ещё действует.

**Q14. Что произойдёт с `warn()`, если передать экземпляр предупреждения вместо строки?**
A: Класс экземпляра становится категорией, а переданный аргумент `category` **игнорируется**. Например, `warnings.warn(FutureWarning("..."))` создаст `FutureWarning` независимо от значения по умолчанию.

## Подводные камни (gotchas)

- **Дедупликация скрывает повторы.** При `default`/`module`/`once` предупреждение может показаться один раз и «исчезнуть». При отладке используйте `always`. Реестр `__warningregistry__` кэширует уже показанные — повторный запуск в той же сессии не покажет их снова.

- **`stacklevel=1` бесполезен в библиотеках.** Забыли указать `stacklevel=2` — пользователь увидит строку вашей библиотеки и не поймёт, где его ошибка.

- **`DeprecationWarning` молчит по умолчанию.** Можно случайно «не заметить», что используете устаревший API. Включайте `-X dev` или фильтры на время разработки/тестов.

- **`catch_warnings` не потокобезопасен.** Он меняет глобальное состояние модуля. В многопоточном коде использование `catch_warnings` в одном потоке повлияет на другие.

- **`record=True` сам по себе не отключает фильтры.** Если действует `default`/`ignore`, предупреждение может не попасть в список. Внутри блока обычно добавляют `warnings.simplefilter("always")`.

- **`resetwarnings()` сносит и системные фильтры** (`-W`, `PYTHONWARNINGS`), а не только пользовательские. Легко неожиданно «оглохнуть» к нужным предупреждениям.

- **Порядок фильтров имеет значение.** Срабатывает первый совпавший. Новый фильтр без `append=True` встаёт в начало и может перекрыть всё остальное.

- **`message` и `module` в `filterwarnings` — это regex.** Спецсимволы (`.`, `(`, `)`) нужно экранировать. `module="mylib.utils"` совпадёт и с `mylibXutils` из-за нес экранированной точки.

- **Предупреждение, превращённое в ошибку, нужно ловить как исключение** соответствующей категории (или `Warning`), а не как `UserWarning` всегда.

- **`SyntaxWarning` возникает на этапе компиляции**, поэтому установка фильтра в рантайме может не успеть — иногда нужен `-W` при запуске.

## Лучшие практики

- В библиотеках **всегда указывайте `stacklevel`** (минимум `2`), чтобы предупреждение указывало на код пользователя. С Python 3.12+ рассмотрите `skip_file_prefixes`.

- Для устаревшего API используйте **правильную категорию**: `DeprecationWarning` (для разработчиков), `FutureWarning` (для конечных пользователей при смене поведения), `PendingDeprecationWarning` (заранее, «тихо»).

- **Не подавляйте предупреждения глобально и навсегда** в библиотечном коде — это решение должно оставаться за приложением/пользователем. Управляйте поведением через `-W`/`PYTHONWARNINGS` или локальный `catch_warnings`.

- В **тестах** используйте `catch_warnings`/`pytest.warns` для проверки факта предупреждения и `simplefilter("error")`, чтобы ловить регрессии (новые deprecation'ы валят CI).

- При разработке запускайте с **`-X dev`**, чтобы видеть `DeprecationWarning`/`ResourceWarning`.

- Для интеграции с логированием используйте **`logging.captureWarnings(True)`**, а не самописное переопределение `showwarning`, если нет особых причин.

- Оборачивайте временные изменения фильтров в **`with warnings.catch_warnings():`**, чтобы не «протекать» в остальной код.

- В `filterwarnings` **экранируйте regex** (`re.escape`) для имён модулей/сообщений, если хотите точного совпадения.

- Документируйте устаревание: предупреждение + запись в CHANGELOG + срок удаления. Предупреждение без плана удаления бесполезно.

## Шпаргалка

```python
import warnings

# --- Выдать предупреждение ---
warnings.warn("сообщение")                                  # UserWarning
warnings.warn("устарело", DeprecationWarning, stacklevel=2) # с категорией и stacklevel
warnings.warn(FutureWarning("поведение изменится"))         # экземпляр -> категория из него

# --- Фильтры ---
warnings.simplefilter("error")                              # все -> ошибки (без regex)
warnings.simplefilter("always", DeprecationWarning)         # категория -> всегда
warnings.filterwarnings("ignore", message="^deprecated",    # с regex
                        category=UserWarning, module=r"mylib\.")
warnings.filterwarnings("error", category=DeprecationWarning, append=True)
warnings.resetwarnings()                                    # очистить ВСЕ фильтры

# --- Действия (action) ---
# "error"   -> выбросить как исключение
# "ignore"  -> скрыть
# "always"  -> показывать всегда (без дедупликации)
# "default" -> один раз на (сообщение+категория+модуль+строка)  [по умолчанию]
# "module"  -> один раз на модуль
# "once"    -> один раз глобально

# --- Тесты / временные изменения ---
with warnings.catch_warnings(record=True) as caught:
    warnings.simplefilter("always")
    do_something()
    assert any(issubclass(w.category, DeprecationWarning) for w in caught)

with warnings.catch_warnings(action="error"):   # Python 3.11+
    ...

# --- Низкоуровневое ---
warnings.warn_explicit("msg", UserWarning, filename="f.py", lineno=10)

# --- Интеграция с logging ---
import logging
logging.captureWarnings(True)   # предупреждения -> логгер 'py.warnings'
```

```bash
# --- CLI / переменные окружения ---
python -W error app.py                       # все -> ошибки
python -W ignore app.py                      # скрыть все
python -W default::DeprecationWarning app.py # показать скрытые Deprecation
python -W "always:msg:UserWarning:mod:0" app.py   # полный формат
python -X dev app.py                         # режим разработчика (видит Deprecation/Resource)

export PYTHONWARNINGS="error::DeprecationWarning,ignore::ResourceWarning"
```

| Категория                  | Аудитория      | По умолчанию |
|----------------------------|----------------|--------------|
| `UserWarning`              | пользователь   | показывается |
| `DeprecationWarning`       | разработчики   | скрыт (кроме `__main__`) |
| `PendingDeprecationWarning`| разработчики   | скрыт        |
| `FutureWarning`            | конечные польз.| показывается |
| `RuntimeWarning`           | разработчики   | показывается |
| `SyntaxWarning`            | разработчики   | показывается |
| `ResourceWarning`          | разработчики   | скрыт (виден в `-X dev`) |
