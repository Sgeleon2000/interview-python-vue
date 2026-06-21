# Traceback — подготовка к собеседованию

## Что это и зачем

Модуль `traceback` из стандартной библиотеки Python — это инструмент для **работы со стеком вызовов и трассировками исключений (traceback)**. Когда возникает исключение, Python формирует объект трассировки (`traceback object`), который содержит цепочку фреймов выполнения — от места возбуждения исключения вверх по стеку вызовов до того места, где исключение было перехвачено (или до самого верха, если не перехвачено).

Модуль `traceback` позволяет:

- **Программно извлекать, форматировать и печатать** трассировки исключений и стек вызовов (то самое сообщение `Traceback (most recent call last): ...`, которое вы видите при падении программы).
- **Получать traceback в виде строки** — критично для логирования, отправки в системы мониторинга (Sentry, ELK, Grafana Loki), записи в БД, отправки по сети.
- **Сериализовать трассировку** (`TracebackException`) для передачи между процессами или сохранения, потому что сами объекты `traceback` и `frame` **нельзя сериализовать через pickle** и они держат ссылки на живые объекты (риск утечек памяти).
- **Анализировать цепочки исключений** (`__cause__`, `__context__`, `raise ... from ...`).
- **Ходить по стеку** (`walk_tb`, `walk_stack`) и строить собственные форматтеры/отчёты об ошибках.

**Зачем это на собеседовании:** middle/senior-разработчик должен уметь правильно логировать исключения (не теряя стек), понимать разницу между `__cause__` и `__context__`, знать почему нельзя хранить traceback-объекты в долгоживущих структурах, уметь получить полный текст ошибки в строку для алертинга и понимать механику `sys.exc_info()` и `exc.__traceback__`.

---

## Ключевые концепции

### Объект traceback (frame chain)

Объект трассировки — это связный список фреймов. Каждый объект `traceback` (тип `types.TracebackType`) имеет атрибуты:

- `tb_frame` — объект фрейма (`frame`), где находится переменные, имя функции, ссылка на код.
- `tb_lineno` — номер строки, на которой произошла остановка в этом фрейме.
- `tb_next` — следующий объект traceback (вниз по стеку, к месту возбуждения).
- `tb_lasti` — индекс инструкции байткода.

Важно: traceback читается **снизу вверх** в выводе ("most recent call last"), но `tb_next` ведёт **вглубь** — от точки перехвата к точке возбуждения исключения.

### Три способа добраться до traceback

1. **`sys.exc_info()`** — возвращает кортеж `(type, value, traceback)` для **текущего обрабатываемого** исключения. Вне блока `except` вернёт `(None, None, None)`.
2. **`exc.__traceback__`** — начиная с Python 3, у каждого объекта исключения есть атрибут `__traceback__`, в котором лежит его трассировка. Это современный и предпочтительный способ.
3. **`raise`** внутри `except` повторно поднимает текущее исключение с его трассировкой.

### Цепочки исключений (exception chaining)

- **`__context__`** — устанавливается **автоматически**, когда исключение возникает во время обработки другого исключения. В выводе появляется фраза `During handling of the above exception, another exception occurred:`.
- **`__cause__`** — устанавливается **явно** через `raise NewError from original`. В выводе: `The above exception was the direct cause of the following exception:`.
- **`__suppress_context__`** — флаг, который ставится в `True` при `raise ... from None`, чтобы подавить вывод цепочки.

### StackSummary и FrameSummary

- **`FrameSummary`** — "лёгкое" представление одного фрейма: имя файла, номер строки, имя функции, текст строки кода, локальные переменные (опционально). В отличие от живого `frame`, **не держит ссылок на реальные объекты и легко сериализуется**.
- **`StackSummary`** — список из `FrameSummary` (подкласс `list`), умеет форматировать себя в текст.

### TracebackException

`TracebackException` — это **снимок (snapshot)** исключения целиком: тип, сообщение, стек (`StackSummary`), цепочка (`__cause__`/`__context__` тоже как `TracebackException`). Не держит ссылок на живые фреймы и объекты, **может быть сериализован** (с оговорками) и форматирован позже, в другом потоке/процессе. Используется внутри `concurrent.futures` и `multiprocessing` для переноса ошибок между процессами.

