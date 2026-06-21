# configparser — подготовка к собеседованию

## Что это и зачем

`configparser` — модуль стандартной библиотеки для чтения и записи конфигурационных файлов в формате **INI**. INI-файл состоит из **секций** (`[section]`), внутри которых лежат пары `ключ = значение`. Формат человекочитаемый, его легко править вручную — поэтому он популярен для конфигов приложений (`setup.cfg`, `pytest.ini`, `tox.ini`, `.gitconfig`, `pip.conf`).

Когда применять:
- простые плоские/секционированные конфиги, редактируемые людьми;
- настройки приложения с группировкой по секциям и значениями по умолчанию;
- когда не нужна вложенность глубже одного уровня.

Когда НЕ применять:
- сложные вложенные структуры/списки/типизированные данные (используйте JSON/TOML/YAML);
- TOML с Python 3.11+ имеет `tomllib` в стандартной библиотеке и часто предпочтительнее для конфигов.

## Ключевые концепции

- **Секция** — именованная группа настроек `[name]`. Есть особая `[DEFAULT]`.
- **Опция (ключ)** — имя параметра. По умолчанию имена приводятся к нижнему регистру.
- **`[DEFAULT]`** — секция со значениями по умолчанию, видимыми во всех остальных секциях.
- **Интерполяция** — подстановка значений других опций внутри значения (`%(name)s` или `${name}`).
- **Типизированный доступ** — `get`, `getint`, `getfloat`, `getboolean`.
- **Всё хранится как строки**; типизация — через методы-геттеры.

## Основные функции/классы/методы

### Чтение INI

Пример файла `config.ini`:

```ini
[DEFAULT]
timeout = 30
retries = 3

[server]
host = example.com
port = 8080
debug = yes

[database]
url = postgres://localhost/mydb
timeout = 60
```

```python
import configparser

config = configparser.ConfigParser()
config.read("config.ini", encoding="utf-8")  # читает один или несколько файлов

# Список секций (БЕЗ DEFAULT)
print(config.sections())            # ['server', 'database']

# Доступ как к вложенному словарю -> всё СТРОКИ
print(config["server"]["host"])     # example.com
print(config["server"]["port"])     # '8080'  (строка!)

# DEFAULT виден во всех секциях:
print(config["server"]["timeout"])  # '30' (из DEFAULT)
print(config["database"]["timeout"])# '60' (переопределено в секции)

# Безопасный доступ с fallback
print(config.get("server", "missing", fallback="def"))  # def
```

### Типизированный доступ

```python
import configparser

config = configparser.ConfigParser()
config.read("config.ini")

port = config.getint("server", "port")        # 8080 (int)
timeout = config.getfloat("database", "timeout")  # 60.0 (float)
debug = config.getboolean("server", "debug")  # True

# getboolean понимает: yes/no, true/false, on/off, 1/0 (регистронезависимо)
```

### Перебор секций и опций

```python
import configparser

config = configparser.ConfigParser()
config.read("config.ini")

for section in config.sections():
    print(f"[{section}]")
    for key, value in config[section].items():  # включает унаследованные из DEFAULT
        print(f"  {key} = {value}")

# Проверки наличия
print("server" in config)                    # True
print(config.has_section("server"))          # True
print(config.has_option("server", "host"))   # True
```

### Запись INI

```python
import configparser

config = configparser.ConfigParser()

# Заполнение программно
config["DEFAULT"] = {"timeout": "30"}
config["server"] = {"host": "localhost", "port": "8000"}
config["server"]["debug"] = "yes"     # значения присваиваются СТРОКАМИ

# Добавление секции отдельно
config.add_section("cache")
config.set("cache", "backend", "redis")

with open("out.ini", "w", encoding="utf-8") as f:
    config.write(f)                   # сериализация обратно в INI
```

### Интерполяция

**BasicInterpolation** (по умолчанию) — синтаксис `%(name)s`:

