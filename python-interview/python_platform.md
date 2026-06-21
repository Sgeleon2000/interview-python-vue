# platform — подготовка к собеседованию

## Что это и зачем

`platform` — это модуль стандартной библиотеки Python, предназначенный для получения информации о **платформе, на которой выполняется код**: операционной системе, её версии, архитектуре процессора, имени машины, а также о самой реализации интерпретатора Python (версия, компилятор, билд).

Зачем он нужен на практике:

- **Кросс-платформенный код.** Разное поведение на Linux / macOS / Windows (пути, кодировки, системные вызовы, доступные библиотеки).
- **Диагностика и логирование.** Сбор информации об окружении в баг-репортах, телеметрии, при логировании запуска сервиса.
- **Условная установка/сборка зависимостей.** Например, выбор бинарного wheel под `x86_64` или `arm64`, под нужную версию libc.
- **Принятие решений в рантайме.** Включить/выключить функциональность в зависимости от ОС или версии Python.

Ключевая идея: `platform` даёт **более человекочитаемую и детальную** информацию, чем низкоуровневые `sys.platform` и `os.name`. Но за это приходится платить: многие функции **зависят от платформы**, могут вызывать внешние утилиты (`uname`, `ver`, `sw_vers`), кэшировать результат и **возвращать пустую строку**, если данные недоступны.

```python
import platform

# Быстрый снимок окружения
print(platform.platform())          # 'macOS-14.5-arm64-arm-64bit'
print(platform.python_version())    # '3.12.3'
print(platform.machine())           # 'arm64'
```

---

## Ключевые концепции

### 1. Три уровня информации об ОС

В Python есть три «источника правды» о том, где мы выполняемся, и их важно не путать:

| Источник | Что возвращает | Пример значения | Уровень |
|----------|----------------|-----------------|---------|
| `os.name` | Имя семейства API ОС | `'posix'`, `'nt'`, `'java'` | Самый грубый |
| `sys.platform` | Идентификатор платформы (фиксируется при сборке Python) | `'linux'`, `'darwin'`, `'win32'` | Средний |
| `platform.system()` | Человекочитаемое имя ОС (часто из `uname`) | `'Linux'`, `'Darwin'`, `'Windows'` | Детальный |

```python
import os, sys, platform

print(os.name)             # 'posix' (Linux/macOS) или 'nt' (Windows)
print(sys.platform)        # 'linux' / 'darwin' / 'win32'
print(platform.system())   # 'Linux' / 'Darwin' / 'Windows'
```

**Важно:** `sys.platform` для Windows — это `'win32'` даже на 64-битной системе (исторически). А `platform.system()` для macOS — это `'Darwin'`, а не `'macOS'`.

### 2. Зависимость от платформы и «пустые строки»

Многие функции `platform` документированы как «может вернуть пустую строку, если значение не определено». Это не баг, а контракт. Например, `platform.processor()` на Linux часто пуст, потому что соответствующая информация недоступна без парсинга `/proc/cpuinfo`.

Поэтому **никогда не полагайтесь слепо** на непустой результат — всегда обрабатывайте случай пустой строки.

### 3. Кэширование

Часть функций (`uname()`, и зависящие от неё) кэшируют результат: первый вызов может запустить внешнюю утилиту, последующие возвращают закэшированное значение. Это влияет на производительность (первый вызов дороже) и на тестируемость (нужно патчить).

### 4. `uname()` как «корень»

`platform.uname()` — центральная функция: возвращает `namedtuple` с шестью полями, и многие отдельные функции (`system()`, `release()`, `version()`, `machine()`, `node()`, `processor()`) — это, по сути, доступ к полям результата `uname()`.

### 5. Информация о Python vs информация об ОС

Модуль делится на два смысловых блока:

- **Про ОС/железо:** `system`, `release`, `version`, `machine`, `processor`, `node`, `platform`, `architecture`, `uname`, `mac_ver`, `win32_ver`, `libc_ver`, `freedesktop_os_release`.
- **Про интерпретатор Python:** `python_version`, `python_version_tuple`, `python_implementation`, `python_compiler`, `python_build`.

---

## Основные функции/классы/методы

### Информация об операционной системе

#### `system()` — имя ОС