### Ограничение глубины (limit)

Почти все функции принимают параметр `limit`:
- Положительное число — взять не более `limit` фреймов.
- Отрицательное число — взять последние `limit` фреймов (ближайшие к месту ошибки).
- `None` — все фреймы (по умолчанию учитывается `sys.tracebacklimit`).

---

## Основные функции/классы/методы

### print_exc / print_exception / print_tb

```python
import sys
import traceback

def divide(a, b):
    return a / b

try:
    divide(1, 0)
except ZeroDivisionError:
    # 1) print_exc — самый частый способ: печатает ТЕКУЩЕЕ исключение
    #    (тип + сообщение + стек) в sys.stderr. Эквивалент того,
    #    что Python печатает сам при падении программы.
    traceback.print_exc()

    # 2) print_exception — то же, но нужно явно передать исключение.
    #    Современная сигнатура (Python 3.10+): print_exception(exc)
    exc = sys.exc_info()[1]            # объект исключения
    traceback.print_exception(exc)

    # Старая сигнатура (работает во всех версиях 3.x):
    etype, value, tb = sys.exc_info()
    traceback.print_exception(etype, value, tb)

    # 3) print_tb — печатает ТОЛЬКО стек (фреймы), без строки
    #    "TypeError: ..." с типом и сообщением.
    traceback.print_tb(exc.__traceback__)

    # Можно направить вывод в произвольный файловый объект через file=
    traceback.print_exc(file=sys.stdout)

    # Ограничить количество фреймов
    traceback.print_exc(limit=1)        # только верхний фрейм
    traceback.print_exc(limit=-1)       # только самый глубокий (место ошибки)
```

### format_exc / format_exception / format_tb (получить строку)

```python
import traceback

try:
    [][5]   # IndexError
except IndexError:
    # 1) format_exc() -> str: ВЕСЬ traceback одной строкой.
    #    Самый удобный способ "получить traceback как строку" для логов.
    text: str = traceback.format_exc()
    print("=== Это строка для лога ===")
    print(text)

    # 2) format_exception(exc) -> list[str]: список строк
    #    (каждая обычно заканчивается \n). Соединяем через "".
    exc = ...  # см. ниже
    import sys
    exc = sys.exc_info()[1]
    lines: list[str] = traceback.format_exception(exc)
    text2 = "".join(lines)

    # 3) format_tb(tb) -> list[str]: только фреймы стека, без типа/сообщения
    frames: list[str] = traceback.format_tb(exc.__traceback__)
    print("".join(frames))

    # Ограничение глубины тоже поддерживается
    short = traceback.format_exc(limit=2)
```

> Запомнить: функции с приставкой **`print_`** пишут в поток (stderr по умолчанию), функции с приставкой **`format_`** возвращают строку/список строк. Хотите строку для лога — берите `format_*`.

### extract_tb / extract_stack -> StackSummary

```python
import traceback

def level_3():
    raise ValueError("упало здесь")

def level_2():
    level_3()

def level_1():
    level_2()

try:
    level_1()
except ValueError as exc:
    # extract_tb превращает traceback в StackSummary (список FrameSummary)
    summary = traceback.extract_tb(exc.__traceback__)
    print(type(summary))          # <class 'traceback.StackSummary'>

    for frame in summary:         # frame — это FrameSummary
        print(f"файл={frame.filename} строка={frame.lineno} "
              f"функция={frame.name} код={frame.line!r}")

    # StackSummary умеет форматировать себя сам
    print("".join(summary.format()))

# extract_stack — снимок ТЕКУЩЕГО стека вызовов (без исключения),
# полезно для отладки "как я сюда попал"
def show_where_i_am():
    stack = traceback.extract_stack()   # StackSummary до текущего места
    print("".join(stack.format()))

show_where_i_am()
```

### StackSummary и FrameSummary

