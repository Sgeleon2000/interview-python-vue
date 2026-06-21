# datetime — подготовка к собеседованию

## Что это и зачем

`datetime` — стандартный модуль Python для работы с датами и временем. Он предоставляет
классы для представления моментов во времени, интервалов между ними, а также средства
для парсинга, форматирования и арифметики над датами.

Зачем это нужно:

- Хранение и сравнение моментов времени (логи, события, дедлайны).
- Вычисление интервалов («сколько прошло», «сколько осталось»).
- Корректная работа с часовыми поясами (timezone), переходом на летнее время (DST).
- Конвертация между человекочитаемыми строками и объектами времени.

Главная идея, которую проверяют на собеседовании: **понимание разницы между «наивным»
(naive) и «осведомлённым» (aware) datetime**, а также то, что для корректного хранения
времени в системах почти всегда нужно UTC + явный часовой пояс.

```python
from datetime import datetime, timezone

# Наивный datetime — без информации о часовом поясе
naive = datetime(2026, 6, 20, 15, 30, 0)
print(naive)            # => 2026-06-20 15:30:00
print(naive.tzinfo)     # => None

# Осведомлённый (aware) datetime — с часовым поясом
aware = datetime(2026, 6, 20, 15, 30, 0, tzinfo=timezone.utc)
print(aware)            # => 2026-06-20 15:30:00+00:00
print(aware.tzinfo)     # => UTC
```

---

## Ключевые концепции

Модуль содержит пять основных типов:

| Класс       | Что представляет                                   | Пример                          |
|-------------|----------------------------------------------------|---------------------------------|
| `date`      | Дата (год, месяц, день)                            | `2026-06-20`                    |
| `time`      | Время суток (час, минута, секунда, микросекунда)   | `15:30:00`                      |
| `datetime`  | Дата + время вместе                                | `2026-06-20 15:30:00`           |
| `timedelta` | Длительность / разница между двумя моментами       | `5 days, 3:00:00`               |
| `tzinfo`    | Абстрактный базовый класс для часовых поясов       | `timezone.utc`, `ZoneInfo(...)` |
| `timezone`  | Конкретная реализация `tzinfo` с фиксированным сдвигом | `timezone(timedelta(hours=3))` |

Концептуально важные моменты:

1. **Все объекты `date`, `time`, `datetime`, `timedelta` неизменяемы (immutable).**
   Любая «модификация» (`replace`, арифметика) возвращает новый объект.

2. **Наивные vs осведомлённые объекты.**
   - *Naive* — не содержит информации о часовом поясе (`tzinfo is None`). Представляет
     «локальное» время без привязки к точке на глобусе.
   - *Aware* — содержит `tzinfo`, однозначно определяет момент во времени (UTC-инстант).

3. **Внутреннее представление.** `datetime` хранит «настенное время» (wall clock) —
   календарные компоненты. UTC-инстант вычисляется из них плюс `tzinfo`.

4. **Эпоха (epoch).** Timestamp — это число секунд (float) с 1970-01-01 00:00:00 UTC
   (Unix epoch).

```python
from datetime import date, time, datetime, timedelta

print(date(2026, 6, 20))                 # => 2026-06-20
print(time(15, 30, 45, 123456))          # => 15:30:45.123456
print(datetime(2026, 6, 20, 15, 30))     # => 2026-06-20 15:30:00
print(timedelta(days=5, hours=3))        # => 5 days, 3:00:00
```

---

## Основные функции/классы/методы

### Создание объектов

```python
from datetime import datetime, date, time

# Явное создание
dt = datetime(2026, 6, 20, 15, 30, 45, 123456)
print(dt)                # => 2026-06-20 15:30:45.123456

# Только обязательные аргументы — год, месяц, день
d = date(2026, 6, 20)
print(d)                 # => 2026-06-20

t = time(15, 30)
print(t)                 # => 15:30:00

# Извлечь компоненты
print(dt.year, dt.month, dt.day)         # => 2026 6 20
print(dt.hour, dt.minute, dt.second)     # => 15 30 45
print(dt.microsecond)                    # => 123456
print(dt.weekday())      # => 5  (понедельник=0 ... воскресенье=6)
print(dt.isoweekday())   # => 6  (понедельник=1 ... воскресенье=7)
```

