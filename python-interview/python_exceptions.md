# Исключения (Exceptions) — подготовка к собеседованию

## Что это и зачем

**Исключение (exception)** — это объект, представляющий ошибку или иное нештатное событие, которое прерывает нормальный поток выполнения программы. В Python механизм исключений — это основной способ сигнализировать об ошибках. В отличие от языков, где принято возвращать коды ошибок, Python придерживается принципа **EAFP** («Easier to Ask Forgiveness than Permission» — проще попросить прощения, чем разрешения): мы пытаемся выполнить операцию и обрабатываем исключение, если что-то пошло не так.

Зачем это знать на собеседовании:
- Понимание иерархии исключений позволяет ловить ошибки на нужном уровне абстракции, не «проглатывая» лишнего.
- Знание о `BaseException` vs `Exception` критично для написания корректного кода с обработкой `KeyboardInterrupt` и `SystemExit`.
- Цепочки исключений (`raise ... from ...`), `ExceptionGroup`, `add_note` — это современные возможности (Python 3.11+), которые часто спрашивают на middle/senior собеседованиях.
- Грамотная работа с `try/except/else/finally` отличает зрелого инженера от новичка.

В Python всё, что можно «выбросить» через `raise`, должно быть экземпляром (или подклассом) `BaseException`. Попытка возбудить что-то другое приведёт к `TypeError`.

```python
raise "ошибка"  # TypeError: exceptions must derive from BaseException
```

## Ключевые концепции

- **Всё наследуется от `BaseException`.** Это корень иерархии. Напрямую от него наследуются только «системные» исключения, которые НЕ должны перехватываться обычным `except Exception`.
- **`Exception`** — базовый класс для всех «обычных» ошибок приложения. В 99% случаев пользовательские исключения и блоки `except` работают именно с `Exception` и его потомками.
- **`KeyboardInterrupt`, `SystemExit`, `GeneratorExit`** наследуются напрямую от `BaseException`, а НЕ от `Exception`. Причина: это не ошибки логики программы, а управляющие сигналы. Если бы они наследовались от `Exception`, то типичный код `try: ... except Exception: pass` случайно перехватывал бы `Ctrl+C` (`KeyboardInterrupt`) и `sys.exit()` (`SystemExit`), мешая корректному завершению программы. Разделение позволяет писать `except Exception` безопасно, не блокируя выход и прерывание.
- **`LookupError`** — общий базовый класс для `KeyError` и `IndexError` (ошибки доступа по ключу/индексу).
- **`ArithmeticError`** — базовый класс для `ZeroDivisionError`, `OverflowError`, `FloatingPointError`.
- **`OSError`** — базовый класс для ошибок операционной системы (файлы, сеть, права доступа). В Python 3.3+ старые `IOError`, `EnvironmentError`, `WindowsError`, `socket.error` стали псевдонимами `OSError`, а появились конкретные подклассы: `FileNotFoundError`, `PermissionError`, `FileExistsError`, `IsADirectoryError`, `NotADirectoryError`, `ConnectionError`, `TimeoutError` и др.
- **Цепочки исключений.** При возбуждении нового исключения внутри `except` Python неявно связывает их через `__context__`. Явная связь делается через `raise ... from ...`, что заполняет `__cause__`.
- **`ExceptionGroup` и `except*`** (Python 3.11+) — механизм для группировки нескольких исключений (например, из конкурентных задач) и их выборочной обработки.
- **`add_note()`** (Python 3.11+) — позволяет добавить дополнительную текстовую информацию к исключению, не меняя его тип.

### Диаграмма иерархии (текстовое дерево)

