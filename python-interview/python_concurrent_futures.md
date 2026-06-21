# concurrent.futures — подготовка к собеседованию

## Что это и зачем

`concurrent.futures` — это высокоуровневый модуль стандартной библиотеки Python (появился в 3.2), который предоставляет **единый, абстрактный интерфейс для асинхронного выполнения задач** через пулы исполнителей (executors). Он скрывает низкоуровневую работу с потоками (`threading`) и процессами (`multiprocessing`) за общим API.

Ключевая идея: вы отдаёте функцию и её аргументы исполнителю, а взамен получаете объект `Future` — «обещание» результата, который будет вычислен где-то в фоне. Затем вы можете:
- дождаться результата (`future.result()`),
- проверить статус (`done()`, `running()`),
- получить исключение (`exception()`),
- обработать результаты по мере готовности (`as_completed`).

Зачем это нужно:
- **Упрощение параллелизма.** Вместо ручного создания, запуска и join'а потоков/процессов вы работаете с пулом фиксированного размера, который сам управляет очередью задач.
- **Единый API для двух моделей.** `ThreadPoolExecutor` и `ProcessPoolExecutor` имеют одинаковый интерфейс, поэтому переключение между потоками и процессами — это смена одного класса.
- **I/O-bound vs CPU-bound.** Потоки эффективны для задач, ожидающих ввода-вывода (сеть, диск, БД), процессы — для тяжёлых вычислений, обходящих GIL.
- **Контроль ресурсов.** Пул ограничивает количество одновременно работающих воркеров (`max_workers`), что защищает от перегрузки системы.

Типичные сценарии: параллельная загрузка сотен URL, обработка пачки файлов, распараллеливание CPU-тяжёлых вычислений (хеширование, обработка изображений, числовые расчёты), батч-обращения к внешним API.

```python
from concurrent.futures import ThreadPoolExecutor
import urllib.request

# Простейший пример: параллельно скачиваем несколько страниц
urls = [
    "https://example.com",
    "https://python.org",
    "https://docs.python.org",
]

def fetch(url: str) -> int:
    # Возвращаем размер тела ответа в байтах
    with urllib.request.urlopen(url, timeout=10) as resp:
        return len(resp.read())

# Пул на 5 потоков; with гарантирует корректное завершение
with ThreadPoolExecutor(max_workers=5) as executor:
    # map применяет fetch к каждому url, сохраняя порядок результатов
    for url, size in zip(urls, executor.map(fetch, urls)):
        print(f"{url}: {size} байт")
```

---

## Ключевые концепции

**Executor (исполнитель)** — абстрактный базовый класс, у которого две конкретные реализации:
- `ThreadPoolExecutor` — пул потоков (один процесс, несколько потоков, общая память, ограничен GIL для CPU-кода).
- `ProcessPoolExecutor` — пул процессов (несколько процессов, изолированная память, обход GIL, но накладные расходы на сериализацию).

**Future (будущее / обещание)** — объект, представляющий результат вычисления, которое ещё может быть не завершено. Future проходит через состояния: `PENDING` (в очереди) → `RUNNING` (выполняется) → `FINISHED` (завершено с результатом или исключением) либо `CANCELLED` (отменено до запуска).

**submit** — ставит одну задачу в очередь и немедленно возвращает `Future`. Не блокирует. Даёт максимальный контроль над каждой задачей.

**map** — высокоуровневый аналог встроенного `map()`: применяет функцию к итерируемому и возвращает итератор результатов **в порядке поданных аргументов** (а не в порядке завершения). Исключения откладываются до момента итерации.

**as_completed** — принимает коллекцию futures и возвращает итератор, который отдаёт каждый future **по мере его завершения** (порядок завершения, а не подачи). Идеально, когда нужно обрабатывать результаты сразу, как только они готовы.

**wait** — блокирует до тех пор, пока не выполнится условие (`ALL_COMPLETED`, `FIRST_COMPLETED`, `FIRST_EXCEPTION`). Возвращает два множества: `done` и `not_done`. Не извлекает результаты — только ждёт.

**max_workers** — максимальное число одновременно активных воркеров. Для `ThreadPoolExecutor` по умолчанию `min(32, os.cpu_count() + 4)`. Для `ProcessPoolExecutor` — `os.cpu_count()`.

**Контекстный менеджер (`with`)** — при выходе из блока неявно вызывается `shutdown(wait=True)`, что гарантирует завершение всех задач и освобождение ресурсов.