### `now()` vs `utcnow()` vs `today()`

```python
from datetime import datetime, timezone

# now() — локальное время системы. Без аргумента -> НАИВНЫЙ объект
local_naive = datetime.now()
print(local_naive.tzinfo)        # => None

# now(tz) — с часовым поясом -> ОСВЕДОМЛЁННЫЙ объект (рекомендуемый способ)
now_utc = datetime.now(timezone.utc)
print(now_utc.tzinfo)            # => UTC

# utcnow() — ВОЗВРАЩАЕТ НАИВНЫЙ объект с UTC-значениями. DEPRECATED с Python 3.12!
# old = datetime.utcnow()        # DeprecationWarning в 3.12+
# Проблема: значение «как в UTC», но tzinfo=None -> легко перепутать с локальным.

# today() — то же, что now() без tz: локальное наивное время
print(datetime.today().tzinfo)   # => None
```

Почему `utcnow()` объявлен устаревшим в 3.12: он возвращал наивный объект, чьё значение
соответствует UTC, но без `tzinfo`. Это источник массы багов: такой объект при
сравнении/вычитании с локальным наивным временем даёт неверный результат, а
`.timestamp()` на нём интерпретирует время как локальное. Правильная замена:
`datetime.now(timezone.utc)`.

### `fromtimestamp()` / `timestamp()`

```python
from datetime import datetime, timezone

ts = 1_780_000_000  # секунды от Unix epoch

# fromtimestamp() БЕЗ tz -> локальное наивное время
local = datetime.fromtimestamp(ts)
print(local)                 # => зависит от часового пояса системы

# fromtimestamp(ts, tz) -> осведомлённое время в нужной зоне (правильный способ)
utc = datetime.fromtimestamp(ts, tz=timezone.utc)
print(utc)                   # => 2026-05-28 08:26:40+00:00

# utcfromtimestamp() — тоже DEPRECATED в 3.12 (наивный результат). Не использовать.

# Обратно в timestamp
dt = datetime(2026, 5, 28, 8, 26, 40, tzinfo=timezone.utc)
print(dt.timestamp())        # => 1780000000.0

# Важно: timestamp() на НАИВНОМ объекте трактует его как ЛОКАЛЬНОЕ время системы!
naive = datetime(2026, 5, 28, 8, 26, 40)
print(naive.timestamp())     # => число зависит от TZ системы (НЕ детерминировано)
```

### `timedelta` — арифметика длительностей

```python
from datetime import datetime, timedelta

d1 = datetime(2026, 6, 20, 12, 0, 0)
d2 = datetime(2026, 6, 25, 18, 30, 0)

# Разница двух datetime -> timedelta
delta = d2 - d1
print(delta)                 # => 5 days, 6:30:00
print(delta.days)            # => 5
print(delta.seconds)         # => 23400  (остаток секунд ВНУТРИ дня, не всего)
print(delta.total_seconds()) # => 455400.0  (вся длительность в секундах)

# Прибавление/вычитание timedelta к datetime -> datetime
print(d1 + timedelta(weeks=2))           # => 2026-07-04 12:00:00
print(d1 - timedelta(hours=36))          # => 2026-06-19 00:00:00

# Арифметика самих timedelta
print(timedelta(hours=10) * 3)           # => 1 day, 6:00:00
print(timedelta(days=1) / timedelta(hours=4))   # => 6.0
print(timedelta(days=1) // timedelta(hours=5))  # => 4

# timedelta нормализуется автоматически
print(timedelta(seconds=90))             # => 0:01:30
print(timedelta(hours=25))               # => 1 day, 1:00:00

# Внутреннее хранение: только days, seconds, microseconds
print(timedelta(minutes=1).seconds)      # => 60
```

### Сравнение и сортировка

