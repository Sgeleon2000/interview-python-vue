# csv — подготовка к собеседованию

## Что это и зачем

`csv` — модуль стандартной библиотеки для чтения и записи данных в формате **CSV** (Comma-Separated Values) и его вариациях (TSV, точка с запятой и др.). CSV — простой текстовый табличный формат: строки = записи, поля разделены символом-разделителем. Несмотря на кажущуюся простоту, формат полон нюансов (кавычки, экранирование, переводы строк внутри полей, разные локали), и `csv` корректно их обрабатывает — поэтому ручной `split(",")` почти всегда ошибка.

Когда применять:
- импорт/экспорт табличных данных (Excel, БД, аналитика);
- обмен данными с не-программистами и внешними системами;
- логи, дампы, ETL.

Когда НЕ применять:
- сложные вложенные структуры (используйте JSON);
- большие объёмы с типами/схемой (Parquet, БД).

## Ключевые концепции

- **reader / writer** — построчная работа со списками полей.
- **DictReader / DictWriter** — работа со словарями (поля по именам колонок из заголовка).
- **Диалект (dialect)** — набор параметров формата: разделитель, символ кавычки, экранирование, перевод строки. Встроенные: `excel`, `excel-tab`, `unix`.
- **Разделитель (`delimiter`)** — символ между полями (`,`, `;`, `\t`, ...).
- **Кавычки (`quotechar`, `quoting`)** — как обрамляются поля, содержащие спецсимволы.
- **`newline=''`** — обязательный аргумент при открытии файла, чтобы csv сам управлял переводами строк.

## Основные функции/классы/методы

### reader — чтение строк как списков

```python
import csv

# newline='' ОБЯЗАТЕЛЕН: иначе ломаются поля с переводами строк
with open("data.csv", newline="", encoding="utf-8") as f:
    reader = csv.reader(f)            # delimiter=',' по умолчанию
    header = next(reader)            # первая строка — заголовок
    for row in reader:
        # row — список строк (все поля как str!)
        print(row)  # ['1', 'Андрей', '30']
```

### writer — запись строк

```python
import csv

rows = [
    ["id", "name", "city"],
    [1, "Андрей", "Москва"],
    [2, "Мария", "Санкт-Петербург, центр"],  # запятая внутри поля
]

with open("out.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.writer(f)
    writer.writerow(["заголовок"])    # одна строка
    writer.writerows(rows)            # много строк сразу
# Поле с запятой автоматически возьмётся в кавычки: "Санкт-Петербург, центр"
```

### DictReader — чтение в словари

```python
import csv

with open("data.csv", newline="", encoding="utf-8") as f:
    reader = csv.DictReader(f)        # первая строка -> ключи
    print(reader.fieldnames)         # ['id', 'name', 'age']
    for row in reader:
        # row — dict: {'id': '1', 'name': 'Андрей', 'age': '30'}
        print(row["name"], row["age"])

# Можно задать имена вручную, если заголовка нет:
with open("noheader.csv", newline="", encoding="utf-8") as f:
    reader = csv.DictReader(f, fieldnames=["id", "name", "age"])
    for row in reader:
        print(row)
```

### DictWriter — запись из словарей

```python
import csv

records = [
    {"id": 1, "name": "Андрей", "city": "Москва"},
    {"id": 2, "name": "Мария", "city": "Питер"},
]

with open("out.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["id", "name", "city"])
    writer.writeheader()             # записать строку заголовка
    writer.writerows(records)

# extrasaction='ignore' — игнорировать лишние ключи; 'raise' (дефолт) — ошибка
# restval='-' — чем заполнять отсутствующие поля при записи
writer = csv.DictWriter(f, fieldnames=["id", "name"], extrasaction="ignore", restval="-")
```

### Диалекты, разделители, кавычки