```python
import configparser

ini = """
[paths]
home = /opt/app
logs = %(home)s/logs
data = %(home)s/data
"""

config = configparser.ConfigParser()  # BasicInterpolation по умолчанию
config.read_string(ini)
print(config["paths"]["logs"])  # /opt/app/logs  — подставилось home
```

**ExtendedInterpolation** — синтаксис `${name}` и ссылки между секциями `${section:option}`:

```python
import configparser

ini = """
[common]
base = /srv

[app]
root = ${common:base}/app
logs = ${root}/logs
"""

config = configparser.ConfigParser(interpolation=configparser.ExtendedInterpolation())
config.read_string(ini)
print(config["app"]["root"])  # /srv/app
print(config["app"]["logs"])  # /srv/app/logs
```

**Отключение интерполяции** (когда в значениях есть `%`, например в форматах логов или паролях):

```python
import configparser

config = configparser.ConfigParser(interpolation=None)  # без интерполяции
config.read_string("[log]\nfmt = %(asctime)s %(message)s\n")
print(config["log"]["fmt"])  # %(asctime)s %(message)s  — не пытается подставить
```

### Полезные параметры ConfigParser

```python
import configparser

config = configparser.ConfigParser(
    delimiters=("=", ":"),         # разделители ключ/значение
    comment_prefixes=("#", ";"),   # префиксы комментариев
    inline_comment_prefixes=(";",),# inline-комментарии
    allow_no_value=True,           # разрешить ключи без значения (флаги)
    default_section="DEFAULT",     # имя секции по умолчанию
)

# Сохранить регистр ключей (по умолчанию ключи -> нижний регистр!)
config.optionxform = str
```

## Частые вопросы на собеседовании

**Q1. Какого типа значения возвращает configparser?**
Всегда строки. Для типизации используют `getint`/`getfloat`/`getboolean`, либо приводят вручную. Это частый источник багов («почему port — строка?»).

**Q2. Что такое секция `[DEFAULT]`?**
Особая секция, чьи опции наследуются всеми остальными секциями. Не попадает в `sections()`, но видна при доступе к опциям любой секции (если не переопределена локально). Используется для общих значений по умолчанию.

**Q3. Как работает `getboolean` и какие значения считаются True/False?**
True: `1`, `yes`, `true`, `on`; False: `0`, `no`, `false`, `off` (регистронезависимо). Любое другое значение → `ValueError`.

**Q4. Что такое интерполяция и какие виды бывают?**
Механизм подстановки значений других опций внутрь значения. `BasicInterpolation` (дефолт) — `%(name)s` в пределах секции/DEFAULT. `ExtendedInterpolation` — `${name}` и кросс-секционные `${section:option}`. Отключается `interpolation=None`.

**Q5. Почему значение с `%` ломает чтение?**
Из-за BasicInterpolation `%` имеет спецсмысл. Одиночный `%` вызовет ошибку интерполяции. Решение: экранировать как `%%` или отключить интерполяцию (`interpolation=None`).

**Q6. Чувствительны ли имена ключей к регистру?**
Нет, по умолчанию ключи приводятся к нижнему регистру (`optionxform = str.lower`). Чтобы сохранить регистр, переопределите `config.optionxform = str`. Имена **секций** регистрозависимы.

**Q7. Как прочитать несколько конфигов с переопределением?**
`config.read(["default.ini", "user.ini"])` — файлы читаются по порядку, последующие переопределяют предыдущие. Отсутствующие файлы молча игнорируются (возвращается список реально прочитанных).

**Q8. configparser vs JSON/TOML/YAML — когда что?**
INI/configparser — простые секционированные конфиги для людей, без вложенности и типов. JSON — обмен данными, есть типы, но не для ручного редактирования (нет комментариев). TOML (`tomllib` с 3.11) — современный стандарт для конфигов с типами и вложенностью. YAML — богатый, но внешняя зависимость и подводные камни.

**Q9. Как обработать дублирующиеся ключи/секции?**
По умолчанию дубль секции/опции в одном файле вызывает `DuplicateSectionError`/`DuplicateOptionError`. Поведение настраивается через `strict=False`.