```python
from datetime import datetime, timezone, timedelta

a = datetime(2026, 6, 20, 10, 0)
b = datetime(2026, 6, 20, 12, 0)
print(a < b)                 # => True
print(a == b)                # => False

# Сортировка списка дат
dates = [b, a, datetime(2026, 1, 1)]
print(sorted(dates))         # => [2026-01-01 ..., 2026-06-20 10:00, 2026-06-20 12:00]

# Сравнение aware-объектов идёт по UTC-инстанту, а не по «настенному» времени!
msk = timezone(timedelta(hours=3))
x = datetime(2026, 6, 20, 15, 0, tzinfo=msk)        # 12:00 UTC
y = datetime(2026, 6, 20, 12, 0, tzinfo=timezone.utc)
print(x == y)                # => True (один и тот же момент!)
```

### `replace()` — создать копию с изменёнными полями

```python
from datetime import datetime, timezone

dt = datetime(2026, 6, 20, 15, 30, 45)

# replace возвращает НОВЫЙ объект (immutable)
print(dt.replace(year=2030))             # => 2030-06-20 15:30:45
print(dt.replace(hour=0, minute=0, second=0, microsecond=0))  # => 2026-06-20 00:00:00

# ВНИМАНИЕ: replace(tzinfo=...) НЕ конвертирует время, а просто «приклеивает» зону!
naive = datetime(2026, 6, 20, 15, 0)
attached = naive.replace(tzinfo=timezone.utc)   # говорит «это и было UTC»
print(attached)              # => 2026-06-20 15:00:00+00:00
# Время компонентов осталось 15:00 — изменился только смысл (теперь это UTC).
```

### `strftime()` — форматирование в строку

```python
from datetime import datetime
import locale

dt = datetime(2026, 6, 20, 15, 30, 45, 123456)

print(dt.strftime("%Y-%m-%d"))           # => 2026-06-20
print(dt.strftime("%H:%M:%S"))           # => 15:30:45
print(dt.strftime("%Y-%m-%d %H:%M:%S.%f"))  # => 2026-06-20 15:30:45.123456
print(dt.strftime("%d.%m.%Y"))           # => 20.06.2026
print(dt.strftime("%j"))                 # => 171  (день года, 001-366)
print(dt.strftime("%A %a"))              # => Saturday Sat  (зависит от локали)
print(dt.strftime("%B %b"))              # => June Jun
print(dt.strftime("%p"))                 # => PM
print(dt.strftime("%I"))                 # => 03  (час в 12-часовом формате)
print(dt.strftime("%w"))                 # => 6   (день недели, 0=воскресенье)
print(dt.strftime("%W"))                 # => 24  (номер недели в году)
```

Самые частые коды форматирования:

| Код  | Значение                              | Пример       |
|------|---------------------------------------|--------------|
| `%Y` | Год, 4 цифры                          | `2026`       |
| `%y` | Год, 2 цифры                          | `26`         |
| `%m` | Месяц, 01-12                          | `06`         |
| `%d` | День месяца, 01-31                    | `20`         |
| `%H` | Часы 24ч, 00-23                       | `15`         |
| `%I` | Часы 12ч, 01-12                       | `03`         |
| `%M` | Минуты, 00-59                         | `30`         |
| `%S` | Секунды, 00-59                        | `45`         |
| `%f` | Микросекунды, 6 цифр                  | `123456`     |
| `%z` | UTC-сдвиг                             | `+0300`      |
| `%Z` | Имя часового пояса                    | `UTC`, `MSK` |
| `%A` | Полное имя дня недели                  | `Saturday`   |
| `%a` | Сокр. имя дня недели                   | `Sat`        |
| `%B` | Полное имя месяца                      | `June`       |
| `%b` | Сокр. имя месяца                       | `Jun`        |
| `%j` | День года, 001-366                    | `171`        |
| `%p` | AM/PM                                 | `PM`         |
| `%%` | Литеральный знак `%`                  | `%`          |

```python
# %z и %Z работают только на aware-объектах
from datetime import datetime, timezone, timedelta
aware = datetime(2026, 6, 20, 15, 0, tzinfo=timezone(timedelta(hours=3)))
print(aware.strftime("%H:%M %z %Z"))     # => 15:00 +0300 UTC+03:00
```