```python
import platform

system = platform.system()
# 'Linux'   — Linux
# 'Darwin'  — macOS (внимание: НЕ 'macOS'!)
# 'Windows' — Windows
# ''        — если не удалось определить

if system == 'Windows':
    sep = '\\'
elif system in ('Linux', 'Darwin'):
    sep = '/'
```

#### `release()` и `version()` — версия ОС

`release()` возвращает «релиз» (короче), `version()` — более подробную «версию». Смысл сильно различается между ОС.

```python
import platform

print(platform.release())
# Linux:   '6.5.0-35-generic'  (версия ядра)
# macOS:   '23.5.0'            (версия ядра Darwin, НЕ '14.5'!)
# Windows: '10' или '11'

print(platform.version())
# Linux:   '#35-Ubuntu SMP PREEMPT_DYNAMIC ...'
# macOS:   'Darwin Kernel Version 23.5.0: ...'
# Windows: '10.0.22631'  (build number)
```

**Подвох:** на macOS `release()` отдаёт версию ядра Darwin (например, `23.5.0`), а не «маркетинговую» версию macOS (`14.5`). Чтобы получить именно версию macOS — используйте `mac_ver()`.

#### `machine()` — архитектура процессора

```python
import platform

arch = platform.machine()
# 'x86_64'  — 64-битный Intel/AMD (Linux, macOS Intel)
# 'AMD64'   — то же на Windows (другое написание!)
# 'arm64'   — Apple Silicon (macOS), некоторые ARM-системы
# 'aarch64' — 64-битный ARM на Linux
# 'i386'/'i686' — 32-битный x86
```

**Подвох:** одна и та же 64-битная Intel-архитектура называется `'x86_64'` на Linux/macOS и `'AMD64'` на Windows. ARM64 — это `'arm64'` на macOS и `'aarch64'` на Linux. Для надёжного определения нормализуйте значения.

```python
import platform

def normalize_arch() -> str:
    """Привести имя архитектуры к единому виду."""
    m = platform.machine().lower()
    if m in ('x86_64', 'amd64'):
        return 'x64'
    if m in ('arm64', 'aarch64'):
        return 'arm64'
    if m in ('i386', 'i686', 'x86'):
        return 'x86'
    return m or 'unknown'
```

#### `processor()` — модель/тип процессора

```python
import platform

print(platform.processor())
# Linux:   часто ''  (пусто!) или 'x86_64'
# macOS:   'arm' или 'i386'
# Windows: 'Intel64 Family 6 Model 142 Stepping 10, GenuineIntel'
```

**Важно:** `processor()` — самая «капризная» функция. На Linux она почти всегда возвращает **пустую строку** (или дублирует `machine()`), потому что точные данные требуют чтения `/proc/cpuinfo`. Не используйте её как надёжный источник.

#### `node()` — имя машины (hostname)

```python
import platform
import socket

print(platform.node())
# Имя хоста, например 'andreis-macbook.local' или ''

# Эквивалентно (но платформенно-зависимо):
print(socket.gethostname())  # часто надёжнее для hostname
```

`node()` может вернуть пустую строку. Для гарантированного получения hostname обычно используют `socket.gethostname()`.

#### `platform()` — единая строка-описание

Самая «сводная» функция: собирает данные воедино в одну строку, удобную для логов.

```python
import platform

print(platform.platform())
# 'macOS-14.5-arm64-arm-64bit'
# 'Linux-6.5.0-35-generic-x86_64-with-glibc2.35'
# 'Windows-10-10.0.22631-SP0'

# aliased=True — использовать «псевдонимы» (например, 'SunOS' -> 'Solaris')
print(platform.platform(aliased=True))

# terse=True — минимальная строка (меньше деталей)
print(platform.platform(terse=True))
# 'macOS-14.5'
```

Параметры:
- `aliased=True` — заменять имена ОС/версий на распространённые псевдонимы (через таблицу алиасов модуля).
- `terse=True` — выдавать только минимально необходимую информацию.

### Информация о Python

#### `python_version()` и `python_version_tuple()`

```python
import platform

print(platform.python_version())        # '3.12.3' (строка)
print(platform.python_version_tuple())  # ('3', '12', '3') — кортеж СТРОК!

# Внимание: элементы — строки, для сравнения приводите к int
major, minor, patch = platform.python_version_tuple()
if (int(major), int(minor)) >= (3, 10):
    print('match-case доступен')
```