```python
import traceback

def boom():
    local_var = 42                       # локальная переменная
    raise RuntimeError("ошибка")

try:
    boom()
except RuntimeError as exc:
    # StackSummary.extract принимает результат walk_tb/walk_stack.
    # capture_locals=True захватывает локальные переменные фреймов
    # (полезно для подробных отчётов об ошибках, но осторожно с PII!).
    summary = traceback.StackSummary.extract(
        traceback.walk_tb(exc.__traceback__),
        capture_locals=True,
    )

    for fs in summary:
        # FrameSummary — атрибуты: filename, lineno, name, line, locals
        print(f"{fs.name} @ {fs.filename}:{fs.lineno}")
        print(f"  строка кода: {fs.line!r}")
        if fs.locals:
            print(f"  локальные:  {fs.locals}")  # значения как repr-строки

    # Можно создать FrameSummary вручную (например, для синтетических отчётов)
    fake = traceback.FrameSummary("my_file.py", 10, "my_func", line="x = 1")
    ss = traceback.StackSummary.from_list([fake])
    print("".join(ss.format()))
```

### TracebackException (capture, format, сериализация/межпроцессный перенос)

```python
import traceback
import pickle

def make_error():
    try:
        1 / 0
    except ZeroDivisionError as e:
        raise ValueError("обёрнутая ошибка") from e

try:
    make_error()
except ValueError as exc:
    # capture — снимок исключения целиком (тип, сообщение, стек, цепочка).
    # Не держит ссылок на живые фреймы -> безопасно хранить долго.
    te = traceback.TracebackException.from_exception(exc, capture_locals=False)

    # format() возвращает генератор строк готового вывода (с цепочкой!)
    text = "".join(te.format())
    print(text)

    # format_exception_only() — только тип и сообщение, без стека
    print("".join(te.format_exception_only()))  # ValueError: обёрнутая ошибка

    # Доступ к полям снимка:
    print(te.exc_type_str)   # 'ValueError' (в 3.13+; ранее te.exc_type)
    print(te.stack)          # StackSummary
    print(te.__cause__)      # вложенный TracebackException (ZeroDivisionError)

    # --- Сериализация / перенос между процессами ---
    # Сам exc/traceback нельзя надёжно запиклить, а TracebackException —
    # можно (если в нём нет несериализуемых локальных переменных).
    blob = pickle.dumps(te)
    te_restored = pickle.loads(blob)
    print("После восстановления:\n", "".join(te_restored.format()))
```

> Именно `TracebackException` стоит за тем, как `concurrent.futures.ProcessPoolExecutor` и `multiprocessing` доставляют текст ошибки из дочернего процесса в родительский: живой traceback не пиклится, а его текстовый снимок — да.

### sys.exc_info() и атрибут __traceback__

```python
import sys
import traceback

try:
    raise KeyError("нет ключа")
except KeyError as exc:
    # Способ 1 (современный): берём traceback прямо из исключения
    tb = exc.__traceback__
    print("__traceback__:", tb)               # <traceback object ...>

    # Способ 2 (классический): sys.exc_info() -> (type, value, traceback)
    etype, evalue, etb = sys.exc_info()
    print(etype, evalue is exc, etb is tb)     # <class 'KeyError'> True True

# ВНЕ блока except sys.exc_info() вернёт (None, None, None)
print(sys.exc_info())   # (None, None, None)

# with_traceback — приклеить конкретный traceback к исключению вручную
def re_raise_with_context():
    try:
        1 / 0
    except ZeroDivisionError as e:
        new = ValueError("преобразованная")
        raise new.with_traceback(e.__traceback__)
```

### Цепочки исключений: __cause__, __context__, raise from, chain

