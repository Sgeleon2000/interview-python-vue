# pathlib — подготовка к собеседованию

## Что это и зачем

`pathlib` — это модуль стандартной библиотеки (с Python 3.4), предоставляющий **объектно-ориентированный** интерфейс для работы с путями файловой системы. Он заменяет процедурный модуль `os.path`, где пути — это просто строки.

Главная идея: путь — это объект (`Path`), у которого есть методы и свойства, а не строка, которую нужно постоянно «склеивать» функциями.

Зачем нужен:
- Читаемость: `Path("/etc") / "nginx" / "nginx.conf"` вместо `os.path.join("/etc", "nginx", "nginx.conf")`.
- Кроссплатформенность: автоматическая обработка разделителей (`/` vs `\`).
- Богатый API: чтение/запись файлов, glob, проверки существования, манипуляции с расширениями — всё в одном объекте.
- Меньше импортов: `os`, `os.path`, `glob`, частично `shutil` заменяются одним `pathlib`.

```python
from pathlib import Path

# Старый стиль (os.path) — путь как строка
import os.path
config = os.path.join(os.path.expanduser("~"), ".config", "app", "settings.ini")

# Новый стиль (pathlib) — путь как объект
config = Path.home() / ".config" / "app" / "settings.ini"
```

## Ключевые концепции

### Иерархия классов

```
PurePath                       (чистые пути, без доступа к ФС)
├── PurePosixPath              (POSIX-семантика: /)
└── PureWindowsPath            (Windows-семантика: \)

Path(PurePath)                 (конкретные пути, с доступом к ФС)
├── PosixPath
└── WindowsPath
```

- **`PurePath`** — только манипуляции с путём как с текстом (разбор, объединение, сравнение). Не обращается к диску. Полезно для путей «другой» ОС или для логики, не требующей ФС.
- **`Path`** — добавляет методы, которые **реально работают с файловой системой**: `exists()`, `read_text()`, `mkdir()`, `glob()` и т.д.

При создании `Path()` автоматически выбирается `PosixPath` или `WindowsPath` в зависимости от текущей ОС.

```python
from pathlib import Path, PurePath, PurePosixPath, PureWindowsPath

p = Path("data/file.txt")           # На Linux/Mac -> PosixPath, на Windows -> WindowsPath
print(type(p))                       # <class 'pathlib.PosixPath'>

# Чистый путь чужой ОС можно разбирать на любой платформе
win = PureWindowsPath(r"C:\Users\Bob\file.txt")
print(win.parts)                     # ('C:\\', 'Users', 'Bob', 'file.txt')
print(win.drive)                     # 'C:'
```

### Компоненты пути

```python
p = Path("/home/user/archive/report.tar.gz")

print(p.parts)        # ('/', 'home', 'user', 'archive', 'report.tar.gz') — кортеж компонентов
print(p.name)         # 'report.tar.gz'   — имя файла с расширением
print(p.stem)         # 'report.tar'      — имя без последнего суффикса
print(p.suffix)       # '.gz'             — последнее расширение
print(p.suffixes)     # ['.tar', '.gz']   — все расширения списком
print(p.parent)       # /home/user/archive — родительская папка
print(p.parents[1])   # /home/user        — родители по индексу
print(p.anchor)       # '/'               — корень+диск
print(p.root)         # '/'
print(p.drive)        # ''  (на Windows был бы 'C:')
```

## Основные функции/классы/методы

### Создание путей и оператор `/`

Оператор `/` перегружен (`__truediv__`) и объединяет части пути — это «фишка» pathlib.

```python
from pathlib import Path

base = Path("/var/log")
log = base / "app" / "error.log"     # /var/log/app/error.log
print(log)

# Можно слева ставить строку, если справа Path
full = "/etc" / Path("hosts")        # /etc/hosts (работает через __rtruediv__)

# joinpath — то же самое, но методом
log2 = base.joinpath("app", "error.log")

# Текущая и домашняя директории
print(Path.cwd())     # текущая рабочая директория
print(Path.home())    # домашняя директория пользователя
```

### Проверки и информация о файле

```python
p = Path("data/file.txt")

p.exists()        # существует ли (файл или папка)
p.is_file()       # это обычный файл?
p.is_dir()        # это директория?
p.is_symlink()    # это символическая ссылка?
p.is_absolute()   # абсолютный ли путь?

# stat() возвращает метаданные (размер, время изменения и т.д.)
st = p.stat()
print(st.st_size)            # размер в байтах
print(st.st_mtime)           # время последней модификации (unix timestamp)

# Удобные методы абсолютизации
print(p.absolute())          # абсолютный путь (НЕ резолвит .. и симлинки)
print(p.resolve())           # КАНОНИЧЕСКИЙ путь: резолвит симлинки, .., делает абсолютным
print(p.resolve(strict=True))# с strict=True кинет FileNotFoundError, если не существует
```

### Чтение и запись

```python
p = Path("note.txt")

# Запись (создаёт/перезаписывает файл целиком)
p.write_text("Привет, мир!\n", encoding="utf-8")
p.write_bytes(b"\x00\x01\x02")

# Чтение
content = p.read_text(encoding="utf-8")    # строка целиком
data = p.read_bytes()                       # bytes целиком

# Для построчной/потоковой работы — open() возвращает обычный файловый объект
with p.open("r", encoding="utf-8") as f:
    for line in f:
        process(line)

# Современный способ читать построчно (3.13+) — но обычно через open()
```

### Создание и удаление

```python
d = Path("output/sub/deep")

# Создать директорию
d.mkdir(parents=True, exist_ok=True)
#   parents=True   — создаёт все промежуточные папки (как mkdir -p)
#   exist_ok=True  — не падать, если папка уже есть

# Создать пустой файл (или обновить mtime)
Path("output/marker.txt").touch(exist_ok=True)

# Удаление
Path("output/marker.txt").unlink(missing_ok=True)  # удалить файл; missing_ok с 3.8
Path("output/sub/deep").rmdir()                     # удалить ПУСТУЮ директорию
# Для рекурсивного удаления непустой папки нужен shutil.rmtree()
```

### Переименование, перемещение, замена

```python
p = Path("old.txt")
p.rename("new.txt")             # переименовать/переместить (вернёт новый Path в 3.8+)
p.replace("target.txt")         # как rename, но атомарно перезапишет цель, если она есть

# Полезные «производные» методы — возвращают НОВЫЙ Path, файл не трогают
p = Path("/data/report.txt")
print(p.with_name("summary.txt"))    # /data/summary.txt
print(p.with_suffix(".md"))          # /data/report.md
print(p.with_stem("final"))          # /data/final.txt   (Python 3.9+)
```

### glob и rglob

```python
project = Path("project")

# glob: поиск по шаблону в ОДНОМ уровне (если нет **)
for py in project.glob("*.py"):
    print(py)

# Рекурсивный поиск: ** означает «любое число вложенных папок»
for py in project.glob("**/*.py"):
    print(py)

# rglob("pattern") — короткая запись для glob("**/pattern")
for py in project.rglob("*.py"):
    print(py)

# Возвращается генератор -> ленивая итерация. Чтобы получить список:
all_tests = list(project.rglob("test_*.py"))
```

Шаблоны glob:
- `*` — любое число символов (кроме `/`)
- `?` — один символ
- `[abc]` — один из символов
- `**` — рекурсивно по поддиректориям (только если включён как отдельный сегмент)

### Итерация по директории

```python
d = Path(".")

# iterdir() — все элементы директории (не рекурсивно), быстрее glob("*")
for child in d.iterdir():
    kind = "DIR " if child.is_dir() else "FILE"
    print(kind, child.name)
```

### relative_to и match

```python
root = Path("/srv/www")
full = Path("/srv/www/static/css/main.css")

print(full.relative_to(root))         # static/css/main.css
# Если full не внутри root -> ValueError

print(full.match("*.css"))            # True  — сопоставление по хвосту пути
print(full.match("static/*/*.css"))   # True
```

## Частые вопросы на собеседовании

**Q1: В чём разница между `PurePath` и `Path`?**
`PurePath` выполняет только текстовые операции с путём (разбор, объединение, сравнение) и **не обращается** к файловой системе. `Path` — наследник, который добавляет операции, реально читающие/изменяющие диск (`exists`, `read_text`, `mkdir`, `glob`). `PurePath` нужен, когда ФС недоступна или вы манипулируете путём «чужой» ОС.

**Q2: Чем `resolve()` отличается от `absolute()`?**
`absolute()` просто делает путь абсолютным относительно `cwd`, но **не нормализует** `..` и не разворачивает симлинки. `resolve()` возвращает канонический путь: разворачивает симлинки, убирает `.` и `..`, делает абсолютным. Для надёжного сравнения путей используйте `resolve()`.

**Q3: Чем `glob()` отличается от `rglob()`?**
`rglob(pat)` эквивалентен `glob("**/" + pat)` — рекурсивный поиск по всем поддиректориям. `glob(pat)` без `**` ищет только в текущем уровне. Оба возвращают генератор.

**Q4: `stem`, `suffix`, `suffixes`, `name` — что есть что для `archive.tar.gz`?**
`name = "archive.tar.gz"`, `suffix = ".gz"` (последнее), `suffixes = ['.tar', '.gz']`, `stem = "archive.tar"` (имя без последнего суффикса). Чтобы получить «чистое» имя без всех расширений, нужно убирать суффиксы вручную.

**Q5: Как объединить пути и почему работает `/`?**
Оператор `/` перегружен через `__truediv__`/`__rtruediv__`. `Path("a") / "b" / "c"` создаёт `Path("a/b/c")`. Это синтаксический сахар для `joinpath`.

**Q6: Что вернёт `Path("/a") / "/b"`?**
`Path("/b")`. Если правый операнд — **абсолютный** путь, он полностью заменяет левый. Это частый источник багов: если в join попадёт абсолютный путь, начало отбрасывается. То же поведение у `os.path.join`.

**Q7: Как рекурсивно удалить непустую директорию средствами pathlib?**
Нельзя напрямую — `rmdir()` удаляет только пустую папку, `unlink()` — только файл. Нужно либо рекурсивно обойти и удалить всё, либо использовать `shutil.rmtree(path)`.

**Q8: `glob` возвращает список или генератор?**
Генератор (точнее, итератор). Это позволяет лениво обрабатывать огромные деревья. Для повторного использования или подсчёта оберните в `list()`.

**Q9: В чём преимущества pathlib над os.path?**
ООП-интерфейс (методы вместо вложенных функций), читаемый оператор `/`, кроссплатформенность, объединение возможностей `os`, `os.path`, `glob` в одном объекте, удобные `read_text`/`write_text`, иммутабельность объектов `Path`.

**Q10: Path иммутабелен?**
Да. Методы вроде `with_suffix`, `with_name`, `/` не меняют исходный объект, а возвращают новый. Это делает `Path` хешируемым (можно класть в `set`/ключи `dict`) и безопасным.

**Q11: Как получить путь относительно базового?**
`child.relative_to(base)`. Если `child` не находится внутри `base`, бросается `ValueError`. В 3.12+ есть `walk_up=True` для подъёма вверх.

**Q12: Как pathlib работает с кодировками при чтении?**
`read_text(encoding=...)` и `open(encoding=...)` принимают кодировку. По умолчанию используется `locale.getencoding()` (зависит от ОС!). В реальном коде **всегда указывайте `encoding="utf-8"` явно**, иначе на Windows можно получить cp1251/cp1252.

## Подводные камни (gotchas)

1. **Абсолютный правый операнд `/` сбрасывает левый.** `Path("/home") / "/etc"` → `Path("/etc")`. Проверяйте, что присоединяемые части относительны.

2. **`resolve()` до 3.6 и поведение `strict`.** По умолчанию `strict=False` — не падает на несуществующих путях. С `strict=True` бросит `FileNotFoundError`. Не путайте с `absolute()`, который вообще не трогает симлинки.

3. **`glob("**")` без явного сегмента.** `**` рекурсивен только как **отдельный компонент** пути (`**/*.py`). `foo**bar` трактуется как обычный `*`.

4. **Кодировка по умолчанию.** `read_text()` без `encoding` зависит от локали. Всегда указывайте `encoding="utf-8"`.

5. **`rename` через границы файловых систем.** `Path.rename()` может бросить `OSError`, если источник и цель на разных дисках/ФС. Для надёжного перемещения используйте `shutil.move`.

6. **`mkdir` без `parents=True`** бросит `FileNotFoundError`, если родителя нет; без `exist_ok=True` — `FileExistsError`, если папка есть.

7. **`Path("")` → `Path(".")`.** Пустая строка превращается в текущую директорию, не в «ничто».

8. **Сравнение путей.** `Path("a/b") == Path("a/./b")` → разные объекты (`a/b` vs `a/b`? нет — pathlib нормализует одиночные `.`, но не `..`). Для надёжного сравнения резолвите оба через `resolve()`.

9. **`glob` не следует за симлинками-петлями** разумно, но рекурсивный `**` может быть медленным на больших деревьях.

10. **`is_file()`/`exists()` на несуществующем пути** возвращают `False`, не бросают исключение. А `stat()` — бросает `FileNotFoundError`.

## Лучшие практики

- Используйте `pathlib` по умолчанию в новом коде вместо `os.path`.
- **Всегда** передавайте `encoding="utf-8"` в `read_text`/`write_text`/`open`.
- Для сравнения и дедупликации путей применяйте `.resolve()`.
- При создании директорий используйте `mkdir(parents=True, exist_ok=True)` для идемпотентности.
- Принимайте `Path` или `str` в публичных API и сразу оборачивайте: `path = Path(path)`.
- Для перемещения файлов между ФС — `shutil.move`, не `rename`.
- Для удаления дерева — `shutil.rmtree`, а не ручной обход.
- Используйте `with_suffix`/`with_name`/`with_stem` вместо ручной строковой склейки расширений.
- Для типизации параметров используйте `os.PathLike` или `str | Path` (или `from pathlib import Path`).

## Шпаргалка

```python
from pathlib import Path

# Создание
p = Path("a") / "b" / "c.txt"        # объединение
Path.cwd(); Path.home()               # текущая/домашняя

# Компоненты
p.name; p.stem; p.suffix; p.suffixes  # file.txt / file / .txt / [...]
p.parent; p.parents[1]; p.parts       # родители; кортеж частей
p.with_suffix(".md"); p.with_name("x"); p.with_stem("y")

# Проверки
p.exists(); p.is_file(); p.is_dir(); p.is_absolute()
p.stat().st_size                       # размер
p.resolve()                            # канонический абсолютный путь

# Чтение/запись
p.read_text(encoding="utf-8"); p.read_bytes()
p.write_text("...", encoding="utf-8"); p.write_bytes(b"...")
with p.open("r", encoding="utf-8") as f: ...

# Директории
p.mkdir(parents=True, exist_ok=True)
p.iterdir()                            # содержимое (не рекурсивно)
p.rmdir()                              # удалить пустую папку

# Файлы
p.touch(); p.unlink(missing_ok=True)
p.rename("new"); p.replace("dst")      # переименовать/заменить

# Поиск
p.glob("*.py"); p.rglob("*.py"); p.glob("**/*.py")
p.match("*.css"); p.relative_to(base)
```
