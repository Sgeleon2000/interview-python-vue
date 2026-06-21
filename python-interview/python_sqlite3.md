# sqlite3 — подготовка к собеседованию

## Что это и зачем

`sqlite3` — встроенный в стандартную библиотеку Python модуль для работы с **SQLite**: легковесной, встраиваемой, файловой (или in-memory) реляционной СУБД. Не требует отдельного сервера — вся БД хранится в одном файле. Модуль соответствует **DB-API 2.0** (PEP 249), поэтому интерфейс (`connect`, `cursor`, `execute`, ...) типичен и для других драйверов БД (`psycopg2`, `mysqlclient`).

Когда применять:
- локальное хранилище приложения (десктоп, мобильные, CLI);
- кэш, прототипы, тесты (in-memory `:memory:`);
- встроенная аналитика, файлы данных, конфиги со сложными запросами.

Когда НЕ применять:
- высоконагруженная многопользовательская запись (SQLite блокирует БД на запись);
- сетевой доступ нескольких клиентов одновременно (используйте PostgreSQL/MySQL).

## Ключевые концепции

- **Connection** — соединение с файлом БД (или `:memory:`).
- **Cursor** — объект для выполнения запросов и итерации по результатам.
- **DB-API 2.0** — стандартный интерфейс Python к БД.
- **Параметризованные запросы** — подстановка значений через плейсхолдеры (`?` или `:name`), а не конкатенацию строк. Защита от SQL-инъекций.
- **Транзакция** — атомарная группа операций; `commit` фиксирует, `rollback` откатывает.
- **Autocommit / isolation_level** — режим управления транзакциями.
- **row_factory** — настройка типа возвращаемой строки (tuple, `sqlite3.Row`, dict, namedtuple).
- **WAL** — режим журналирования (Write-Ahead Logging) для лучшей конкурентности.

## Основные функции/классы/методы

### connect, cursor, execute

```python
import sqlite3

# Подключение к файлу (создастся, если нет). Для теста: ":memory:"
conn = sqlite3.connect("app.db")
cur = conn.cursor()

# Создание таблицы (DDL)
cur.execute("""
    CREATE TABLE IF NOT EXISTS users (
        id    INTEGER PRIMARY KEY AUTOINCREMENT,
        name  TEXT NOT NULL,
        email TEXT UNIQUE,
        age   INTEGER
    )
""")
conn.commit()
conn.close()
```

### Параметризованные запросы (защита от SQL-инъекций)

> **Всегда** передавайте значения через плейсхолдеры. **Никогда** не вклеивайте данные в SQL через f-строки/`format`/конкатенацию.

```python
import sqlite3

conn = sqlite3.connect(":memory:")
cur = conn.cursor()
cur.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT, age INTEGER)")

# ПРАВИЛЬНО: позиционные плейсхолдеры '?'
cur.execute("INSERT INTO users (name, age) VALUES (?, ?)", ("Андрей", 30))

# ПРАВИЛЬНО: именованные плейсхолдеры ':name'
cur.execute(
    "INSERT INTO users (name, age) VALUES (:name, :age)",
    {"name": "Мария", "age": 25},
)

# Выборка с параметром
cur.execute("SELECT * FROM users WHERE age > ?", (26,))
print(cur.fetchall())  # [(1, 'Андрей', 30)]

conn.commit()
```

Почему это безопасно: драйвер передаёт значения отдельно от текста запроса, они не интерпретируются как SQL. Демонстрация уязвимости — в разделе gotchas.

### executemany — пакетная вставка

```python
import sqlite3

conn = sqlite3.connect(":memory:")
cur = conn.cursor()
cur.execute("CREATE TABLE nums (x INTEGER)")

rows = [(i,) for i in range(1000)]  # последовательность кортежей-параметров
cur.executemany("INSERT INTO nums (x) VALUES (?)", rows)  # быстро и безопасно
conn.commit()
print(cur.execute("SELECT COUNT(*) FROM nums").fetchone())  # (1000,)
```

`executemany` существенно быстрее цикла из отдельных `execute` (одна транзакция, меньше накладных расходов).

### Методы выборки: fetchone / fetchmany / fetchall