**initializer** — функция, вызываемая один раз в каждом воркере при его создании (для настройки соединений, логирования, глобальных ресурсов).

**Pickling** — для `ProcessPoolExecutor` функции, аргументы и результаты должны сериализоваться через `pickle`, чтобы передаваться между процессами. Это накладывает важные ограничения.

---

## Основные функции/классы/методы

### ThreadPoolExecutor.submit() и объект Future

```python
from concurrent.futures import ThreadPoolExecutor, Future
import time

def slow_square(x: int) -> int:
    time.sleep(1)  # имитируем долгую операцию (например, сетевой запрос)
    return x * x

with ThreadPoolExecutor(max_workers=3) as executor:
    # submit немедленно возвращает Future, не дожидаясь вычисления
    future: Future = executor.submit(slow_square, 5)

    print(future.running())   # True или False — выполняется ли уже
    print(future.done())      # False — ещё не готово

    # result() БЛОКИРУЕТ до получения результата
    result = future.result()  # вернёт 25 примерно через 1 секунду
    print(result)             # 25
    print(future.done())      # True
```

### Методы Future: result / exception / cancel / done

```python
from concurrent.futures import ThreadPoolExecutor
import time

def task(x):
    time.sleep(2)
    if x == 0:
        raise ValueError("ноль недопустим")
    return 10 / x

with ThreadPoolExecutor(max_workers=2) as executor:
    f_ok = executor.submit(task, 5)
    f_err = executor.submit(task, 0)

    # result(timeout) — ждёт результат, при timeout кидает TimeoutError
    print(f_ok.result(timeout=5))     # 2.0

    # exception() — возвращает объект исключения (или None), НЕ перебрасывая его
    exc = f_err.exception()
    print(type(exc).__name__)         # ValueError
    print(str(exc))                   # ноль недопустим

    # А вот result() для упавшей задачи ПЕРЕБРОСИТ исключение
    try:
        f_err.result()
    except ValueError as e:
        print(f"Поймали: {e}")

    # cancel() — отменяет ТОЛЬКО ещё не запущенные (PENDING) задачи
    f_pending = executor.submit(task, 100)
    cancelled = f_pending.cancel()    # True, если успели отменить до запуска
    print(f"Отменено: {cancelled}")
    print(f_pending.cancelled())      # True, если cancel сработал
```

### map: порядок результатов и отложенные исключения

```python
from concurrent.futures import ThreadPoolExecutor

def process(n):
    if n == 3:
        raise RuntimeError(f"сбой на {n}")
    return n * 100

with ThreadPoolExecutor(max_workers=4) as executor:
    # map возвращает результаты В ПОРЯДКЕ ПОДАЧИ аргументов,
    # независимо от того, какая задача завершилась раньше
    results = executor.map(process, [1, 2, 3, 4, 5])

    # ВАЖНО: исключение из process(3) проявится ТОЛЬКО при итерации
    # до этого элемента, а не в момент вызова map
    try:
        for r in results:
            print(r)   # напечатает 100, 200, затем выбросит RuntimeError
    except RuntimeError as e:
        print(f"Ошибка при итерации: {e}")

# map с timeout — общий таймаут на ВСЮ итерацию
with ThreadPoolExecutor(max_workers=2) as executor:
    # timeout=5 означает: если за 5 сек не получить следующий результат — TimeoutError
    for r in executor.map(lambda x: x, range(3), timeout=5):
        print(r)
```

### as_completed: обработка по мере готовности

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time, random

def work(task_id):
    delay = random.uniform(0.1, 2.0)
    time.sleep(delay)
    return task_id, delay

with ThreadPoolExecutor(max_workers=5) as executor:
    # Часто используют dict {future: метаданные} для связи результата с задачей
    futures = {executor.submit(work, i): i for i in range(10)}

    # as_completed отдаёт futures по мере ЗАВЕРШЕНИЯ (не по порядку submit)
    for future in as_completed(futures):
        task_id = futures[future]   # узнаём, какая задача завершилась
        try:
            result_id, delay = future.result()
            print(f"Задача {task_id} готова за {delay:.2f}с")
        except Exception as e:
            print(f"Задача {task_id} упала: {e}")

    # as_completed тоже принимает timeout — на всю операцию ожидания