**Подвох:** `python_version_tuple()` возвращает кортеж **строк**, а не чисел. Для сравнения версий лучше использовать `sys.version_info` (кортеж int с именованными полями).

```python
import sys
# Предпочтительный способ сравнения версий Python:
if sys.version_info >= (3, 11):
    ...
```

#### `python_implementation()` — реализация интерпретатора

```python
import platform

impl = platform.python_implementation()
# 'CPython'    — эталонная реализация
# 'PyPy'       — JIT-реализация
# 'Jython'     — на JVM (устаревшая)
# 'IronPython' — на .NET

if impl == 'PyPy':
    print('Учитываем особенности JIT и GC PyPy')
```

#### `python_compiler()` — компилятор сборки

```python
import platform

print(platform.python_compiler())
# 'GCC 11.4.0'
# 'Clang 15.0.0 (clang-1500.0.40.1)'
# 'MSC v.1929 64 bit (AMD64)'  — Windows
```

#### `python_build()` — номер и дата билда

```python
import platform

print(platform.python_build())
# ('main', 'Apr  9 2024 08:09:14')  — (build_no, build_date), кортеж строк
```

### `architecture()` — битность исполняемого файла

```python
import platform

print(platform.architecture())
# ('64bit', '')        — Linux/macOS, ELF/Mach-O
# ('64bit', 'WindowsPE')

bits, linkage = platform.architecture()
is_64 = bits == '64bit'

# По умолчанию анализирует sys.executable; можно указать другой файл:
print(platform.architecture(sys.executable))
```

**Важно:** `architecture()` определяет битность **интерпретатора Python** (через анализ исполняемого файла), а не «битность ОС». На 64-битной ОС может стоять 32-битный Python — тогда вернётся `'32bit'`.

Для определения битности Python надёжнее и быстрее:

```python
import sys
is_64bit = sys.maxsize > 2**32   # True для 64-битного Python
```

### `uname()` — всё об ОС одним вызовом

```python
import platform

u = platform.uname()
print(u)
# uname_result(system='Darwin', node='mac.local', release='23.5.0',
#              version='Darwin Kernel Version 23.5.0: ...',
#              machine='arm64')

# Доступ по полям (namedtuple):
print(u.system)    # 'Darwin'
print(u.node)      # 'mac.local'
print(u.release)   # '23.5.0'
print(u.version)   # 'Darwin Kernel Version ...'
print(u.machine)   # 'arm64'
print(u.processor) # 'arm'  — шестое поле, ленивое (вычисляется при доступе)
```

`uname()` возвращает `uname_result` — `namedtuple`-подобный объект с полями:
`system`, `node`, `release`, `version`, `machine`, `processor`.

Особенность: поле `processor` вычисляется **лениво** (при первом обращении), потому что может быть дорогим. Результат `uname()` кэшируется на уровне модуля.

### Платформенно-специфичные функции

Эти функции имеют смысл **только на своей ОС**, на других возвращают пустые/нулевые значения.

#### `mac_ver()` — версия macOS

```python
import platform

print(platform.mac_ver())
# ('14.5', ('', '', ''), 'arm64')
# (release, (version, dev_stage, non_release_version), machine)

# Получить именно маркетинговую версию macOS (а не Darwin!):
mac_release = platform.mac_ver()[0]   # '14.5'

if platform.system() == 'Darwin':
    major = int(platform.mac_ver()[0].split('.')[0])
    if major >= 11:
        print('Big Sur или новее')
```

На не-macOS вернёт `('', ('', '', ''), '')`.

#### `win32_ver()` и `win32_edition()` — версия Windows

```python
import platform

print(platform.win32_ver())
# ('10', '10.0.22631', 'SP0', 'Multiprocessor Free')
# (release, version, csd, ptype)

# Редакция Windows (Python 3.8+):
print(platform.win32_edition())  # 'Professional', 'Core', ...
print(platform.win32_is_iot())   # True на Windows IoT
```

#### `libc_ver()` — версия C-библиотеки (Linux)