```python
cur.execute("SELECT id, name FROM users")

row = cur.fetchone()          # одна строка или None
batch = cur.fetchmany(100)    # список до N строк
rest = cur.fetchall()         # все оставшиеся строки

# Итерация по курсору (ленивая, экономит память на больших результатах)
for row in cur.execute("SELECT * FROM users"):
    print(row)
```

### Транзакции: commit / rollback

```python
import sqlite3

conn = sqlite3.connect(":memory:")
cur = conn.cursor()
cur.execute("CREATE TABLE accounts (id INTEGER, balance INTEGER)")
cur.executemany("INSERT INTO accounts VALUES (?, ?)", [(1, 100), (2, 50)])
conn.commit()

try:
    # Перевод средств — атомарно
    cur.execute("UPDATE accounts SET balance = balance - 30 WHERE id = 1")
    cur.execute("UPDATE accounts SET balance = balance + 30 WHERE id = 2")
    # raise RuntimeError("сбой")  # если раскомментировать — откат
    conn.commit()                 # фиксируем обе операции
except Exception:
    conn.rollback()               # откатываем всё при ошибке
    raise
```

По умолчанию (`isolation_level` != None в режиме legacy) sqlite3 неявно открывает транзакцию перед DML (`INSERT`/`UPDATE`/`DELETE`) и требует явного `commit()`. DDL и `SELECT` ведут себя по-разному в зависимости от версии Python.

### Context manager соединения

`with conn:` — это менеджер **транзакции**, а не закрытия соединения. На выходе без ошибки → `commit()`, при исключении → `rollback()`. Само соединение **не закрывается**.

```python
import sqlite3

conn = sqlite3.connect(":memory:")
conn.execute("CREATE TABLE t (x INTEGER)")

# Транзакция: commit при успехе, rollback при исключении
with conn:
    conn.execute("INSERT INTO t VALUES (?)", (1,))
    conn.execute("INSERT INTO t VALUES (?)", (2,))
# здесь уже сделан commit, но conn ОТКРЫТО

with conn:
    conn.execute("INSERT INTO t VALUES (?)", (3,))
    raise ValueError("сбой")  # этот INSERT откатится
# -> ValueError пробрасывается, но (3,) НЕ записан

conn.close()  # закрывать нужно отдельно (или через contextlib.closing)
```

Чтобы и транзакция, и закрытие:

```python
import sqlite3
from contextlib import closing

with closing(sqlite3.connect(":memory:")) as conn:  # гарантирует close()
    with conn:                                      # гарантирует commit/rollback
        conn.execute("CREATE TABLE t (x)")
        conn.execute("INSERT INTO t VALUES (1)")
```

### row_factory — формат строк

По умолчанию строки — кортежи. Часто удобнее доступ по имени.

```python
import sqlite3

conn = sqlite3.connect(":memory:")
conn.execute("CREATE TABLE u (id INTEGER, name TEXT)")
conn.execute("INSERT INTO u VALUES (1, 'Андрей')")

# sqlite3.Row: доступ по индексу И по имени колонки, поддержка keys()
conn.row_factory = sqlite3.Row
row = conn.execute("SELECT * FROM u").fetchone()
print(row["name"], row[1])     # Андрей Андрей
print(row.keys())              # ['id', 'name']
print(dict(row))               # {'id': 1, 'name': 'Андрей'}

# Свой factory: словарь
def dict_factory(cursor, row):
    cols = [c[0] for c in cursor.description]
    return dict(zip(cols, row))

conn.row_factory = dict_factory
print(conn.execute("SELECT * FROM u").fetchone())  # {'id': 1, 'name': 'Андрей'}
```

В Python 3.12+ есть встроенный `sqlite3.Row` и улучшенный режим `autocommit`.

### Полезные настройки соединения

```python
import sqlite3

conn = sqlite3.connect("app.db", timeout=5.0)   # ждать снятия блокировки 5с
conn.execute("PRAGMA foreign_keys = ON")        # включить внешние ключи (по умолчанию OFF!)
conn.execute("PRAGMA journal_mode = WAL")       # WAL: параллельное чтение при записи
conn.row_factory = sqlite3.Row
```