```

### wait: FIRST_COMPLETED / FIRST_EXCEPTION / ALL_COMPLETED

```python
from concurrent.futures import (
    ThreadPoolExecutor, wait,
    FIRST_COMPLETED, FIRST_EXCEPTION, ALL_COMPLETED,
)
import time

def job(x):
    time.sleep(x)
    return x

with ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(job, t) for t in (1, 2, 3, 4)]

    # ALL_COMPLETED (по умолчанию) — ждать завершения ВСЕХ
    done, not_done = wait(futures, return_when=ALL_COMPLETED)
    print(f"Готово: {len(done)}, осталось: {len(not_done)}")

with ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(job, t) for t in (1, 2, 3, 4)]

    # FIRST_COMPLETED — вернуться, как только завершится ХОТЯ БЫ ОДНА
    done, not_done = wait(futures, return_when=FIRST_COMPLETED, timeout=10)
    print(f"Первая готова. done={len(done)}, not_done={len(not_done)}")
    # Оставшиеся продолжат выполняться; их можно отменить или дождаться

with ThreadPoolExecutor(max_workers=5) as executor:
    def maybe_fail(x):
        time.sleep(x)
        if x == 2:
            raise ValueError("упал")
        return x

    futures = [executor.submit(maybe_fail, t) for t in (1, 2, 3, 4)]
    # FIRST_EXCEPTION — вернуться при первом исключении ИЛИ когда все готовы
    done, not_done = wait(futures, return_when=FIRST_EXCEPTION)
    print(f"Остановились на исключении: done={len(done)}")
```

### ProcessPoolExecutor для CPU-bound

```python
from concurrent.futures import ProcessPoolExecutor
import math

def is_prime(n: int) -> bool:
    # CPU-тяжёлая проверка простоты — выигрывает от процессов (обход GIL)
    if n < 2:
        return False
    for i in range(2, int(math.isqrt(n)) + 1):
        if n % i == 0:
            return False
    return True

# КРИТИЧЕСКИ ВАЖНО для ProcessPoolExecutor: защита точки входа
if __name__ == "__main__":
    numbers = [112272535095293, 112582705942171, 115280095190773, 11]
    with ProcessPoolExecutor(max_workers=4) as executor:
        for n, prime in zip(numbers, executor.map(is_prime, numbers)):
            print(f"{n}: {'простое' if prime else 'составное'}")
```

### initializer: настройка воркеров

```python
from concurrent.futures import ThreadPoolExecutor
import threading

# Глобальный ресурс на уровне воркера (например, соединение с БД)
_local = threading.local()

def init_worker(config):
    # Вызывается ОДИН раз при старте каждого воркера
    _local.connection = f"connection({config})"
    print(f"Воркер инициализирован: {_local.connection}")

def use_resource(x):
    # Используем ресурс, созданный в initializer
    return f"{_local.connection} -> {x}"

with ThreadPoolExecutor(
    max_workers=3,
    initializer=init_worker,
    initargs=("db_url",),   # аргументы для initializer
) as executor:
    results = list(executor.map(use_resource, range(5)))
    print(results)
# Если initializer бросит исключение — пул станет нерабочим (broken)
```

### shutdown(wait=...)

```python
from concurrent.futures import ThreadPoolExecutor
import time

executor = ThreadPoolExecutor(max_workers=2)
futures = [executor.submit(time.sleep, 2) for _ in range(4)]

# shutdown(wait=True) — дождаться завершения всех задач (поведение по умолчанию)
# shutdown(wait=False) — вернуться немедленно, но новые задачи уже не принимаются
# shutdown(cancel_futures=True) — отменить ещё не запущенные задачи (Python 3.9+)
executor.shutdown(wait=True)
print("Все задачи завершены")

# Эквивалент через with (предпочтительно):
with ThreadPoolExecutor(max_workers=2) as ex:
    ex.submit(time.sleep, 1)