```python
import platform

print(platform.libc_ver())
# ('glibc', '2.35')   — типично для Linux
# ('', '')            — если определить не удалось

# Полезно для совместимости manylinux-wheels:
lib, ver = platform.libc_ver()
```

`libc_ver()` анализирует исполняемый файл (по умолчанию `sys.executable`) в поисках строк версии libc. Не определяет musl надёжно — для musl/glibc-различения часто используют сторонние утилиты.

#### `freedesktop_os_release()` — данные дистрибутива Linux (Python 3.10+)

```python
import platform

# Только Linux/BSD, читает /etc/os-release или /usr/lib/os-release
try:
    info = platform.freedesktop_os_release()
    print(info['NAME'])         # 'Ubuntu'
    print(info['VERSION_ID'])   # '22.04'
    print(info['ID'])           # 'ubuntu'
    print(info.get('PRETTY_NAME'))  # 'Ubuntu 22.04.4 LTS'
except OSError:
    # Файл os-release отсутствует (например, на macOS/Windows)
    info = {}
```

Возвращает `dict` с полями из стандарта freedesktop `os-release`. **Заменяет** устаревшую и удалённую `platform.linux_distribution()` (была убрана в Python 3.8).

---

## Частые вопросы на собеседовании

**Вопрос 1. В чём разница между `platform.system()`, `sys.platform` и `os.name`?**

Ответ. Это три уровня детализации.
- `os.name` — грубое имя семейства API ОС: `'posix'` (Linux, macOS, BSD), `'nt'` (Windows). Хорош, когда нужно лишь POSIX vs Windows.
- `sys.platform` — строка, **зафиксированная при компиляции Python**: `'linux'`, `'darwin'`, `'win32'`. Удобна и быстра (не вызывает внешних утилит), классика для условий совместимости. Подвох: `'win32'` даже на 64-битной Windows.
- `platform.system()` — человекочитаемое имя, часто из `uname`: `'Linux'`, `'Darwin'`, `'Windows'`. Может быть дороже (внешний вызов) и вернуть `''`.

Для простых проверок ОС предпочтительнее `sys.platform`; `platform` — когда нужны детали (версия, архитектура).

**Вопрос 2. Почему `platform.system()` на macOS возвращает `'Darwin'`, а не `'macOS'`?**

Ответ. `system()` отдаёт имя ядра ОС из `uname`, а ядро macOS называется Darwin. Маркетинговое название и версию macOS даёт отдельная функция `mac_ver()` (например, `mac_ver()[0]` == `'14.5'`). Аналогично `release()` на macOS вернёт версию ядра Darwin (`23.5.0`), а не `14.5`.

**Вопрос 3. Чем `platform.machine()` отличается между ОС и как надёжно определить архитектуру?**

Ответ. Имена не унифицированы: 64-битный Intel — это `'x86_64'` на Linux/macOS, но `'AMD64'` на Windows. ARM64 — `'arm64'` на macOS и `'aarch64'` на Linux. Надёжное решение — нормализовать значения через словарь/функцию (привести к нижнему регистру и сопоставить с каноническими именами). Также помните, что `machine()` может вернуть пустую строку.

**Вопрос 4. `platform.architecture()` определяет битность ОС или Python?**

Ответ. Битность **интерпретатора Python** — она анализирует исполняемый файл `sys.executable`. На 64-битной ОС с 32-битным Python вернётся `('32bit', ...)`. Если нужна именно битность процесса Python, быстрее и без запуска подпроцессов: `sys.maxsize > 2**32`.

**Вопрос 5. Что возвращает `uname()` и чем он удобен?**

Ответ. `uname_result` — `namedtuple`-подобный объект с полями `system`, `node`, `release`, `version`, `machine`, `processor`. Это «корневая» функция: `system()`, `release()`, `machine()` и др. по сути читают её поля. Удобен тем, что собирает всё одним вызовом, поддерживает доступ по имени поля и кэшируется. Особенность — поле `processor` ленивое (вычисляется при обращении, т.к. может быть дорогим).

**Вопрос 6. Почему `platform.processor()` иногда возвращает пустую строку?**