```python
import csv

# --- Свой разделитель (например, ';' в русском Excel) ---
with open("ru.csv", newline="", encoding="utf-8") as f:
    reader = csv.reader(f, delimiter=";")
    for row in reader:
        print(row)

# --- TSV (табуляция) ---
with open("data.tsv", newline="", encoding="utf-8") as f:
    reader = csv.reader(f, delimiter="\t")  # или dialect='excel-tab'

# --- Управление кавычками при записи ---
with open("out.csv", "w", newline="", encoding="utf-8") as f:
    # QUOTE_ALL — брать в кавычки ВСЕ поля
    writer = csv.writer(f, quoting=csv.QUOTE_ALL)
    writer.writerow(["a", "b", "c"])  # "a","b","c"

# Режимы quoting:
#   csv.QUOTE_MINIMAL    — кавычки только когда нужно (дефолт)
#   csv.QUOTE_ALL        — всегда
#   csv.QUOTE_NONNUMERIC — все нечисловые поля в кавычках; при ЧТЕНИИ конвертирует числа в float
#   csv.QUOTE_NONE       — не использовать кавычки (нужен escapechar)
```

### Регистрация собственного диалекта

```python
import csv

csv.register_dialect(
    "ru_excel",
    delimiter=";",
    quotechar='"',
    quoting=csv.QUOTE_MINIMAL,
    lineterminator="\r\n",
)

with open("out.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.writer(f, dialect="ru_excel")
    writer.writerow(["a;b", "c"])  # 'a;b' возьмётся в кавычки: "a;b";c
```

### Sniffer — автоопределение диалекта

```python
import csv

with open("unknown.csv", newline="", encoding="utf-8") as f:
    sample = f.read(2048)
    f.seek(0)
    dialect = csv.Sniffer().sniff(sample)        # угадать разделитель/кавычки
    has_header = csv.Sniffer().has_header(sample) # есть ли заголовок
    reader = csv.reader(f, dialect=dialect)
    rows = list(reader)
```

### QUOTE_NONNUMERIC и типизация при чтении

```python
import csv
import io

data = '1,2.5,"text"\n'
reader = csv.reader(io.StringIO(data), quoting=csv.QUOTE_NONNUMERIC)
print(next(reader))  # [1.0, 2.5, 'text']  — НЕкавыченные поля стали float!
# Осторожно: всё без кавычек пытается стать float, иначе ValueError
```

## Частые вопросы на собеседовании

**Q1. Почему нельзя парсить CSV через `line.split(",")`?**
Потому что поля могут содержать сам разделитель внутри кавычек (`"Москва, центр"`), экранированные кавычки (`""`), переводы строк внутри поля. `split` всё это сломает. Модуль `csv` корректно обрабатывает кавычки и экранирование.

**Q2. Зачем `newline=''` при открытии файла?**
Чтобы модуль csv сам управлял переводами строк (`\r\n`). Без него возможны лишние пустые строки на Windows и поломка многострочных полей. Это обязательный паттерн из документации.

**Q3. В чём разница reader/writer и DictReader/DictWriter?**
`reader`/`writer` работают со списками полей (по позициям). `DictReader`/`DictWriter` — со словарями по именам колонок (берут/пишут заголовок). Словарный вариант устойчивее к перестановке колонок и читабельнее.

**Q4. Какого типа значения возвращает csv при чтении?**
Все поля — строки (`str`), типизации нет. Числа/даты нужно приводить вручную. Исключение: `QUOTE_NONNUMERIC` приводит некавыченные поля к `float`.

**Q5. Что такое диалект?**
Именованный набор параметров формата: `delimiter`, `quotechar`, `quoting`, `escapechar`, `lineterminator`, `doublequote`, `skipinitialspace`. Встроенные: `excel` (дефолт), `excel-tab`, `unix`. Можно регистрировать свои.

**Q6. Как обрабатываются кавычки внутри поля?**
По умолчанию (`doublequote=True`) кавычка внутри кавыченного поля удваивается: `He said ""hi""`. Альтернатива — `escapechar` при `doublequote=False`.

**Q7. Как читать русский Excel-CSV с `;`?**
Указать `delimiter=";"` (или зарегистрировать диалект). Русская локаль Excel часто использует `;`, т.к. запятая — десятичный разделитель.

**Q8. Как обрабатывать большие файлы?**
Читать построчно через итератор `reader`/`DictReader` (ленивый, не грузит всё в память), а не `list(reader)`. Писать `writerow` в цикле или `writerows` батчами.