# здесь неявно вызван shutdown(wait=True)
```

---

## Частые вопросы на собеседовании

**1. В чём разница между ThreadPoolExecutor и ProcessPoolExecutor? Когда что использовать?**

`ThreadPoolExecutor` использует потоки в рамках одного процесса — общая память, дешёвое создание, но ограничен **GIL** (Global Interpreter Lock): в один момент Python-байткод исполняет только один поток. Поэтому потоки эффективны для **I/O-bound** задач (сеть, диск, БД, ожидание ответа API), где поток большую часть времени ждёт и отпускает GIL.

`ProcessPoolExecutor` запускает отдельные процессы — каждый со своим интерпретатором и GIL, поэтому он **обходит GIL** и даёт настоящий параллелизм на нескольких ядрах. Подходит для **CPU-bound** задач (вычисления, хеширование, обработка изображений). Платой служат накладные расходы: запуск процессов, **сериализация (pickle)** аргументов и результатов, отсутствие общей памяти.

Правило: I/O ждёт — берите потоки; процессор считает — берите процессы.

**2. Чем отличается submit от map?**

- `submit(fn, *args)` ставит **одну** задачу и возвращает `Future` сразу. Даёт контроль над каждой задачей по отдельности (статус, отмена, индивидуальная обработка исключений). Хорошо комбинируется с `as_completed`.
- `map(fn, iterable)` применяет функцию ко всему итерируемому и возвращает **итератор результатов в порядке подачи аргументов**. Удобно, когда задачи однородны и порядок важен. Но: исключения откладываются до итерации, и нет лёгкого способа отменить отдельную задачу.

Главные отличия: `submit` отдаёт futures (порядок завершения через `as_completed`), `map` отдаёт результаты (порядок подачи). У `submit` исключение всплывает при `result()`, у `map` — при итерации до проблемного элемента.

**3. В каком порядке as_completed возвращает результаты, а в каком map?**

`as_completed` отдаёт futures **в порядке их завершения** — первый готовый приходит первым, независимо от порядка submit. `map` отдаёт результаты **строго в порядке поданных аргументов**, даже если последний элемент посчитался раньше первого (тогда итератор будет ждать первый). Если важна минимальная задержка обработки — `as_completed`; если важно соответствие входу-выходу — `map`.

**4. Как обрабатываются исключения, возникшие в воркере?**

Исключение не «теряется» и не падает в основном потоке немедленно — оно **сохраняется внутри Future**. Получить его можно тремя способами:
- `future.result()` — **перебросит** исключение в вызывающем коде (с сохранённым traceback воркера).
- `future.exception()` — **вернёт** объект исключения, не перебрасывая (или `None`).
- При `map` — исключение выбросится при **итерации** до соответствующего результата.

Если же исключение возникло в `initializer` — пул становится «сломанным» (broken), и все последующие задачи завершатся с `BrokenThreadPool`/`BrokenProcessPool`.

**5. Что делает метод cancel() и когда он не работает?**

`future.cancel()` пытается отменить задачу и возвращает `True`/`False`. Он работает **только если задача ещё не начала выполняться** (в состоянии `PENDING`). Если задача уже `RUNNING`, отменить её нельзя — `cancel()` вернёт `False`. В Python нет безопасного способа прервать уже выполняющуюся функцию извне. Для массовой отмены непринятых задач при завершении используют `shutdown(cancel_futures=True)` (3.9+).

**6. Зачем нужен контекстный менеджер with и что происходит при выходе?**

`with Executor(...) as ex:` гарантирует, что при выходе из блока (в т.ч. при исключении) будет вызван `shutdown(wait=True)`. Это означает: пул перестанет принимать новые задачи и **дождётся завершения всех уже отправленных**, после чего корректно освободит ресурсы (потоки/процессы). Без `with` легко забыть `shutdown()`, что приведёт к «висящим» потокам/процессам и незавершённой программе.

**7. Что делает wait и чем отличаются ALL_COMPLETED, FIRST_COMPLETED, FIRST_EXCEPTION?**

`wait(fs, return_when=...)` блокирует до выполнения условия и возвращает кортеж `(done, not_done)` множеств futures. Сам по себе он **не извлекает результаты**, только ждёт.
- `ALL_COMPLETED` (по умолчанию) — ждать завершения всех.
- `FIRST_COMPLETED` — вернуться, как только завершится хотя бы одна (успешно или с ошибкой).
- `FIRST_EXCEPTION` — вернуться при первом исключении; если исключений нет — ведёт себя как `ALL_COMPLETED`.

Полезно для паттернов «получить первый ответ» или «прервать всё при первой ошибке».

**8. Что такое max_workers и каковы значения по умолчанию?**

`max_workers` — верхний предел одновременно работающих воркеров. Задачи сверх лимита ждут в очереди. По умолчанию:
- `ThreadPoolExecutor`: `min(32, (os.cpu_count() or 1) + 4)` — баланс между I/O-параллелизмом и ограничением числа потоков.
- `ProcessPoolExecutor`: `os.cpu_count()` — обычно нет смысла иметь больше процессов, чем ядер для CPU-задач.

Для I/O-bound число воркеров можно делать заметно больше числа ядер; для CPU-bound — примерно по числу ядер.

**9. Зачем нужен initializer и initargs?**

`initializer` — функция, вызываемая **один раз при создании каждого воркера** (а не на каждую задачу). Используется для дорогой однократной настройки: открыть соединение с БД, настроить логирование, загрузить модель в память, установить глобальное состояние воркера. `initargs` — кортеж аргументов для неё. Важно: если `initializer` бросит исключение, весь пул становится нерабочим.

**10. Какие ограничения накладывает ProcessPoolExecutor из-за pickling?**

Для передачи в другой процесс функция, её аргументы и результат должны быть **picklable**. Из-за этого:
- Нельзя передавать **лямбды**, локальные/вложенные функции, методы динамических объектов — они не сериализуются.
- Целевая функция должна быть определена на верхнем уровне модуля.
- Аргументы и результаты не должны содержать непиклимых объектов (открытые файлы, сокеты, соединения, локи, генераторы).
- Большие аргументы/результаты дорого сериализовать и копировать между процессами.

В потоках (`ThreadPoolExecutor`) этих ограничений нет — память общая.

**11. Почему ProcessPoolExecutor требует if __name__ == "__main__"?**

На Windows и macOS (метод старта `spawn`) дочерние процессы **импортируют главный модуль заново**. Если код, создающий пул и отправляющий задачи, находится на верхнем уровне модуля (не под защитой `if __name__ == "__main__":`), каждый дочерний процесс при импорте снова попытается создать пул — возникнет **бесконечная рекурсия порождения процессов** или `RuntimeError`. Защита точки входа гарантирует, что код запуска выполнится только в главном процессе.

**12. Что произойдёт, если внутри задачи воркера ждать результат другой задачи того же пула?**

Можно получить **deadlock**. Если все воркеры заняты задачами, которые ждут (`.result()`) завершения других задач, поставленных в очередь того же пула, но для этих новых задач нет свободных воркеров — система замирает навсегда. Вложенное использование одного пула «сам в себя» опасно. Решение: не блокироваться внутри воркера на задачах того же пула; использовать отдельные пулы или иную декомпозицию.

**13. Как задать таймаут и что происходит при его истечении?**

- `future.result(timeout=N)` / `future.exception(timeout=N)` — ждать не дольше N секунд, иначе `TimeoutError`. **Важно:** сама задача при этом не отменяется и продолжает выполняться.
- `executor.map(fn, it, timeout=N)` — общий таймаут на получение каждого следующего результата при итерации.
- `wait(fs, timeout=N)` и `as_completed(fs, timeout=N)` — таймаут на всю операцию ожидания.

Таймаут на стороне Future не прерывает работу воркера — это лишь ограничение времени ожидания в вызывающем коде.

**14. Чем concurrent.futures отличается от asyncio?**

`concurrent.futures` основан на потоках/процессах ОС и подходит для блокирующего кода (синхронные функции). `asyncio` — кооперативная однопоточная конкурентность на корутинах (`async/await`) с одним event loop, эффективная для тысяч I/O-операций без накладных расходов на потоки. Их связывают: `loop.run_in_executor(executor, fn, *args)` позволяет запускать блокирующие функции из asyncio в пуле `concurrent.futures`, не блокируя event loop. Объекты Future у них разные (`concurrent.futures.Future` vs `asyncio.Future`).

---

## Подводные камни (gotchas)

**1. Исключения в map проявляются только при итерации.**
Вызов `executor.map(fn, data)` сам по себе **не бросает** исключение, даже если задача уже упала. Ошибка всплывёт лишь когда итератор дойдёт до проблемного элемента. Более того, поскольку `map` отдаёт результаты по порядку, исключение в первом элементе блокирует получение остальных.

```python
from concurrent.futures import ThreadPoolExecutor