## Частые вопросы на собеседовании

**Q1. Как защититься от SQL-инъекций в sqlite3?**
Использовать **параметризованные запросы** с плейсхолдерами `?` (позиционные) или `:name` (именованные), передавая значения вторым аргументом `execute`. Никогда не формировать SQL конкатенацией/f-строками с пользовательскими данными.

**Q2. Что делает `with conn:`?**
Управляет **транзакцией**: `commit()` при успешном выходе, `rollback()` при исключении. Соединение при этом **не закрывается** — это частая ловушка. Для закрытия используйте `contextlib.closing` или `conn.close()`.

**Q3. В чём разница между Connection и Cursor?**
Connection — соединение с БД и владелец транзакции (`commit`/`rollback`/`close`). Cursor — объект выполнения запросов и итерации по результатам. У `Connection` есть шорткаты `execute/executemany/executescript`, которые сами создают временный курсор.

**Q4. Зачем `executemany` вместо цикла?**
Производительность: одна транзакция и один подготовленный запрос вместо множества round-trip. Также чище код. Параметры передаются последовательностью кортежей/словарей.

**Q5. Когда нужен `commit()`?**
После любых изменяющих данные операций (`INSERT`/`UPDATE`/`DELETE`), иначе изменения не сохранятся (откатятся при закрытии). В legacy-режиме sqlite3 открывает транзакцию неявно; `commit()` обязателен.

**Q6. Что такое `row_factory` и `sqlite3.Row`?**
`row_factory` задаёт, как формируется каждая строка результата. `sqlite3.Row` даёт доступ к полям и по индексу, и по имени, плюс `keys()`, и легко конвертируется в dict — удобнее голых кортежей.

**Q7. `parse_float`/типы — как sqlite3 хранит типы?**
SQLite использует динамическую типизацию (type affinity). Python-типы маппятся: `int`→INTEGER, `float`→REAL, `str`→TEXT, `bytes`→BLOB, `None`→NULL. `datetime`/`Decimal` нужно адаптировать (`detect_types`, регистрация адаптеров/конвертеров).

**Q8. Как хранить `datetime`?**
Через адаптеры/конвертеры или вручную как ISO-строку/timestamp. В новых версиях Python встроенные datetime-адаптеры устарели (deprecated) — рекомендуется регистрировать свои или хранить как TEXT/INTEGER.

**Q9. Потокобезопасен ли объект соединения?**
По умолчанию соединение нельзя использовать из другого потока (`check_same_thread=True`). Для многопоточности — отдельное соединение на поток или пул. Сам файл SQLite сериализует запись блокировкой.

**Q10. Что такое WAL-режим?**
`PRAGMA journal_mode=WAL` — журналирование с упреждающей записью: читатели не блокируют писателя и наоборот (один писатель). Улучшает конкурентность по сравнению с дефолтным rollback journal.

**Q11. Что вернёт `fetchone()`, если строк нет?**
`None`. `fetchall()`/`fetchmany()` вернут пустой список `[]`.

**Q12. Чем `execute` отличается от `executescript`?**
`execute` выполняет **один** SQL-стейтмент с параметрами. `executescript` выполняет несколько стейтментов из одной строки (без параметров) и неявно делает `commit` перед выполнением. `executescript` нельзя использовать с пользовательским вводом.

## Подводные камни (gotchas)

### БЕЗОПАСНОСТЬ — SQL-инъекции

> Конкатенация пользовательских данных в SQL — критическая уязвимость.

```python
import sqlite3

conn = sqlite3.connect(":memory:")
conn.execute("CREATE TABLE users (name TEXT)")
conn.execute("INSERT INTO users VALUES ('admin')")

# ОПАСНО! Никогда так не делайте:
user_input = "'; DROP TABLE users; --"
# conn.execute(f"SELECT * FROM users WHERE name = '{user_input}'")
# -> атакующий может выполнить ПРОИЗВОЛЬНЫЙ SQL: удалить таблицу, прочитать чужие данные

# БЕЗОПАСНО — параметризация:
conn.execute("SELECT * FROM users WHERE name = ?", (user_input,))
# значение трактуется как СТРОКА, а не как SQL
```