```text
BaseException
 ├── BaseExceptionGroup        # 3.11+ (базовый класс групп)
 ├── KeyboardInterrupt          # Ctrl+C — НЕ ловится except Exception
 ├── SystemExit                 # sys.exit() — НЕ ловится except Exception
 ├── GeneratorExit              # .close() на генераторе — НЕ ловится except Exception
 └── Exception                  # базовый класс «обычных» ошибок
      ├── ArithmeticError
      │    ├── ZeroDivisionError
      │    ├── OverflowError
      │    └── FloatingPointError
      ├── AssertionError         # assert
      ├── AttributeError         # obj.missing_attr
      ├── BufferError
      ├── EOFError               # input() при конце файла
      ├── ExceptionGroup         # 3.11+ (наследует Exception и BaseExceptionGroup)
      ├── ImportError
      │    └── ModuleNotFoundError   # 3.6+
      ├── LookupError
      │    ├── IndexError        # list[100]
      │    └── KeyError          # dict['нет такого ключа']
      ├── MemoryError
      ├── NameError
      │    └── UnboundLocalError
      ├── OSError                # = IOError = EnvironmentError
      │    ├── BlockingIOError
      │    ├── ChildProcessError
      │    ├── ConnectionError
      │    │    ├── BrokenPipeError
      │    │    ├── ConnectionAbortedError
      │    │    ├── ConnectionRefusedError
      │    │    └── ConnectionResetError
      │    ├── FileExistsError
      │    ├── FileNotFoundError
      │    ├── InterruptedError
      │    ├── IsADirectoryError
      │    ├── NotADirectoryError
      │    ├── PermissionError
      │    ├── ProcessLookupError
      │    └── TimeoutError
      ├── ReferenceError
      ├── RuntimeError
      │    ├── NotImplementedError
      │    └── RecursionError     # превышена глубина рекурсии
      ├── StopIteration          # конец итератора
      ├── StopAsyncIteration     # конец async-итератора
      ├── SyntaxError
      │    └── IndentationError
      │         └── TabError
      ├── SystemError
      ├── TypeError              # неверный тип операнда
      ├── ValueError             # верный тип, но неверное значение
      │    └── UnicodeError
      │         ├── UnicodeDecodeError
      │         ├── UnicodeEncodeError
      │         └── UnicodeTranslateError
      └── Warning                # базовый класс предупреждений
           ├── DeprecationWarning
           ├── UserWarning
           └── ...
```

## Основные функции/классы/методы

### `BaseException` — атрибуты

Любое исключение хранит несколько важных атрибутов:

```python
try:
    raise ValueError("неверное значение", 42)
except ValueError as e:
    print(e.args)        # => ('неверное значение', 42) — кортеж аргументов
    print(str(e))        # => ('неверное значение', 42)
    print(e.__cause__)   # => None — явная причина (raise ... from ...)
    print(e.__context__) # => None — неявный контекст (исключение во время обработки)
    print(e.__traceback__)  # => объект traceback (или None)
```

### `ZeroDivisionError`, `ArithmeticError`

```python
try:
    result = 10 / 0
except ArithmeticError as e:       # ловит и ZeroDivisionError тоже
    print(type(e).__name__)        # => ZeroDivisionError
    print(e)                       # => division by zero
```

### `ValueError` vs `TypeError`

Это частая пара для путаницы:
- `TypeError` — **тип** значения не подходит для операции.
- `ValueError` — тип правильный, но **само значение** недопустимо.

```python
int("abc")     # ValueError: invalid literal for int() with base 10: 'abc'
int([1, 2])    # TypeError: int() argument must be a string... not 'list'
len(42)        # TypeError: object of type 'int' has no len()
[].index(5)    # ValueError: 5 is not in list
```

### `KeyError`, `IndexError`, `LookupError`

```python
d = {"a": 1}
try:
    d["b"]
except LookupError as e:           # LookupError — родитель KeyError и IndexError
    print(repr(e))                 # => KeyError('b')

lst = [1, 2, 3]
try:
    lst[10]
except IndexError as e:
    print(e)                       # => list index out of range
```

### `AttributeError`, `NameError`

```python
class A:
    pass

a = A()
try:
    a.foo                          # нет такого атрибута
except AttributeError as e:
    print(e)  # => 'A' object has no attribute 'foo'

try:
    print(undefined_variable)      # имя не определено
except NameError as e:
    print(e)  # => name 'undefined_variable' is not defined
```

### `StopIteration` / `StopAsyncIteration`

`StopIteration` сигнализирует об окончании итератора. Цикл `for` ловит его автоматически.

```python
it = iter([1, 2])
print(next(it))   # => 1
print(next(it))   # => 2
print(next(it))   # StopIteration

# Можно передать значение по умолчанию, чтобы не ловить исключение:
print(next(it, "конец"))  # => конец
```

В генераторах `return value` транслируется в `StopIteration(value)`:

```python
def gen():
    yield 1
    return 42  # станет StopIteration.value

g = gen()
next(g)                # => 1
try:
    next(g)
except StopIteration as e:
    print(e.value)     # => 42
```