def f(x):
    if x == 1:
        raise ValueError("сбой")
    return x

with ThreadPoolExecutor() as ex:
    results = ex.map(f, [0, 1, 2])   # здесь исключения НЕТ
    print(next(results))             # 0 — ок
    print(next(results))             # ВОТ ЗДЕСЬ выбросит ValueError
```

**2. Нельзя неправильно вкладывать ProcessPoolExecutor (рекурсивное порождение).**
Если создавать `ProcessPoolExecutor` или сабмитить задачи на верхнем уровне модуля без `if __name__ == "__main__":`, на Windows/macOS дочерние процессы при импорте модуля снова создадут пул — получится лавинообразное порождение процессов и краш. Также опасно вкладывать пул процессов внутрь задачи другого пула процессов без чёткого понимания семантики — это легко приводит к взрыву числа процессов.

```python
# ПЛОХО: код запуска на верхнем уровне
from concurrent.futures import ProcessPoolExecutor
# with ProcessPoolExecutor() as ex:   # <-- на Windows/macOS это сломается
#     ex.map(work, data)

# ПРАВИЛЬНО: защита точки входа
if __name__ == "__main__":
    with ProcessPoolExecutor() as ex:
        ex.map(work, data)
```

**3. Deadlock при ожидании внутри воркера своих же задач.**
Если функция-воркер сабмитит задачи в тот же пул и блокируется на их `result()`, а все воркеры заняты такими же ожидающими задачами — пул зависает навсегда (нет свободных воркеров, чтобы выполнить вложенные задачи).

```python
from concurrent.futures import ThreadPoolExecutor