### `strptime()` — парсинг строки в datetime

```python
from datetime import datetime

# Строка + формат -> datetime
dt = datetime.strptime("2026-06-20 15:30:45", "%Y-%m-%d %H:%M:%S")
print(dt)                    # => 2026-06-20 15:30:45

dt2 = datetime.strptime("20.06.2026", "%d.%m.%Y")
print(dt2)                   # => 2026-06-20 00:00:00

# Парсинг с часовым поясом
dt3 = datetime.strptime("2026-06-20T15:00:00+0300", "%Y-%m-%dT%H:%M:%S%z")
print(dt3)                   # => 2026-06-20 15:00:00+03:00
print(dt3.tzinfo)            # => UTC+03:00

# Несовпадение формата -> ValueError
try:
    datetime.strptime("2026/06/20", "%Y-%m-%d")
except ValueError as e:
    print("Ошибка:", e)      # => Ошибка: time data '2026/06/20' does not match format '%Y-%m-%d'
```

### ISO 8601: `isoformat()` / `fromisoformat()`

```python
from datetime import datetime, timezone, timedelta

dt = datetime(2026, 6, 20, 15, 30, 45, tzinfo=timezone(timedelta(hours=3)))

# isoformat() — стандартный машинный формат
print(dt.isoformat())                # => 2026-06-20T15:30:45+03:00
print(dt.isoformat(sep=' '))         # => 2026-06-20 15:30:45+03:00
print(dt.isoformat(timespec='minutes'))  # => 2026-06-20T15:30+03:00

# fromisoformat() — обратный разбор (Python 3.7+; в 3.11 расширен до полного ISO 8601)
parsed = datetime.fromisoformat("2026-06-20T15:30:45+03:00")
print(parsed == dt)                  # => True

# Python 3.11+: понимает 'Z' и компактные формы
print(datetime.fromisoformat("2026-06-20T12:00:00Z"))  # => 2026-06-20 12:00:00+00:00
```

### `tzinfo`, `timezone.utc`, `astimezone()`

```python
from datetime import datetime, timezone, timedelta

# timezone.utc — готовая UTC-зона
utc_now = datetime(2026, 6, 20, 12, 0, tzinfo=timezone.utc)
print(utc_now)               # => 2026-06-20 12:00:00+00:00

# Фиксированный сдвиг
msk = timezone(timedelta(hours=3), name="MSK")
print(datetime(2026, 6, 20, 15, 0, tzinfo=msk))   # => 2026-06-20 15:00:00+03:00

# astimezone() — КОНВЕРТАЦИЯ в другую зону (момент времени сохраняется!)
in_msk = utc_now.astimezone(msk)
print(in_msk)                # => 2026-06-20 15:00:00+03:00
print(in_msk == utc_now)     # => True (один и тот же инстант)

# astimezone() на наивном объекте -> предполагает локальную зону системы
naive = datetime(2026, 6, 20, 12, 0)
print(naive.astimezone(timezone.utc))   # => конвертирует из локального времени в UTC
```

### `zoneinfo.ZoneInfo` (Python 3.9+) и DST

```python
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo

# ZoneInfo использует базу IANA tz (имена вида 'Europe/Moscow', 'America/New_York')
ny = ZoneInfo("America/New_York")
dt = datetime(2026, 3, 8, 1, 30, tzinfo=ny)
print(dt.utcoffset())        # => -5:00:00 (EST, зимнее время)

# Тот же часовой пояс ПОСЛЕ перехода на летнее время
dt2 = datetime(2026, 3, 8, 3, 30, tzinfo=ny)
print(dt2.utcoffset())       # => -4:00:00 (EDT, летнее время)

# DST-арифметика: прибавление timedelta работает по «настенным часам»,
# смещение пересчитывается через astimezone.
before = datetime(2026, 3, 8, 1, 30, tzinfo=ny)   # до перехода
after = before + timedelta(hours=2)
print(after)                 # => 2026-03-08 03:30:00-05:00 (НАИВНОЕ сложение, сдвиг старый!)
# Чтобы получить корректный момент, нормализуем через UTC:
print((before.astimezone(ZoneInfo("UTC")) + timedelta(hours=2)).astimezone(ny))
# => 2026-03-08 04:30:00-04:00  (учитывает «потерянный» час DST)

# fold — разрешение неоднозначности при «откате» часов (осенью один час повторяется)
ambiguous = datetime(2026, 11, 1, 1, 30, tzinfo=ny)
print(ambiguous.replace(fold=0).utcoffset())   # => -4:00:00 (первое вхождение, EDT)
print(ambiguous.replace(fold=1).utcoffset())   # => -5:00:00 (второе вхождение, EST)
```