Дополнительно:
- Плейсхолдеры работают **только для значений**, не для имён таблиц/колонок. Имена таблиц/колонок нельзя параметризовать — их валидируйте по белому списку.
- Не используйте `executescript` с пользовательским вводом — он не параметризуется и выполняет несколько стейтментов.

```python
# Имя колонки нельзя через '?'. Если оно динамическое — белый список:
ALLOWED_COLUMNS = {"name", "email", "age"}
column = "age"
if column not in ALLOWED_COLUMNS:
    raise ValueError("Недопустимая колонка")
conn.execute(f"SELECT {column} FROM users WHERE id = ?", (1,))  # имя проверено, значение параметр
```

### Прочие подводные камни

- **`with conn:` не закрывает соединение** — только commit/rollback. Используйте `closing()`.
- **Забытый `commit()`** — изменения теряются при закрытии.
- **Внешние ключи выключены по умолчанию.** Нужен `PRAGMA foreign_keys = ON` на каждое соединение.
- **Один параметр-кортеж.** `execute("... ?", (x))` — ошибка: `(x)` не кортеж. Нужно `(x,)`.
- **Блокировки записи.** При конкуренции возможна `database is locked`; помогает `timeout=` и WAL.
- **Потоки.** `check_same_thread` по умолчанию запрещает шаринг соединения между потоками.
- **Динамическая типизация SQLite.** Можно записать строку в INTEGER-колонку (type affinity мягкая). Валидируйте на стороне приложения.
- **`:memory:` БД исчезает** при закрытии соединения; для шаринга между соединениями нужен URI с `cache=shared`.
- **`lastrowid`** доступен на курсоре после INSERT; `rowcount` — число затронутых строк (может быть -1 для SELECT).

## Лучшие практики

1. **Всегда параметризуйте** значения (`?`/`:name`); имена таблиц/колонок — через белый список.
2. Оборачивайте изменения в транзакции (`with conn:`); закрывайте соединение через `closing()`.
3. Включайте `PRAGMA foreign_keys = ON` и (для конкурентности) `journal_mode = WAL`.
4. Для пакетов используйте `executemany`/одну транзакцию вместо множества коммитов.
5. Ставьте `conn.row_factory = sqlite3.Row` для читаемого кода.
6. Указывайте `timeout` для устойчивости к блокировкам; не шарьте соединение между потоками.
7. Не используйте `executescript` с недоверенным вводом.
8. Для больших выборок итерируйте курсор / используйте `fetchmany`, а не `fetchall`.
9. Закрывайте курсоры/соединения; используйте контекстные менеджеры.

## Шпаргалка

```python
import sqlite3
from contextlib import closing

# --- Соединение ---
conn = sqlite3.connect("app.db", timeout=5.0)
conn.row_factory = sqlite3.Row
conn.execute("PRAGMA foreign_keys = ON")

# --- Параметризация (защита от инъекций) ---
conn.execute("INSERT INTO t (a, b) VALUES (?, ?)", (1, 2))          # позиционные
conn.execute("INSERT INTO t (a, b) VALUES (:a, :b)", {"a": 1, "b": 2})  # именованные
conn.executemany("INSERT INTO t (a) VALUES (?)", [(1,), (2,), (3,)])    # пакет

# --- Выборка ---
cur = conn.execute("SELECT * FROM t WHERE a > ?", (0,))
cur.fetchone()   # одна строка или None
cur.fetchmany(50)
cur.fetchall()
for row in conn.execute("SELECT * FROM t"):   # ленивая итерация
    ...

# --- Транзакции ---
with conn:                       # commit при успехе / rollback при исключении
    conn.execute("UPDATE ...")
# conn НЕ закрыт!
conn.commit(); conn.rollback()   # явно

# --- Закрытие + транзакция ---
with closing(sqlite3.connect("app.db")) as conn:
    with conn:
        conn.execute("...")

# --- Метаданные ---
cur.lastrowid   # id последней вставки
cur.rowcount    # число затронутых строк
cur.description # описание колонок

# ОПАСНО: f"... WHERE x = '{user}'"   -> SQL-инъекция!
# БЕЗОПАСНО: "... WHERE x = ?", (user,)
```