```python
import traceback

# 1) __context__ — НЕЯВНАЯ цепочка (ошибка во время обработки ошибки)
def implicit_chain():
    try:
        1 / 0
    except ZeroDivisionError:
        raise ValueError("вторичная")   # __context__ = ZeroDivisionError

try:
    implicit_chain()
except ValueError as exc:
    print("context:", exc.__context__)      # ZeroDivisionError(...)
    print("cause:", exc.__cause__)          # None
    # В выводе: "During handling of the above exception..."
    print("".join(traceback.format_exception(exc)))

# 2) __cause__ — ЯВНАЯ цепочка через raise ... from ...
def explicit_chain():
    try:
        1 / 0
    except ZeroDivisionError as e:
        raise ValueError("вторичная") from e   # __cause__ = e

try:
    explicit_chain()
except ValueError as exc:
    print("cause:", exc.__cause__)          # ZeroDivisionError(...)
    # В выводе: "The above exception was the direct cause..."

# 3) raise ... from None — подавить цепочку (__suppress_context__ = True)
def suppressed():
    try:
        1 / 0
    except ZeroDivisionError:
        raise ValueError("чисто") from None   # цепочка скрыта в выводе

# 4) Параметр chain=False — НЕ печатать/форматировать цепочку
try:
    explicit_chain()
except ValueError as exc:
    # chain=True (по умолчанию) — печатает всю цепочку,
    # chain=False — только верхнее исключение.
    print("".join(traceback.format_exception(exc, chain=False)))
```

### walk_tb / walk_stack

```python
import sys
import traceback

def deep():
    raise RuntimeError("boom")

try:
    deep()
except RuntimeError as exc:
    # walk_tb — генератор пар (frame, lineno) ВНИЗ по traceback
    # (от места перехвата к месту возбуждения).
    for frame, lineno in traceback.walk_tb(exc.__traceback__):
        print(f"walk_tb: {frame.f_code.co_name} строка {lineno}")

# walk_stack — генератор (frame, lineno) ВВЕРХ по стеку вызовов
# от заданного фрейма к корню. None -> начать с фрейма вызывающего.
def who_called_me():
    for frame, lineno in traceback.walk_stack(None):
        print(f"walk_stack: {frame.f_code.co_name} строка {lineno}")

def caller():
    who_called_me()

caller()

# Эти генераторы — то, что StackSummary.extract принимает на вход.
```

### print_stack (текущий стек без исключения)

```python
import traceback

def a():
    b()

def b():
    # print_stack печатает ТЕКУЩИЙ стек вызовов (не исключение!) в stderr.
    # Удобно для отладки: "как управление дошло до этой точки".
    traceback.print_stack()
    # Аналогично есть format_stack() -> list[str] для получения строки
    text = "".join(traceback.format_stack())

a()
```

### Как залогировать traceback (logging)

```python
import logging

logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
logger = logging.getLogger(__name__)

def risky():
    return 1 / 0

# Способ 1: logging.exception() — ТОЛЬКО внутри блока except.
# Автоматически добавляет traceback текущего исключения. Уровень ERROR.
try:
    risky()
except ZeroDivisionError:
    logging.exception("Не удалось поделить")   # лог + traceback

# Способ 2: logger.error(..., exc_info=True) — то же, но управляем уровнем.
try:
    risky()
except ZeroDivisionError:
    logger.error("Ошибка деления", exc_info=True)

# Способ 3: передать конкретное исключение в exc_info (3.5+)
try:
    risky()
except ZeroDivisionError as exc:
    logger.warning("Поймали, но продолжаем", exc_info=exc)

# Способ 4: вручную получить строку и залогировать как обычный текст
import traceback
try:
    risky()
except ZeroDivisionError:
    logger.error("Ручное логирование:\n%s", traceback.format_exc())
```

### Как получить traceback в виде строки (сводка способов)

```python
import io
import sys
import traceback

try:
    1 / 0
except ZeroDivisionError as exc:
    # Вариант A: самый короткий
    s1 = traceback.format_exc()

    # Вариант B: из объекта исключения (без зависимости от sys.exc_info)
    s2 = "".join(traceback.format_exception(exc))

    # Вариант C: через TracebackException (для последующей сериализации)
    s3 = "".join(traceback.TracebackException.from_exception(exc).format())

    # Вариант D: перенаправить print_exc в StringIO
    buf = io.StringIO()
    traceback.print_exc(file=buf)
    s4 = buf.getvalue()

    assert s1.strip() == s4.strip()
```

---

## Частые вопросы на собеседовании

**Q1: В чём разница между `print_exc()` и `format_exc()`?**
A: `print_exc()` печатает трассировку текущего исключения в поток (по умолчанию `sys.stderr`) и возвращает `None`. `format_exc()` ничего не печатает, а **возвращает строку** с тем же содержимым. Для логирования и отправки в системы мониторинга нужен именно `format_exc()`, потому что нам нужна строка, а не вывод в stderr.