### `datetime.min` / `datetime.max` и границы

```python
from datetime import datetime, date, timedelta

print(datetime.min)          # => 0001-01-01 00:00:00
print(datetime.max)          # => 9999-12-31 23:59:59.999999
print(date.min, date.max)    # => 0001-01-01 9999-12-31
print(timedelta.max)         # => 999999999 days, 23:59:59.999999

# Часто используются как «нейтральные» границы для сравнений/сортировки:
items = [datetime(2026, 1, 1), None]
key = lambda x: x or datetime.min
print(sorted(items, key=key))  # None уходит в начало
```

### Wall clock vs monotonic (`time.monotonic`)

```python
import time
from datetime import datetime, timezone

# datetime.now() — «настенные часы»: могут прыгать назад (NTP, перевод часов, DST).
# НЕ подходит для измерения длительности!

# Для измерения интервалов используйте монотонные часы:
start = time.monotonic()
time.sleep(0.05)
elapsed = time.monotonic() - start
print(round(elapsed, 2))     # => 0.05  (гарантированно не идёт назад)

# time.perf_counter() — максимальная точность для бенчмарков
t0 = time.perf_counter()
sum(range(1_000_000))
print(round(time.perf_counter() - t0, 4))  # => например 0.0123
```

---

## Частые вопросы на собеседовании

**Q1. В чём разница между naive и aware datetime?**

*A:* Наивный (naive) объект имеет `tzinfo is None` — он не привязан к часовому поясу и
представляет «локальное время вообще», без однозначной точки на временной оси.
Осведомлённый (aware) объект содержит `tzinfo`, что однозначно определяет UTC-инстант.
Их нельзя смешивать: сравнение naive с aware через `<`/`>` бросает `TypeError`, а
вычитание тоже падает. Для хранения и передачи времени между системами всегда используйте
aware UTC.

```python
from datetime import datetime, timezone
naive = datetime(2026, 6, 20, 12, 0)
aware = datetime(2026, 6, 20, 12, 0, tzinfo=timezone.utc)
try:
    naive < aware
except TypeError as e:
    print(e)   # => can't compare offset-naive and offset-aware datetimes
```

**Q2. Почему `datetime.utcnow()` объявлен устаревшим в Python 3.12 и чем его заменить?**

*A:* `utcnow()` возвращал **наивный** объект, чьи компоненты соответствуют UTC, но без
`tzinfo`. Это провоцировало баги: такой объект выглядит как UTC, но методы вроде
`.timestamp()` и `.astimezone()` трактуют наивное время как **локальное**. Замена:
`datetime.now(timezone.utc)` — он возвращает корректный aware-объект.

```python
from datetime import datetime, timezone
correct = datetime.now(timezone.utc)
print(correct.tzinfo)   # => UTC
```

**Q3. Что вернут `delta.seconds` и `delta.total_seconds()` и в чём разница?**

*A:* `timedelta` хранит данные в трёх нормализованных полях: `days`, `seconds`
(0..86399 — остаток секунд внутри неполного дня) и `microseconds`. Атрибут `.seconds`
даёт только этот остаток, а `.total_seconds()` — полную длительность в секундах как float.

```python
from datetime import timedelta
d = timedelta(days=1, hours=2)
print(d.seconds)          # => 7200   (только 2 часа!)
print(d.total_seconds())  # => 93600.0 (сутки + 2 часа)
```

**Q4. Чем `replace(tzinfo=...)` отличается от `astimezone(...)`?**