`StopAsyncIteration` — аналог для асинхронных итераторов (метод `__anext__`).

### `RuntimeError`, `RecursionError`

```python
def recurse(n):
    return recurse(n + 1)

try:
    recurse(0)
except RecursionError as e:        # подкласс RuntimeError
    print("слишком глубокая рекурсия")
```

`RuntimeError` используется, когда ошибка не подходит ни под одну другую категорию. `NotImplementedError` — его подкласс, бросается из абстрактных методов-заглушек.

```python
class Base:
    def process(self):
        raise NotImplementedError("Переопределите в подклассе")
```

### `OSError` и подклассы

```python
try:
    open("/no/such/file.txt")
except FileNotFoundError as e:     # подкласс OSError
    print(e.errno)                 # => 2
    print(e.strerror)              # => No such file or directory
    print(e.filename)              # => /no/such/file.txt

# Можно ловить обобщённо и анализировать errno:
import errno
try:
    open("/root/secret", "w")
except OSError as e:
    if e.errno == errno.EACCES:
        print("нет прав доступа")  # PermissionError
```

### `ImportError` / `ModuleNotFoundError`

```python
try:
    import nonexistent_module
except ModuleNotFoundError as e:   # подкласс ImportError (3.6+)
    print(e.name)                  # => nonexistent_module

try:
    from os import nonexistent_name
except ImportError as e:           # сам модуль есть, но имени в нём нет
    print(e.name, e.path)
```

### Пользовательские исключения

Наследуйтесь от `Exception` (а не от `BaseException`!). Можно добавлять собственные атрибуты.

```python
class AppError(Exception):
    """Базовое исключение приложения."""

class InsufficientFundsError(AppError):
    def __init__(self, balance, amount):
        self.balance = balance
        self.amount = amount
        super().__init__(f"Недостаточно средств: баланс {balance}, требуется {amount}")

try:
    raise InsufficientFundsError(100, 250)
except AppError as e:              # ловим по базовому классу всю «семью»
    print(e)                       # => Недостаточно средств: баланс 100, требуется 250
    print(e.amount)                # => 250
```

### `raise`, `raise ... from ...`, цепочки

```python
def parse_config(raw):
    try:
        return int(raw)
    except ValueError as e:
        # Явная цепочка: __cause__ = e, и traceback покажет
        # "The above exception was the direct cause of the following exception"
        raise AppError("Не удалось разобрать конфиг") from e

# Неявная цепочка (implicit chaining):
try:
    try:
        1 / 0
    except ZeroDivisionError:
        # Здесь raise без from — но Python всё равно свяжет через __context__
        # traceback: "During handling of the above exception, another occurred"
        raise ValueError("ошибка обработки")
    except ValueError as e:
        print(type(e.__context__).__name__)  # => ZeroDivisionError
        print(e.__cause__)                    # => None (т.к. не было from)
except Exception:
    pass
```

### Подавление контекста: `from None`

Иногда исходное исключение — деталь реализации, и его не нужно показывать в трейсбеке.

```python
def get_value(d, key):
    try:
        return d[key]
    except KeyError:
        # from None подавляет контекст: __suppress_context__ = True
        raise ValueError(f"Неизвестный ключ: {key}") from None

try:
    get_value({}, "x")
except ValueError as e:
    print(e.__context__)            # => KeyError('x') (всё ещё хранится)
    print(e.__suppress_context__)   # => True (но не печатается в трейсбеке)
```

### `try / except / else / finally` — порядок выполнения

```python
def demo(x):
    try:
        print("1. try")
        if x == 0:
            raise ValueError("ноль")
        result = 100 / x
    except ValueError as e:
        print("2. except ValueError")
        return "обработано"
    else:
        # else выполняется ТОЛЬКО если в try НЕ было исключения
        print("3. else")
        return result
    finally:
        # finally выполняется ВСЕГДА: при успехе, при исключении, при return/break
        print("4. finally")

print(demo(0))
# 1. try
# 2. except ValueError
# 4. finally    <- finally выполнился даже после return в except
# обработано

print(demo(5))
# 1. try
# 3. else
# 4. finally
# 20.0
```

Порядок: `try` → (`except` ИЛИ `else`) → `finally`. `else` нужен, чтобы отделить «код, который может бросить исключение» от «кода, который выполняется при успехе».

### Повторное возбуждение (re-raise)