**Q2: Как получить traceback в виде строки?**
A: Чаще всего — `traceback.format_exc()` внутри блока `except`. Альтернативы: `"".join(traceback.format_exception(exc))` (из самого объекта исключения, не зависит от того, активен ли блок except) или через `TracebackException.from_exception(exc)` с последующим `format()`.

**Q3: Чем отличаются `__cause__` и `__context__`?**
A: `__context__` ставится **автоматически**, когда новое исключение возникает внутри обработки старого (`except`), и в выводе печатается "During handling of the above exception...". `__cause__` ставится **явно** через `raise New from old`, и печатается "The above exception was the direct cause...". `raise ... from None` подавляет вывод контекста (`__suppress_context__ = True`).

**Q4: Что вернёт `sys.exc_info()` вне блока `except`?**
A: Кортеж `(None, None, None)`. `sys.exc_info()` отдаёт данные только о **текущем обрабатываемом** исключении. Современная альтернатива — взять `exc.__traceback__` у самого объекта исключения.

**Q5: Почему нельзя хранить объект traceback в долгоживущей структуре (кэше, очереди, глобале)?**
A: Объект traceback держит ссылки на **живые фреймы**, а фреймы — на все локальные переменные и через них на большие объекты. Это вызывает **утечки памяти** и продлевает жизнь объектов. Кроме того, traceback и frame **не сериализуются через pickle**. Решение — сделать снимок через `TracebackException` или `extract_tb` (получаем `StackSummary`), которые хранят только текст и не держат живых ссылок.

**Q6: Как правильно логировать исключение?**
A: Внутри `except` использовать `logging.exception("сообщение")` (уровень ERROR, traceback добавляется автоматически) или `logger.error("...", exc_info=True)` для контроля уровня. Не нужно вручную форматировать и конкатенировать в само сообщение — `exc_info` сделает это правильно и единообразно. Главное правило: **никогда не глотать исключение без логирования стека** (`except: pass` — антипаттерн).

**Q7: Что делает параметр `limit`?**
A: Ограничивает число фреймов. Положительное значение — первые N фреймов (от верха стека), отрицательное — последние N (ближайшие к месту ошибки), `None` — все (с учётом `sys.tracebacklimit`). Полезно, чтобы укоротить шумные трассировки в логах.

**Q8: Зачем нужен `TracebackException` и где он применяется в стандартной библиотеке?**
A: Это сериализуемый снимок исключения вместе с цепочкой и стеком, не держащий живых ссылок. Применяется для **переноса ошибок между процессами**: `concurrent.futures.ProcessPoolExecutor` и `multiprocessing` используют его, чтобы доставить текст ошибки из дочернего процесса в родительский, ведь живой traceback запиклить нельзя.

**Q9: В чём разница между `extract_tb` и `walk_tb`?**
A: `walk_tb(tb)` — низкоуровневый генератор пар `(frame, lineno)`, отдающий **живые фреймы**. `extract_tb(tb)` строит из traceback `StackSummary` — список лёгких `FrameSummary` (только текст: файл, строка, функция, код), без живых ссылок. На практике `extract_tb` = `StackSummary.extract(walk_tb(tb))`.

**Q10: Как залогировать исключение, которое уже не является "текущим" (вне except)?**
A: Передать сам объект исключения: `logger.error("msg", exc_info=exc)` (поддерживается с 3.5) или `"".join(traceback.format_exception(exc))`. Поскольку у объекта есть `exc.__traceback__`, мы не зависим от `sys.exc_info()`.

**Q11: Что такое `print_tb` и чем он отличается от `print_exception`?**
A: `print_tb` печатает **только фреймы стека**, без строки с типом и сообщением исключения. `print_exception` печатает полную картину: стек + строку вида `ValueError: ...` + цепочку исключений. То же различие у `format_tb` vs `format_exception`.