*A:* `replace(tzinfo=tz)` просто «приклеивает» зону, не меняя компоненты времени — он
заявляет, что данное настенное время и было в этой зоне. `astimezone(tz)` **конвертирует**
момент в другую зону, пересчитывая компоненты так, чтобы UTC-инстант остался прежним.

```python
from datetime import datetime, timezone, timedelta
dt = datetime(2026, 6, 20, 15, 0, tzinfo=timezone.utc)
print(dt.replace(tzinfo=timezone(timedelta(hours=3))))  # => 15:00:00+03:00 (момент СДВИНУЛСЯ)
print(dt.astimezone(timezone(timedelta(hours=3))))      # => 18:00:00+03:00 (момент тот же)
```

**Q5. Как сравниваются два aware-объекта в разных зонах?**

*A:* Сравнение и равенство aware-объектов идёт по UTC-инстанту, а не по «настенному»
времени. Два объекта с разными компонентами и зонами могут быть равны, если указывают на
один момент.

```python
from datetime import datetime, timezone, timedelta
a = datetime(2026, 6, 20, 15, 0, tzinfo=timezone(timedelta(hours=3)))
b = datetime(2026, 6, 20, 12, 0, tzinfo=timezone.utc)
print(a == b)   # => True
```

**Q6. Чем `strptime` отличается от `fromisoformat`, и что предпочесть?**

*A:* `strptime(s, fmt)` парсит по произвольному формату, но медленнее и требует точного
указания шаблона. `fromisoformat(s)` парсит строго ISO 8601 — он быстрее и не требует
формата. Если вход гарантированно ISO (например, из API/БД), берите `fromisoformat`. В
Python 3.11+ он понимает суффикс `Z` и большинство вариаций ISO.

```python
from datetime import datetime
print(datetime.fromisoformat("2026-06-20T12:00:00Z"))   # => 2026-06-20 12:00:00+00:00
print(datetime.strptime("20/06/2026", "%d/%m/%Y"))      # => 2026-06-20 00:00:00
```

**Q7. Как корректно работать с переходом на летнее время (DST)?**

*A:* Используйте `zoneinfo.ZoneInfo` с именами IANA. Ключевое правило: храните и считайте
интервалы в UTC, а в локальную зону переводите только для отображения. Прибавление
`timedelta` к aware-объекту работает по настенным часам и НЕ перескакивает «потерянный»
час автоматически — для корректной арифметики переходите в UTC, складывайте, затем
возвращайтесь через `astimezone`. Для неоднозначных моментов (осенний откат) используйте
атрибут `fold`.

```python
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo
ny = ZoneInfo("America/New_York")
start = datetime(2026, 3, 8, 1, 30, tzinfo=ny)
# Корректно: через UTC
result = (start.astimezone(ZoneInfo("UTC")) + timedelta(hours=2)).astimezone(ny)
print(result)   # => 2026-03-08 04:30:00-04:00
```

**Q8. Почему нельзя использовать `datetime.now()` для измерения длительности кода?**

*A:* `datetime.now()` отражает «настенные» часы, которые могут скачкообразно меняться
(синхронизация NTP, ручной перевод, DST). Разница может оказаться отрицательной или
неточной. Для измерения интервалов используйте монотонные часы `time.monotonic()` (не
идут назад) или `time.perf_counter()` (максимальная точность для бенчмарков).

```python
import time
start = time.monotonic()
time.sleep(0.01)
print(time.monotonic() - start > 0)   # => True (всегда)
```

**Q9. Что произойдёт при вычитании двух datetime и каков тип результата?**

*A:* Вычитание двух `datetime` (или двух `date`) даёт `timedelta`. Оба операнда должны
быть одного «вида»: оба naive или оба aware, иначе `TypeError`. Для aware-объектов разница
считается по UTC-инстантам, поэтому корректно учитывает разные зоны.

```python
from datetime import datetime, timezone, timedelta
a = datetime(2026, 6, 20, 15, 0, tzinfo=timezone(timedelta(hours=3)))
b = datetime(2026, 6, 20, 12, 0, tzinfo=timezone.utc)
print(a - b)   # => 0:00:00 (один момент)
```