```python
import logging

def process():
    try:
        risky_operation()
    except Exception:
        logging.exception("Ошибка при обработке")
        raise              # «голый» raise повторно бросает ТЕКУЩЕЕ исключение,
                           # сохраняя оригинальный traceback

def risky_operation():
    raise RuntimeError("сбой")
```

«Голый» `raise` внутри `except` повторно поднимает текущее исключение без потери трейсбека — это правильный способ залогировать и пробросить дальше.

### `ExceptionGroup` и `except*` (Python 3.11+)

```python
# Группировка нескольких исключений:
eg = ExceptionGroup(
    "несколько ошибок",
    [ValueError("плохое значение"), TypeError("плохой тип"), KeyError("k")],
)

try:
    raise eg
except* ValueError as group:
    # except* собирает ВСЕ ValueError из группы в подгруппу
    print("Поймали ValueError:", group.exceptions)
except* TypeError as group:
    print("Поймали TypeError:", group.exceptions)
# KeyError не пойман — будет повторно поднят как ExceptionGroup
```

`except*` может сработать НЕСКОЛЬКО раз (по одному на каждый совпавший тип), в отличие от обычного `except`. Применяется в основном с `asyncio.TaskGroup`, где несколько задач могут упасть одновременно.

### `add_note()` (Python 3.11+)

```python
try:
    raise ValueError("базовая ошибка")
except ValueError as e:
    e.add_note("Контекст: обработка пользователя id=42")
    e.add_note("Повтор не помог")
    print(e.__notes__)  # => ['Контекст: обработка пользователя id=42', 'Повтор не помог']
    # raise  # ноты будут напечатаны в трейсбеке под сообщением
```

## Частые вопросы на собеседовании

**Q: Почему `KeyboardInterrupt` и `SystemExit` наследуются от `BaseException`, а не от `Exception`?**
A: Чтобы их случайно не перехватывал распространённый паттерн `except Exception`. Это управляющие сигналы, а не ошибки логики: `KeyboardInterrupt` — нажатие `Ctrl+C`, `SystemExit` — вызов `sys.exit()`. Если бы они были потомками `Exception`, код вроде `try: ... except Exception: pass` блокировал бы завершение программы и игнорировал прерывание пользователя. `GeneratorExit` (бросается при `.close()` генератора) — по той же причине.

**Q: В чём разница между `ValueError` и `TypeError`?**
A: `TypeError` — операнд неподходящего **типа** (`len(5)`, `"a" + 1`). `ValueError` — тип верный, но **значение** недопустимо (`int("abc")`, `math.sqrt(-1)` в некоторых случаях). Правило: сначала проверяется тип, потом значение.

**Q: Чем отличается `raise X` от `raise X from Y` и от `raise X from None`?**
A: `raise X` внутри `except` неявно устанавливает `X.__context__` = текущее исключение (implicit chaining), трейсбек печатает «During handling of the above exception...». `raise X from Y` явно устанавливает `X.__cause__ = Y` (explicit chaining), трейсбек печатает «The above exception was the direct cause...». `raise X from None` подавляет вывод контекста (`__suppress_context__ = True`) — полезно, чтобы скрыть внутренние детали реализации.

**Q: Когда выполняется блок `else` в `try/except/else/finally`?**
A: `else` выполняется только если в `try` **не возникло** исключения. Его смысл — вынести из `try` код, который не должен перехватываться `except`, тем самым сузив область, где ловятся ошибки. `finally` выполняется всегда.

**Q: Выполнится ли `finally`, если в `try` или `except` есть `return`?**
A: Да. `finally` выполняется ВСЕГДА, даже после `return`, `break`, `continue` или другого исключения. Более того, если в `finally` есть свой `return`, он **перезапишет** возвращаемое значение из `try`/`except` — это известная ловушка.

**Q: Что делает «голый» `raise` без аргументов?**
A: Повторно возбуждает текущее обрабатываемое исключение, сохраняя оригинальный traceback. Используется для логирования с последующим пробросом. Вне блока `except` «голый» `raise` даёт `RuntimeError: No active exception to re-raise`.

**Q: Как поймать сразу `KeyError` и `IndexError` одним блоком?**
A: Тремя способами: `except (KeyError, IndexError)` (кортеж), либо через общего родителя `except LookupError`, либо `except Exception` (слишком широко). Предпочтительнее явный кортеж или `LookupError`.