Ответ. Потому что точные данные о процессоре платформенно-зависимы и не всегда доступны без дополнительного парсинга. На Linux функция часто отдаёт `''` (или дублирует `machine()`), так как детали лежат в `/proc/cpuinfo`, который модуль не читает. Это документированное поведение — нельзя полагаться на непустой результат. Для реальных данных о CPU используют сторонние библиотеки (например, `py-cpuinfo`) или чтение `/proc/cpuinfo` напрямую.

**Вопрос 7. Как корректно сравнивать версии Python — через `platform` или `sys`?**

Ответ. `platform.python_version_tuple()` возвращает кортеж **строк** (`('3', '12', '3')`), поэтому прямое сравнение лексикографически ошибочно (например, `'9' > '10'`). Для сравнения используйте `sys.version_info` — это `namedtuple` с полями-int (`major`, `minor`, `micro`, ...), и сравнение `sys.version_info >= (3, 11)` работает корректно. `platform.python_version()` хорош для вывода в лог как строка.

**Вопрос 8. Как определить дистрибутив Linux в современном Python?**

Ответ. Через `platform.freedesktop_os_release()` (Python 3.10+) — она читает `/etc/os-release` и возвращает `dict` с ключами `NAME`, `VERSION_ID`, `ID`, `PRETTY_NAME` и т.д. Старая `platform.linux_distribution()` была удалена в Python 3.8. На не-Linux функция бросает `OSError`, что нужно обрабатывать. Для широкой совместимости иногда применяют пакет `distro`.

**Вопрос 9. Что такое параметры `aliased` и `terse` у `platform.platform()`?**

Ответ. `platform()` собирает сводную строку об ОС. `aliased=True` подставляет общеупотребительные псевдонимы вместо «сырых» имён (например, `'SunOS'` → `'Solaris'`). `terse=True` даёт сокращённый вариант с минимумом деталей (например, `'macOS-14.5'` вместо полной строки с архитектурой и линковкой). Полезно для компактных логов.

**Вопрос 10. Какие подвохи у функций `platform` с точки зрения надёжности?**

Ответ. Главные:
1. Многие функции **могут вернуть пустую строку** (`processor`, `node`, `libc_ver` и др.).
2. Часть функций **запускает внешние процессы** (`uname`, `ver`) — это медленно и может падать в урезанных окружениях/контейнерах.
3. Результаты **кэшируются** — затрудняет тестирование (нужно патчить).
4. Имена **не унифицированы** между ОС (`AMD64` vs `x86_64`).
5. Платформенно-специфичные функции на «чужой» ОС дают пустышки.
Поэтому пишут защитный код: проверки на пусто, обёртки, нормализацию, обработку исключений.

**Вопрос 11. Как собрать диагностический снимок окружения для баг-репорта?**

Ответ. Комбинируют функции в один словарь/строку: `platform.platform()`, `platform.python_version()`, `platform.python_implementation()`, `platform.machine()`, плюс `sys.version`. Важно оборачивать каждый вызов так, чтобы пустые значения и исключения (например, от `freedesktop_os_release()`) не ломали сбор. Пример — в разделе «Шпаргалка».

**Вопрос 12. Что быстрее для простой проверки «это Windows?» — `platform` или `sys`?**

Ответ. `sys.platform == 'win32'` или `os.name == 'nt'` — быстрее, потому что значение зафиксировано при сборке интерпретатора и не требует запуска подпроцессов. `platform.system() == 'Windows'` тоже корректно, но потенциально дороже (первый вызов может дёрнуть `uname`/системные API). Для горячего пути выбирают `sys`/`os`.

---

## Подводные камни (gotchas)

### 1. Пустые строки — это норма, не ошибка

```python
import platform

proc = platform.processor()
if not proc:                       # на Linux часто пусто!
    proc = platform.machine() or 'unknown'
```

Любая функция, отдающая строку, может вернуть `''`. Всегда предусматривайте fallback.

### 2. macOS: Darwin-версия ≠ версия macOS

```python
import platform

# НЕПРАВИЛЬНО — это версия ядра Darwin:
release = platform.release()       # '23.5.0'

# ПРАВИЛЬНО — маркетинговая версия macOS:
if platform.system() == 'Darwin':
    macos = platform.mac_ver()[0]  # '14.5'
```

### 3. `machine()`: `AMD64` против `x86_64`