executor = ThreadPoolExecutor(max_workers=2)

def outer(x):
    # ОПАСНО: ждём задачу того же пула изнутри воркера
    inner = executor.submit(lambda: x * 2)
    return inner.result()   # при нехватке воркеров — deadlock

# Если все 2 воркера заняты outer'ами, ждущими inner'ов,
# которым негде выполниться — вечная блокировка.
# Решение: отдельный пул для вложенных задач или иная архитектура.
```

**4. Необходимость if __name__ == "__main__" для ProcessPoolExecutor.**
Это не «совет», а требование на платформах со `spawn` (Windows, macOS по умолчанию с Python 3.8+). Без защиты точки входа дочерние процессы повторно исполнят модульный код запуска. Симптомы: `RuntimeError` про текущий процесс / freeze_support, либо бесконечное порождение процессов. На Linux со `fork` может «повезти», но полагаться на это нельзя — код должен быть переносимым.

**5. Лямбды и локальные функции непиклимы в ProcessPoolExecutor.**
`executor.submit(lambda x: x, 5)` в пуле процессов упадёт с ошибкой pickle. Целевые функции должны быть определены на уровне модуля. В потоках лямбды работают (память общая).

**6. result(timeout) не отменяет задачу.**
Истечение таймаута в `result()` бросает `TimeoutError`, но воркер продолжает крутить задачу в фоне, занимая ресурс пула. Таймаут — это про ожидание в вызывающем коде, а не про прерывание вычисления.

**7. cancel() бесполезен для уже запущенных задач.**
Отменить можно только задачи, ещё не покинувшие очередь. Долгие задачи невозможно прервать стандартными средствами — нужно встраивать собственные точки проверки флага отмены внутрь самой функции.

**8. Исключение в initializer ломает весь пул.**
Если `initializer` падает, пул переходит в broken-состояние, и все задачи завершаются с `BrokenThreadPool` / `BrokenProcessPool`. Инициализатор должен быть надёжным.

**9. Слишком много потоков для CPU-bound не ускоряет.**
Из-за GIL увеличение числа потоков для чисто вычислительных задач не даёт ускорения (а иногда замедляет из-за переключений контекста). Для CPU-bound нужны процессы.

**10. Накладные расходы процессов на мелких задачах.**
Если задача дёшева, а данные большие, стоимость pickle и межпроцессного копирования превысит выигрыш от параллелизма. Для мелких CPU-задач имеет смысл группировать их в батчи (`chunksize` в `map` у `ProcessPoolExecutor`).

---

## Лучшие практики

- **Используйте `with` вместо ручного `shutdown()`.** Контекстный менеджер гарантирует корректное завершение даже при исключениях.
- **Выбирайте пул по типу нагрузки.** I/O-bound → `ThreadPoolExecutor`; CPU-bound → `ProcessPoolExecutor`. Не используйте потоки для тяжёлых вычислений (GIL) и не плодите процессы для лёгкого I/O.
- **Связывайте future с метаданными через dict.** Паттерн `{executor.submit(fn, x): x for x in data}` в связке с `as_completed` позволяет узнать, какая задача завершилась.
- **Всегда оборачивайте `future.result()` в try/except.** Исключения воркеров перебрасываются именно тут — обрабатывайте их, чтобы одна упавшая задача не уронила весь цикл.
- **Защищайте точку входа для процессов.** Весь код запуска `ProcessPoolExecutor` — под `if __name__ == "__main__":`.
- **Делайте функции и аргументы picklable для процессов.** Никаких лямбд/локальных функций/непиклимых объектов; функции — на уровне модуля.
- **Подбирайте `max_workers` осознанно.** Для I/O можно больше ядер; для CPU — около числа ядер. Замеряйте, а не угадывайте.
- **Не блокируйтесь внутри воркера на задачах того же пула.** Это путь к deadlock.
- **Используйте `as_completed`, когда важна скорость реакции,** и `map`, когда важен порядок «вход = выход».
- **Используйте `chunksize` в `ProcessPoolExecutor.map`** для большого числа мелких задач, чтобы снизить накладные расходы на IPC.
- **Используйте `shutdown(cancel_futures=True)` (3.9+)** для быстрой отмены непринятых задач при завершении.
- **Применяйте `initializer`** для дорогой однократной настройки воркеров (соединения, логирование), но делайте его устойчивым к ошибкам.
- **Помните про таймауты:** `result(timeout)`, `wait(timeout)`, `as_completed(timeout)`, `map(timeout)` — они защищают вызывающий код, но не прерывают воркеры.

---

## Шпаргалка

```python
from concurrent.futures import (
    ThreadPoolExecutor, ProcessPoolExecutor,
    as_completed, wait,
    FIRST_COMPLETED, FIRST_EXCEPTION, ALL_COMPLETED,
)