**Q: Имеет ли значение порядок блоков `except`?**
A: Да! Блоки проверяются сверху вниз, срабатывает первый подходящий. Поэтому более специфичные исключения должны идти ПЕРЕД более общими. `except Exception` перед `except ValueError` сделает второй блок недостижимым (в новых версиях это даже не предупреждается, но логика всё равно ломается).

```python
try:
    int("x")
except Exception:        # перехватит всё первым
    print("общий")
except ValueError:       # НИКОГДА не выполнится
    print("значение")
```

**Q: Что такое `ExceptionGroup` и `except*`? Зачем они нужны?**
A: `ExceptionGroup` (3.11+) — контейнер для нескольких исключений, возникших одновременно (типичный случай — `asyncio.TaskGroup`, где параллельно упало несколько задач). `except*` позволяет выборочно обрабатывать подмножество исключений из группы по типу; он может сработать несколько раз, а необработанные исключения переподнимаются как остаточная группа.

**Q: Чем `ModuleNotFoundError` отличается от `ImportError`?**
A: `ModuleNotFoundError` (3.6+) — подкласс `ImportError`, возбуждается именно когда модуль не найден. `ImportError` (родитель) возбуждается шире — например, когда модуль найден, но в нём нет запрашиваемого имени (`from os import nonexistent`).

**Q: Как `StopIteration` связан с генераторами и циклом `for`?**
A: Итератор сигнализирует об окончании, бросая `StopIteration`. Цикл `for` ловит его внутренне и завершается. В генераторах `return value` транслируется в `StopIteration(value)`, доступное через `.value`. Важно (PEP 479, начиная с 3.7): `StopIteration`, «утёкший» внутри генератора, превращается в `RuntimeError`, чтобы не маскировать ошибки.

**Q: Безопасно ли писать `except:` (голый except)?**
A: Нет. Голый `except:` ловит АБСОЛЮТНО всё, включая `KeyboardInterrupt` и `SystemExit`, делая программу непрерываемой и скрывая системные сигналы. Если нужна широкая обработка — используйте `except Exception:`.

## Подводные камни (gotchas)

### 1. Голый `except:` ловит слишком много

```python
# ПЛОХО: перехватит даже Ctrl+C и sys.exit()
try:
    long_running_task()
except:                  # noqa
    pass

# ХОРОШО:
try:
    long_running_task()
except Exception:
    log_and_continue()
```

### 2. Ловля `BaseException`

```python
# ПЛОХО: блокирует KeyboardInterrupt/SystemExit так же, как голый except
try:
    ...
except BaseException:
    ...
```

Ловите `BaseException` только в очень специфичных случаях (например, корневой обработчик, который ДОЛЖЕН выполнить cleanup, а затем непременно сделать `raise`).

### 3. `return` в `finally` «съедает» исключение и значение

```python
def trap():
    try:
        return "из try"
    finally:
        return "из finally"   # перезапишет всё, ДАЖЕ исключение!

print(trap())   # => из finally

def swallow():
    try:
        raise ValueError("важная ошибка")
    finally:
        return "проглочено"   # исключение бесследно исчезает!

print(swallow())  # => проглочено — ValueError потерян!
```

Никогда не используйте `return`/`break`/`continue` в `finally`.

### 4. Проглатывание исключений (silent failure)

```python
# ПЛОХО: ошибка исчезает без следа, отладка превращается в ад
try:
    risky()
except Exception:
    pass

# ЛУЧШЕ: хотя бы залогировать
import logging
try:
    risky()
except Exception:
    logging.exception("risky() упал")
```

### 5. Неправильный порядок блоков `except` (наследование)

Специфичное — перед общим. Иначе общий блок «затенит» специфичные (см. вопрос выше).

### 6. Изменяемый объект по умолчанию и состояние после исключения

```python
def append_safe(item, acc=[]):   # ОПАСНО: общий изменяемый дефолт
    acc.append(item)
    if item < 0:
        raise ValueError("отрицательное")
    return acc

# Даже при исключении acc уже изменён — состояние «грязное» между вызовами
```

При исключении частичные изменения состояния не откатываются автоматически. Если нужна атомарность — работайте с копией и фиксируйте результат только в конце (или используйте транзакции/контекстные менеджеры).

### 7. Потеря трейсбека при `raise e` вместо `raise`

```python
try:
    risky()
except Exception as e:
    raise e      # технически работает, но в старом коде путало трейсбек;
                 # ЛУЧШЕ просто raise (без аргумента) — сохраняет контекст чище
```