**Q12: Как добавить в отчёт об ошибке значения локальных переменных фреймов?**
A: Через `capture_locals=True` в `StackSummary.extract(...)` или `TracebackException.from_exception(exc, capture_locals=True)`. Локальные сохраняются как их `repr()`-строки. Осторожно: это может раскрыть пароли/токены/персональные данные в логах — на проде применять с фильтрацией.

**Q13: Что делает `raise` без аргументов внутри `except`?**
A: Повторно поднимает **текущее** исключение с его исходным traceback (стек не теряется). Это правильный способ "перебросить выше" пойманное исключение, в отличие от `raise exc`, который при наличии может перезаписать строку traceback (хотя в 3.x исходный стек обычно сохраняется через `__traceback__`).

**Q14: Чем `walk_stack` отличается от `walk_tb` по направлению?**
A: `walk_tb` идёт **вниз** по traceback — от места перехвата к месту возбуждения исключения. `walk_stack` идёт **вверх** по стеку вызовов — от текущего/заданного фрейма к корню (кто меня вызвал). `walk_stack(None)` начинает с фрейма вызывающего.

---

## Подводные камни (gotchas)

- **Хранение traceback-объектов = утечка памяти.** Объект traceback держит фреймы, фреймы — локальные переменные, через них — большие объекты. Если положить `exc.__traceback__` или `sys.exc_info()[2]` в кэш/очередь/глобальную переменную, всё это дерево не освободится. Делайте снимок (`TracebackException` / `extract_tb`).

- **`sys.exc_info()` вне `except` возвращает `(None, None, None)`.** Не полагайтесь на него в коллбэках/тредах, выполняющихся после выхода из обработчика. Передавайте сам объект исключения.

- **Циклическая ссылка через `e` в Python 3.** Имя из `except ... as e:` после выхода из блока **удаляется** (`del e`), потому что исключение через `__traceback__` ссылается на фрейм, а фрейм — на `e`. Если нужно сохранить исключение наружу, присвойте его другой переменной до выхода из `except`.

- **`logging.exception()` работает только внутри `except`.** Вызванный вне обработчика, он залогирует "NoneType: None" вместо стека. Вне except используйте `exc_info=exc`.

- **`raise exc` против `raise from`.** Если в обработчике сделать `raise SomethingElse(...)` без `from`, Python всё равно прицепит исходное как `__context__` (неявная цепочка). Чтобы показать причинно-следственную связь явно — используйте `from e`. Чтобы скрыть шумный контекст — `from None`.

- **`print_exc()` пишет в stderr, а не в логи.** В продакшене это часто значит, что ошибка уйдёт "в никуда" или в неструктурированный поток. Используйте `logging`.

- **`capture_locals=True` раскрывает секреты.** Локальные переменные могут содержать пароли, токены, ключи. В отчётах для пользователей/в общих логах это утечка чувствительных данных.

- **`limit` с положительным числом обрезает не тот конец.** Положительный `limit` оставляет фреймы **сверху** (внешние вызовы), а самое интересное — место ошибки — внизу. Если хотите оставить место ошибки, используйте **отрицательный** `limit`.

- **Смена сигнатуры `print_exception`/`format_exception` в 3.10.** Раньше требовались три аргумента `(type, value, tb)`, с 3.10 можно передать один объект `exc`. Старая форма всё ещё поддерживается, но в новом коде предпочтительнее однопараметрическая.

- **`exc_type` у `TracebackException` устарел.** В Python 3.13 атрибут `exc_type` депрекейтнут в пользу строкового `exc_type_str`, потому что хранение живого класса исключения мешало сериализации.

- **Не пиклится traceback напрямую.** Попытка `pickle.dumps(exc)` или `pickle.dumps(tb)` упадёт/потеряет traceback. Для передачи между процессами сериализуйте `TracebackException` (и то — без несериализуемых локальных).

---

## Лучшие практики

1. **Логируйте стек, а не голое сообщение.** Внутри `except` всегда `logging.exception(...)` или `logger.error(..., exc_info=True)`. Никогда не `except: pass` без причины и без записи.

2. **Не теряйте причину.** При перевозбуждении делайте `raise НовоеИсключение(...) from original`, чтобы сохранить причинно-следственную цепочку для отладки.