# --- Создание пула ---
ThreadPoolExecutor(max_workers=N, initializer=fn, initargs=(...))   # I/O-bound
ProcessPoolExecutor(max_workers=N)                                  # CPU-bound

# --- Отправка задач ---
future = ex.submit(fn, *args, **kwargs)   # одна задача -> Future, не блокирует
results = ex.map(fn, iterable, timeout=T, chunksize=K)  # итератор в порядке подачи

# --- Методы Future ---
future.result(timeout=T)   # ждёт, перебрасывает исключение воркера
future.exception(timeout=T)# возвращает исключение или None (не бросает)
future.cancel()            # отменяет только PENDING -> True/False
future.cancelled()         # был ли отменён
future.running()           # выполняется сейчас
future.done()              # завершён (готов/упал/отменён)
future.add_done_callback(cb)  # колбэк по завершении

# --- Ожидание группы ---
for f in as_completed(futures, timeout=T):   # по мере завершения
    f.result()
done, not_done = wait(futures, timeout=T, return_when=ALL_COMPLETED)
#   return_when: ALL_COMPLETED | FIRST_COMPLETED | FIRST_EXCEPTION

# --- Завершение ---
ex.shutdown(wait=True)                     # дождаться всех (по умолчанию)
ex.shutdown(wait=False)                    # не ждать
ex.shutdown(cancel_futures=True)           # отменить непринятые (3.9+)

# --- Идиомы ---
with ThreadPoolExecutor() as ex:           # авто-shutdown(wait=True)
    futs = {ex.submit(fn, x): x for x in data}
    for f in as_completed(futs):
        try:
            print(futs[f], f.result())
        except Exception as e:
            print("ошибка:", e)

# --- ProcessPool: ОБЯЗАТЕЛЬНО ---
if __name__ == "__main__":
    with ProcessPoolExecutor() as ex:
        list(ex.map(cpu_fn, data))   # cpu_fn на уровне модуля, аргументы picklable
```

**Памятка различий:**

| Аспект | ThreadPoolExecutor | ProcessPoolExecutor |
|---|---|---|
| Нагрузка | I/O-bound | CPU-bound |
| GIL | ограничивает | обходит |
| Память | общая | изолированная |
| Накладные расходы | низкие | высокие (pickle, IPC) |
| Лямбды/локальные ф-ии | можно | нельзя (нужен pickle) |
| `if __name__=="__main__"` | не нужно | обязательно (spawn) |
| max_workers по умолчанию | min(32, cpu+4) | os.cpu_count() |

**Памятка submit vs map vs as_completed:**

| | submit | map | as_completed |
|---|---|---|---|
| Возвращает | Future | итератор результатов | итератор Future |
| Порядок | — | порядок подачи | порядок завершения |
| Исключение всплывает | при `result()` | при итерации | при `result()` |
| Отмена отдельной задачи | легко | трудно | легко |
```