### 8. `except (ValueError)` — это НЕ кортеж

```python
# Это лишние скобки вокруг одного класса, работает как обычный except ValueError
except (ValueError):
    ...

# А вот это — кортеж из ДВУХ типов:
except (ValueError, TypeError):
    ...

# ОПАСНО в очень старом коде Python 2: except ValueError, e — это была
# привязка имени, в Python 3 синтаксис только: except ValueError as e
```

### 9. `StopIteration` внутри генератора → `RuntimeError` (PEP 479)

```python
def gen():
    yield next(iter([]))   # next() бросит StopIteration внутри генератора

list(gen())   # RuntimeError: generator raised StopIteration
```

### 10. Сравнение исключений по строке сообщения

```python
# ПЛОХО: хрупко, ломается при смене формулировки
except Exception as e:
    if "not found" in str(e):
        ...

# ХОРОШО: ловить по типу или анализировать errno/коды
except FileNotFoundError:
    ...
```

## Лучшие практики

- **Ловите максимально узко.** Перехватывайте конкретный тип (`except FileNotFoundError`), а не `Exception`, если знаете, чего ожидаете.
- **Никогда не используйте голый `except:`** — пишите минимум `except Exception:`.
- **Не «глотайте» исключения молча.** Минимум — логируйте через `logging.exception(...)`, который автоматически добавит трейсбек.
- **Создавайте иерархию пользовательских исключений** от одного базового класса приложения (`class AppError(Exception)`), чтобы можно было ловить «всю семью» одним блоком.
- **Используйте `raise ... from ...`** для сохранения причинно-следственной связи, и `from None` — чтобы скрыть нерелевантные внутренние детали.
- **Применяйте `else`**, чтобы сузить блок `try` только до кода, который реально может бросить исключение.
- **Не ставьте `return`/`break`/`continue` в `finally`.**
- **Используйте `finally` или контекстные менеджеры (`with`)** для гарантированного освобождения ресурсов.
- **Повторно поднимайте через «голый» `raise`**, чтобы не терять оригинальный трейсбек.
- **Не используйте исключения для штатного управления потоком** там, где хватает обычной логики (но `StopIteration` для итераторов — это норма Python).
- **Применяйте `add_note()` (3.11+)** для обогащения исключения контекстом без смены типа.
- **Для конкурентного кода (asyncio)** учитывайте `ExceptionGroup` и обрабатывайте через `except*`.

## Шпаргалка

```text
ИЕРАРХИЯ:
  BaseException
    ├─ KeyboardInterrupt, SystemExit, GeneratorExit  (НЕ ловятся except Exception!)
    └─ Exception
         ├─ ArithmeticError → ZeroDivisionError, OverflowError
         ├─ LookupError     → KeyError, IndexError
         ├─ OSError         → FileNotFoundError, PermissionError, ConnectionError, TimeoutError
         ├─ RuntimeError    → RecursionError, NotImplementedError
         ├─ ImportError     → ModuleNotFoundError
         ├─ ValueError, TypeError, AttributeError, NameError
         └─ StopIteration, StopAsyncIteration

ПОРЯДОК try:   try → (except | else) → finally   (finally ВСЕГДА)
  - else:    только при успехе try
  - finally: всегда; НЕ ставить тут return!

ЦЕПОЧКИ:
  raise X            → X.__context__ = текущее (неявно, "During handling...")
  raise X from Y     → X.__cause__   = Y       (явно, "...direct cause...")
  raise X from None  → подавить контекст (__suppress_context__ = True)
  raise (без арг.)   → re-raise текущего, сохраняя traceback

ВАЖНОЕ:
  - except узко; никогда голый except:
  - except Exception, НЕ BaseException
  - специфичные except ВЫШЕ общих
  - не глотать молча → logging.exception(...)
  - пользовательские → class AppError(Exception)

ТИПЫ ОШИБОК:
  TypeError  → неверный ТИП        ValueError → верный тип, неверное ЗНАЧЕНИЕ
  KeyError   → нет ключа в dict     IndexError → выход за границы списка
  AttributeError → нет атрибута     NameError  → имя не определено

3.11+:
  ExceptionGroup + except*  → группы исключений (asyncio.TaskGroup)
  e.add_note("...")         → доп. контекст в traceback (e.__notes__)
```