3. **Для повторного выброса того же исключения — `raise` без аргументов.** Это сохраняет исходный traceback.

4. **Для хранения/переноса ошибки делайте снимок.** Используйте `TracebackException.from_exception(exc)` или `extract_tb`, а не сами объекты traceback/frame. Это и безопасно по памяти, и сериализуемо.

5. **Получаете строку — берите `format_*`, печатаете в поток — `print_*`.** Не изобретайте `StringIO`-обёртки там, где есть `format_exc()`.

6. **Управляйте уровнем логирования осознанно.** `logging.exception` всегда ERROR; если ошибка ожидаема и обрабатывается — логируйте на WARNING/INFO через `logger.log(level, ..., exc_info=True)`.

7. **Не включайте `capture_locals=True` в общих логах** без маскирования секретов. Применяйте только в защищённых диагностических каналах.

8. **Укорачивайте шумные трассировки через `limit`** (обычно отрицательный, чтобы оставить место ошибки), но не теряйте важный контекст.

9. **В новом коде используйте однопараметрические формы** `print_exception(exc)` / `format_exception(exc)` и атрибут `exc.__traceback__` вместо `sys.exc_info()`.

10. **Для структурированного логирования** (JSON-логи в Sentry/ELK) кладите `traceback.format_exc()` в поле и не смешивайте со свободным текстом сообщения.

---

## Шпаргалка

```text
ПОЛУЧИТЬ TRACEBACK:
  sys.exc_info()                -> (type, value, traceback)  [только в except]
  exc.__traceback__             -> traceback объекта исключения (предпочтительно)
  exc.with_traceback(tb)        -> приклеить tb к исключению

ПЕЧАТЬ В ПОТОК (по умолчанию stderr):
  traceback.print_exc()                 # текущее исключение целиком
  traceback.print_exception(exc)        # заданное исключение целиком (3.10+)
  traceback.print_tb(tb)                # только фреймы стека
  traceback.print_stack()               # текущий стек (без исключения)

ПОЛУЧИТЬ СТРОКУ:
  traceback.format_exc()         -> str        # текущее исключение
  traceback.format_exception(e)  -> list[str]  # "".join(...) -> str
  traceback.format_tb(tb)        -> list[str]  # только фреймы
  traceback.format_stack()       -> list[str]  # текущий стек

ИЗВЛЕЧЬ СТРУКТУРУ (StackSummary -> [FrameSummary]):
  traceback.extract_tb(tb)       -> StackSummary  # из traceback
  traceback.extract_stack()      -> StackSummary  # из текущего стека
  StackSummary.extract(walk_tb(tb), capture_locals=True)

ХОДИТЬ ПО ФРЕЙМАМ (генераторы (frame, lineno)):
  traceback.walk_tb(tb)          # вниз: перехват -> возбуждение
  traceback.walk_stack(frame)    # вверх: текущий -> корень (None = вызывающий)

СНИМОК ДЛЯ СЕРИАЛИЗАЦИИ/ПЕРЕНОСА:
  te = traceback.TracebackException.from_exception(exc, capture_locals=False)
  "".join(te.format())                  # полный текст с цепочкой
  "".join(te.format_exception_only())   # только "Type: message"
  pickle.dumps(te)                      # можно передать в другой процесс

ЦЕПОЧКИ ИСКЛЮЧЕНИЙ:
  exc.__context__       # неявная (ошибка во время обработки ошибки)
  exc.__cause__         # явная (raise New from original)
  raise New from orig   # установить __cause__
  raise New from None   # подавить вывод цепочки (__suppress_context__)
  format_exception(exc, chain=False)   # не выводить цепочку

LIMIT:
  limit > 0   -> первые N фреймов (внешние, от верха)
  limit < 0   -> последние N фреймов (внутренние, у места ошибки)
  limit = None-> все (с учётом sys.tracebacklimit)

ЛОГИРОВАНИЕ (внутри except):
  logging.exception("msg")              # ERROR + traceback автоматически
  logger.error("msg", exc_info=True)    # любой уровень + traceback
  logger.error("msg", exc_info=exc)     # передать конкретное исключение (3.5+)
```