**Q9. Что делают `extrasaction` и `restval` в DictWriter?**
`extrasaction='raise'` (дефолт) — ошибка при лишних ключах сверх `fieldnames`; `'ignore'` — игнорировать. `restval` — значение для отсутствующих в словаре полей.

**Q10. Как угадать формат неизвестного CSV?**
`csv.Sniffer().sniff(sample)` определяет разделитель/кавычки, `has_header(sample)` — наличие заголовка. Не стопроцентно надёжно — для критичных данных задавайте диалект явно.

**Q11. Какую кодировку использовать?**
Указывайте `encoding="utf-8"` (часто `utf-8-sig` для файлов из Excel с BOM). CSV сам по себе кодировку не хранит.

## Подводные камни (gotchas)

- **Забытый `newline=''`** → лишние пустые строки на Windows и поломка многострочных полей. Всегда указывайте.
- **Все значения — строки.** Нет автоматической типизации; `"30"` это строка, не int. Приводите явно.
- **`split(",")` вместо csv** — ломается на кавычках/запятых внутри полей. Не делайте так.
- **Кодировка и BOM.** Excel часто пишет UTF-8 с BOM; читайте `encoding="utf-8-sig"`, иначе первый ключ будет `'﻿id'`.
- **Локаль/разделитель.** В разных странах Excel использует `;` вместо `,`. Не хардкодьте `,`.
- **`QUOTE_NONNUMERIC` при чтении** превращает некавыченные поля в `float` — может упасть с `ValueError` на нечисловых данных.
- **Лимит размера поля.** Очень большие поля вызывают `_csv.Error: field larger than field limit`; решается `csv.field_size_limit(большое_число)`.
- **DictReader и дубликаты колонок.** Одинаковые имена колонок схлопываются — последнее значение перезатирает.
- **Пустые строки.** Файл может содержать пустые строки между записями; учитывайте при обработке.
- **lineterminator** влияет только на запись (writer), при чтении csv распознаёт `\r\n` и `\n` автоматически.

## Лучшие практики

1. Всегда открывайте файлы с `newline=""` и явным `encoding` (`utf-8`/`utf-8-sig`).
2. Предпочитайте `DictReader`/`DictWriter` — код устойчивее к порядку колонок и читабельнее.
3. Не парсите CSV руками (`split`) — используйте модуль.
4. Для больших файлов итерируйте построчно, не грузите всё в память.
5. Приводите типы явно после чтения (int/float/datetime).
6. Регистрируйте диалект, если формат нестандартный (разделитель `;`, особые кавычки).
7. Для Excel-совместимости используйте `dialect='excel'` и `utf-8-sig`.
8. Валидируйте заголовок (`fieldnames`) перед обработкой данных.

## Шпаргалка

```python
import csv

# --- Чтение (списки) ---
with open("in.csv", newline="", encoding="utf-8") as f:
    for row in csv.reader(f, delimiter=","):
        ...  # row — список строк

# --- Чтение (словари) ---
with open("in.csv", newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):
        row["name"]  # доступ по имени колонки

# --- Запись (списки) ---
with open("out.csv", "w", newline="", encoding="utf-8") as f:
    w = csv.writer(f)
    w.writerow(["id", "name"])
    w.writerows([[1, "a"], [2, "b"]])

# --- Запись (словари) ---
with open("out.csv", "w", newline="", encoding="utf-8") as f:
    w = csv.DictWriter(f, fieldnames=["id", "name"], extrasaction="ignore", restval="")
    w.writeheader()
    w.writerows([{"id": 1, "name": "a"}])

# --- Параметры ---
delimiter=";"            # разделитель (',' дефолт, '\t' для TSV)
quotechar='"'            # символ кавычки
quoting=csv.QUOTE_MINIMAL  # / QUOTE_ALL / QUOTE_NONNUMERIC / QUOTE_NONE
escapechar="\\"          # при QUOTE_NONE / doublequote=False

# --- Диалекты ---
csv.register_dialect("ru", delimiter=";")
csv.Sniffer().sniff(sample)       # автоопределение
csv.field_size_limit(10**7)       # увеличить лимит размера поля

# Помни:
#   newline='' ОБЯЗАТЕЛЕН
#   все поля — str (нет типизации)
#   utf-8-sig для файлов из Excel (BOM)
```