**Q10. Поддерживает ли configparser списки?**
Нативно — нет. Распространённый приём: хранить как строку с разделителем и парсить вручную (`value.split(",")`), либо переходить на TOML/JSON для структурных данных.

## Подводные камни (gotchas)

- **Всё строки.** Забытая конвертация (`getint`/`getfloat`/`getboolean`) — классический баг.
- **`%` и интерполяция.** Значения с процентами (форматы логов, URL-encoded, пароли) ломают BasicInterpolation. Экранируйте `%%` или `interpolation=None`.
- **Регистр ключей теряется** — ключи в нижнем регистре по умолчанию. Установите `optionxform = str` при необходимости.
- **`[DEFAULT]` протекает всюду.** Опции из DEFAULT видны во всех секциях и в `items()` — иногда неожиданно.
- **`read()` молча игнорирует отсутствующие файлы.** Если файл не найден, ошибки нет — проверяйте возвращённый список путей.
- **Нет вложенности.** INI плоский (секция → ключи); для структур нужен другой формат.
- **`getboolean` строгий** — нестандартные значения (`y`, `enabled`) вызовут `ValueError`.
- **Дубликаты** в strict-режиме (дефолт) бросают исключение.
- **Комментарии не сохраняются** при перезаписи файла через `write()` — теряются при round-trip.
- **Inline-комментарии выключены по умолчанию** — нужно явно задать `inline_comment_prefixes`.

## Лучшие практики

1. Используйте типизированные геттеры (`getint`/`getfloat`/`getboolean`), а не ручное приведение строк.
2. Применяйте `fallback=` для опциональных настроек вместо обработки `NoOptionError`.
3. Кладите общие значения в `[DEFAULT]`, специфичные — переопределяйте в секциях.
4. Для значений с `%` экранируйте `%%` или отключайте интерполяцию.
5. Указывайте `encoding="utf-8"` при чтении/записи.
6. Для многоуровневых конфигов или типизированных данных предпочитайте TOML (`tomllib`)/JSON.
7. Сохраняйте регистр ключей (`optionxform = str`) только если это действительно нужно (например, для имён переменных окружения).
8. Проверяйте, что нужные файлы реально прочитаны (анализируйте возврат `read()`).

## Шпаргалка

```python
import configparser

config = configparser.ConfigParser()
config.read("config.ini", encoding="utf-8")     # read() / read_string() / read_dict()

# --- Чтение ---
config.sections()                       # секции (без DEFAULT)
config["server"]["host"]                # доступ как dict -> СТРОКА
config.get("server", "host", fallback="localhost")
config.getint("server", "port")         # int
config.getfloat("db", "timeout")        # float
config.getboolean("server", "debug")    # bool (yes/no/true/false/on/off/1/0)
config.has_section("server")
config.has_option("server", "host")

# --- Запись ---
config["section"] = {"key": "value"}    # значения — строки
config.add_section("new")
config.set("new", "key", "val")
with open("out.ini", "w", encoding="utf-8") as f:
    config.write(f)

# --- Интерполяция ---
# Basic:    %(name)s           (дефолт, в пределах секции/DEFAULT)
# Extended: ${name} / ${sec:opt}
configparser.ConfigParser(interpolation=configparser.ExtendedInterpolation())
configparser.ConfigParser(interpolation=None)   # отключить (для значений с %)

# --- Настройки ---
config.optionxform = str                # сохранить регистр ключей
ConfigParser(allow_no_value=True)        # ключи-флаги без значения
ConfigParser(delimiters=("=", ":"))      # разделители
ConfigParser(inline_comment_prefixes=(";",))  # inline-комментарии

# Помни:
#   значения всегда str -> используй getint/getfloat/getboolean
#   [DEFAULT] наследуется всеми секциями
#   '%' экранируется как '%%' (или interpolation=None)
#   read() молча игнорирует отсутствующие файлы
```