**Q10. Как получить «начало дня» (полночь) и «начало месяца» для даты?**

*A:* Полночь — через `replace`. Начало месяца — `replace(day=1, ...)`. Для последнего дня
месяца удобно перейти к первому числу следующего месяца и вычесть один день (либо
использовать `calendar.monthrange`).

```python
from datetime import datetime, timedelta
import calendar
dt = datetime(2026, 6, 20, 15, 30, 45)
midnight = dt.replace(hour=0, minute=0, second=0, microsecond=0)
print(midnight)   # => 2026-06-20 00:00:00
month_start = dt.replace(day=1, hour=0, minute=0, second=0, microsecond=0)
print(month_start)  # => 2026-06-01 00:00:00
last_day = calendar.monthrange(dt.year, dt.month)[1]
print(dt.replace(day=last_day))  # => 2026-06-30 15:30:45
```

**Q11. Почему объекты datetime неизменяемы и как это влияет на код?**

*A:* `datetime`, `date`, `time`, `timedelta` — immutable. Это делает их хешируемыми
(можно использовать как ключи словарей и элементы множеств) и потокобезопасными. Любая
«мутация» возвращает новый объект; забыть присвоить результат — частая ошибка.

```python
from datetime import datetime
dt = datetime(2026, 6, 20)
dt.replace(year=2030)        # результат ОТБРОШЕН
print(dt.year)               # => 2026
dt = dt.replace(year=2030)   # правильно
print(dt.year)               # => 2030
```

**Q12. Как сериализовать datetime в JSON и обратно?**

*A:* `json` не умеет сериализовать `datetime` напрямую. Стандартный приём — переводить в
ISO-строку через `isoformat()`, а при чтении парсить `fromisoformat()`. Для aware-времени
это сохраняет часовой пояс.

```python
import json
from datetime import datetime, timezone
dt = datetime(2026, 6, 20, 12, 0, tzinfo=timezone.utc)
payload = json.dumps({"ts": dt.isoformat()})
print(payload)               # => {"ts": "2026-06-20T12:00:00+00:00"}
back = datetime.fromisoformat(json.loads(payload)["ts"])
print(back == dt)            # => True
```

---

## Подводные камни (gotchas)

1. **`utcnow()` / `utcfromtimestamp()` возвращают naive.** Они выглядят как UTC, но без
   `tzinfo`. В 3.12 помечены deprecated. Используйте `datetime.now(timezone.utc)` и
   `datetime.fromtimestamp(ts, timezone.utc)`.

2. **Смешивание naive и aware → `TypeError`.** Нельзя сравнивать/вычитать наивный и
   осведомлённый объекты.

3. **`replace(tzinfo=...)` не конвертирует.** Он меняет смысл, а не значение. Для
   конвертации нужен `astimezone`.

4. **`.timestamp()` на naive объекте использует локальную зону системы.** Результат
   зависит от настроек машины и недетерминирован между серверами.

5. **`delta.seconds` ≠ полная длительность.** Это только остаток внутри дня; берите
   `total_seconds()`.

6. **Арифметика timedelta поверх DST некорректна по умолчанию.** `dt + timedelta(hours=2)`
   считает по настенным часам и не учитывает «потерянный/повторённый» час. Считайте в UTC.

7. **`pytz` ≠ `zoneinfo`.** Старый `pytz` требует `localize()` и `normalize()`; его наивное
   присвоение через `replace(tzinfo=pytz_zone)` даёт неверный LMT-сдвиг. В новом коде
   используйте стандартный `zoneinfo`.

8. **Объекты immutable.** Забытое присваивание результата `replace`/арифметики — частая
   ошибка.

9. **`strftime("%Y")` для годов < 1000** может дать нестандартный вывод (без ведущих нулей
   на некоторых платформах). А `%Y` не работает для дат до года 1000 единообразно.

10. **Локаль влияет на `%A`, `%a`, `%B`, `%b`, `%p`.** Имена дней/месяцев зависят от
    `locale`; для машинных форматов используйте ISO, а не текстовые имена.