```python
import platform

# НЕПРАВИЛЬНО — сломается на Windows:
if platform.machine() == 'x86_64':
    ...

# ПРАВИЛЬНО — учитываем все написания:
if platform.machine().lower() in ('x86_64', 'amd64'):
    ...
```

### 4. `python_version_tuple()` — кортеж строк, а не int

```python
import platform

# НЕПРАВИЛЬНО — лексикографическое сравнение строк:
ver = platform.python_version_tuple()
if ver >= ('3', '9'):              # коварно: '3.10' даст ('3', '10'), и '10' < '9' как строки!
    ...

# ПРАВИЛЬНО — сравнивайте int-версию:
import sys
if sys.version_info >= (3, 9):
    ...
```

### 5. `architecture()` ≠ битность ОС, и она дорогая

```python
import platform, sys

# architecture() анализирует исполняемый файл — это сравнительно дорого.
# Если нужна лишь битность Python-процесса:
is_64 = sys.maxsize > 2**32        # быстро и точно
```

### 6. Внешние подпроцессы и контейнеры

В минимальных Docker-образах (`scratch`, `distroless`) или при урезанном `PATH` функции, дёргающие `uname`/`ver`, могут вести себя нестабильно. Не стройте критичную логику на их непустом результате без обработки исключений.

### 7. Кэширование мешает тестам

```python
import platform
from unittest import mock

# Патчим саму функцию, а не пытаемся «сбросить кэш»:
with mock.patch.object(platform, 'system', return_value='Windows'):
    assert platform.system() == 'Windows'
```

`uname()` кэширует результат, поэтому в тестах патчат конкретные функции `platform`, а не переменные окружения.

### 8. `freedesktop_os_release()` бросает `OSError` вне Linux

```python
import platform

try:
    distro = platform.freedesktop_os_release().get('PRETTY_NAME', '')
except (OSError, AttributeError):  # OSError — нет файла; AttributeError — Python < 3.10
    distro = ''
```

### 9. Удалённые функции

`platform.linux_distribution()` и `platform.dist()` **удалены** начиная с Python 3.8. Не используйте их — замена: `freedesktop_os_release()` или пакет `distro`.

---

## Лучшие практики

1. **Для простых проверок ОС используйте `sys.platform` / `os.name`,** а `platform.*` — когда реально нужны детали (версия, архитектура, билд). Это быстрее и стабильнее.

2. **Всегда обрабатывайте пустые строки и исключения.** Оборачивайте вызовы в защитный код с fallback-значениями.

3. **Нормализуйте архитектуру и имена ОС** через собственные функции-адаптеры — не сравнивайте с «сырыми» значениями напрямую.

4. **Версии Python сравнивайте через `sys.version_info`,** а `platform.python_version()` берите только для отображения.

5. **Битность определяйте через `sys.maxsize > 2**32`,** если не нужен анализ конкретного файла.

6. **В тестах патчьте конкретные функции `platform`** (через `unittest.mock`), а не полагайтесь на окружение.

7. **Для глубокой информации о CPU/дистрибутиве** используйте специализированные пакеты (`py-cpuinfo`, `distro`) — `platform` для этого недостаточно надёжен.

8. **Инкапсулируйте определение платформы в один модуль** проекта, чтобы остальной код зависел от ваших нормализованных констант, а не от `platform` напрямую.

```python
# platform_info.py — единая точка определения окружения проекта
import platform
import sys

IS_WINDOWS = sys.platform.startswith('win')
IS_MACOS = sys.platform == 'darwin'
IS_LINUX = sys.platform.startswith('linux')

def arch() -> str:
    m = platform.machine().lower()
    return {'x86_64': 'x64', 'amd64': 'x64',
            'arm64': 'arm64', 'aarch64': 'arm64'}.get(m, m or 'unknown')

IS_64BIT = sys.maxsize > 2 ** 32
```

---

## Шпаргалка

```python
import platform, sys, os

# ── ОС ───────────────────────────────────────────────────────────
platform.system()            # 'Linux' / 'Darwin' / 'Windows' / ''
platform.release()           # релиз/версия ядра ('6.5.0-35-generic', '23.5.0', '10')
platform.version()           # подробная версия/билд ОС
platform.platform()          # сводная строка: 'macOS-14.5-arm64-arm-64bit'
platform.platform(terse=True)    # короткая: 'macOS-14.5'
platform.platform(aliased=True)  # с псевдонимами имён ОС

# ── Железо / архитектура ─────────────────────────────────────────
platform.machine()           # 'x86_64' / 'AMD64' / 'arm64' / 'aarch64' / ''
platform.processor()         # модель CPU (часто '' на Linux!)
platform.architecture()      # ('64bit', '') — битность ИНТЕРПРЕТАТОРА
platform.node()              # hostname (может быть ''); надёжнее socket.gethostname()

# ── Python ───────────────────────────────────────────────────────
platform.python_version()        # '3.12.3' (str)
platform.python_version_tuple()  # ('3', '12', '3') — кортеж СТРОК
platform.python_implementation() # 'CPython' / 'PyPy' / 'Jython' / 'IronPython'
platform.python_compiler()       # 'Clang 15.0.0 ...' / 'GCC 11.4.0' / 'MSC v.1929 ...'
platform.python_build()          # ('main', 'Apr  9 2024 08:09:14')

# ── uname: всё сразу (namedtuple) ────────────────────────────────
u = platform.uname()
u.system, u.node, u.release, u.version, u.machine, u.processor

# ── Платформенно-специфичные ─────────────────────────────────────
platform.mac_ver()           # ('14.5', ('', '', ''), 'arm64')  — версия macOS
platform.win32_ver()         # ('10', '10.0.22631', 'SP0', 'Multiprocessor Free')
platform.win32_edition()     # 'Professional' (Python 3.8+, Windows)
platform.libc_ver()          # ('glibc', '2.35') — Linux
platform.freedesktop_os_release()  # dict с NAME/VERSION_ID/ID (Python 3.10+, Linux)

# ── Сравнение с sys / os ─────────────────────────────────────────
sys.platform                 # 'linux' / 'darwin' / 'win32' (фиксируется при сборке)
os.name                      # 'posix' / 'nt'
sys.version_info >= (3, 11)  # ПРАВИЛЬНОЕ сравнение версий Python
sys.maxsize > 2 ** 32        # True для 64-битного Python (быстро)


# ── Готовый сбор диагностики (устойчивый к ошибкам) ──────────────
def collect_env() -> dict:
    """Безопасно собрать снимок окружения для логов/баг-репортов."""
    def safe(fn, default=''):
        try:
            return fn() or default
        except Exception:
            return default

    info = {
        'platform': safe(platform.platform),
        'system': safe(platform.system),
        'release': safe(platform.release),
        'machine': safe(platform.machine),
        'processor': safe(platform.processor),
        'node': safe(platform.node),
        'python_version': safe(platform.python_version),
        'python_implementation': safe(platform.python_implementation),
        'python_compiler': safe(platform.python_compiler),
        'is_64bit': sys.maxsize > 2 ** 32,
    }
    # Версия macOS отдельно (т.к. release() даёт версию ядра Darwin):
    if platform.system() == 'Darwin':
        info['macos_version'] = safe(lambda: platform.mac_ver()[0])
    # Дистрибутив Linux:
    if sys.platform.startswith('linux'):
        try:
            os_rel = platform.freedesktop_os_release()
            info['distro'] = os_rel.get('PRETTY_NAME', '')
            info['libc'] = '-'.join(platform.libc_ver())
        except (OSError, AttributeError):
            pass
    return info


if __name__ == '__main__':
    for k, v in collect_env().items():
        print(f'{k:24}: {v}')
```

### Памятка по выбору инструмента

| Задача | Чем решать |
|--------|-----------|
| Простая проверка ОС в горячем коде | `sys.platform`, `os.name` |
| Подробная инфо об ОС для логов | `platform.platform()`, `platform.uname()` |
| Версия macOS (маркетинговая) | `platform.mac_ver()[0]` |
| Дистрибутив Linux | `platform.freedesktop_os_release()` |
| Архитектура (нормализованная) | своя обёртка над `platform.machine()` |
| Битность Python-процесса | `sys.maxsize > 2**32` |
| Сравнение версий Python | `sys.version_info` |
| Реальные данные о CPU | `py-cpuinfo`, `/proc/cpuinfo` |