11. **`datetime.now()` для бенчмарков — ошибка.** Используйте `time.monotonic()` /
    `time.perf_counter()`.

12. **`fromisoformat()` до Python 3.11 не понимал суффикс `Z`** и многие ISO-вариации.
    Учитывайте версию интерпретатора.

---

## Лучшие практики

- **Храните время в UTC, aware.** Внутри системы — `datetime.now(timezone.utc)`; локальную
  зону применяйте только на границе отображения пользователю.
- **Для часовых поясов — `zoneinfo.ZoneInfo`** (стандартная библиотека, 3.9+). На Windows
  при отсутствии системной tz-базы установите пакет `tzdata`.
- **Сериализация — через `isoformat()` / `fromisoformat()`.** Это машинно-надёжный формат.
- **Не используйте `utcnow()` / `utcfromtimestamp()`.** Только `now(timezone.utc)` и
  `fromtimestamp(ts, timezone.utc)`.
- **Для измерения интервалов — монотонные часы** (`time.monotonic`, `time.perf_counter`),
  никогда не `datetime.now()`.
- **DST-арифметику ведите в UTC**, конвертируя в локаль только для вывода.
- **Не смешивайте naive и aware**; на входе системы нормализуйте всё к aware UTC.
- **Помните про immutability** — присваивайте результат `replace`/арифметики.
- **Текстовые форматы (`%A`, `%B`) — только для отображения**, не для парсинга и логики.

---

## Шпаргалка

```python
from datetime import datetime, date, time, timedelta, timezone
from zoneinfo import ZoneInfo
import time as time_mod

# --- Создание ---
datetime(2026, 6, 20, 15, 30, 45)          # naive datetime
datetime.now(timezone.utc)                  # aware UTC — ПРАВИЛЬНОЕ «сейчас»
datetime.fromtimestamp(1780000000, timezone.utc)  # из Unix timestamp -> aware
date.today()                                # текущая дата (локальная)

# --- В timestamp и обратно ---
datetime(2026, 6, 20, tzinfo=timezone.utc).timestamp()   # -> float секунд

# --- timedelta ---
timedelta(days=5, hours=3, minutes=30)      # длительность
(d2 - d1).total_seconds()                   # полная разница в секундах

# --- Изменение / конвертация зон ---
dt.replace(hour=0, minute=0, second=0, microsecond=0)    # полночь (НЕ меняет зону)
dt.replace(tzinfo=timezone.utc)             # ПРИКЛЕИТЬ зону (не конвертирует)
dt.astimezone(ZoneInfo("Europe/Moscow"))    # КОНВЕРТИРОВАТЬ в зону

# --- Форматирование ---
dt.strftime("%Y-%m-%d %H:%M:%S")            # настенное время -> строка
dt.isoformat()                              # ISO 8601 строка

# --- Парсинг ---
datetime.strptime("20.06.2026", "%d.%m.%Y") # по своему формату
datetime.fromisoformat("2026-06-20T12:00:00Z")  # ISO 8601 (3.11+ понимает Z)

# --- Границы ---
datetime.min, datetime.max                  # 0001-01-01 .. 9999-12-31

# --- Измерение длительности (НЕ datetime!) ---
t0 = time_mod.monotonic(); ...; dur = time_mod.monotonic() - t0

# --- Коды strftime: %Y %m %d  %H %M %S %f  %z %Z  %A %a %B %b  %j %p %% ---
```

### Чеклист «правильного» обращения со временем

- [ ] «Сейчас» → `datetime.now(timezone.utc)`, а не `utcnow()`.
- [ ] Хранение/передача → aware UTC в ISO-формате.
- [ ] Часовые пояса → `zoneinfo.ZoneInfo`, не `pytz`.
- [ ] Конвертация зон → `astimezone`, а не `replace(tzinfo=...)`.
- [ ] Длительность кода → `time.monotonic` / `time.perf_counter`.
- [ ] DST-арифметика → считаем в UTC, отображаем в локали.
- [ ] Не смешиваем naive и aware.
