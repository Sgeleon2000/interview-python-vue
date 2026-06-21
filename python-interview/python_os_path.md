# os.path — подготовка к собеседованию

## Что это и зачем

`os.path` — это подмодуль `os` стандартной библиотеки для **манипуляций с путями файловой системы как со строками**. Появился задолго до `pathlib` и до сих пор широко встречается в существующем коде, библиотеках и легаси.

Зачем нужен:
- Кроссплатформенная склейка/разбор путей с учётом разделителей ОС (`/` на POSIX, `\` на Windows).
- Получение информации о пути: существует ли, абсолютный ли, какое расширение и т.д.
- Лёгкий, без создания объектов — функции принимают и возвращают **строки** (или bytes).

Важно понимать: `os.path` работает с путём как с текстом. Большинство функций (`join`, `split`, `splitext`, `basename`, `dirname`, `normpath`) — **чисто строковые** и не обращаются к диску. Лишь некоторые (`exists`, `isfile`, `isdir`, `getsize`, `abspath`) реально трогают файловую систему.

```python
import os.path

path = os.path.join("/var", "log", "app.log")   # '/var/log/app.log'
name = os.path.basename(path)                     # 'app.log'
```

В новом коде предпочтительнее `pathlib`, но знать `os.path` обязательно — его много в проде.

## Ключевые концепции

- **Пути — строки или bytes.** Функции полиморфны: дайте `str` — получите `str`, дайте `bytes` — получите `bytes`.
- **Платформозависимость.** `os.path` — это алиас на `posixpath` (POSIX) или `ntpath` (Windows). Разделитель — `os.sep` (`/` или `\`), разделитель путей в `PATH` — `os.pathsep` (`:` или `;`).
- **Строковые vs обращающиеся к ФС функции.** `join/split/splitext/normpath` не читают диск; `exists/isfile/getsize/abspath/realpath` — читают.
- **`normpath` ≠ `realpath`.** `normpath` нормализует строку (убирает `..`, `.`, лишние слеши) **без** обращения к ФС и без разворачивания симлинков. `realpath` разворачивает симлинки (обращается к ФС).

```python
import os
print(os.sep)       # '/' на Linux/Mac, '\\' на Windows
print(os.pathsep)   # ':' на Linux/Mac, ';' на Windows
```

## Основные функции/классы/методы

### join — объединение путей

```python
import os.path

os.path.join("a", "b", "c")          # 'a/b/c'
os.path.join("/a", "b")              # '/a/b'
os.path.join("/a", "/b")            # '/b'   — абсолютный аргумент СБРАСЫВАЕТ предыдущее!
os.path.join("a", "")               # 'a/'   — пустой хвост добавляет разделитель
```

Правило: если любой компонент — абсолютный путь, всё, что было до него, отбрасывается.

### split, splitext, basename, dirname

```python
import os.path

p = "/home/user/report.tar.gz"

os.path.split(p)        # ('/home/user', 'report.tar.gz')  — (голова, хвост)
os.path.dirname(p)      # '/home/user'                     — то же, что split()[0]
os.path.basename(p)     # 'report.tar.gz'                  — то же, что split()[1]

os.path.splitext(p)     # ('/home/user/report.tar', '.gz') — отделяет ПОСЛЕДНЕЕ расширение
os.path.splitext("archive.tar.gz")   # ('archive.tar', '.gz')
os.path.splitext(".bashrc")          # ('.bashrc', '')   — ведущая точка НЕ расширение

# splitdrive (актуально на Windows)
os.path.splitdrive("C:/data/x")      # ('C:', '/data/x')  на Windows
```

Связь: `split()` режет по последнему разделителю на (`dirname`, `basename`); `splitext()` отделяет расширение.

### exists, isfile, isdir, lexists

```python
import os.path

os.path.exists("/etc/hosts")    # True/False — существует ли (файл/папка/симлинк на существующее)
os.path.isfile("/etc/hosts")    # это обычный файл?
os.path.isdir("/etc")           # это директория?
os.path.islink("/some/link")    # это символическая ссылка?
os.path.lexists("/broken/link") # существует ли сама ссылка (даже битая)?

# Важно: exists() возвращает False для битого симлинка, lexists() — True
```

### abspath, realpath, normpath, expanduser, expandvars

```python
import os.path

# abspath: делает абсолютным относительно cwd + нормализует (НЕ разворачивает симлинки)
os.path.abspath("data/../config.ini")   # '/current/dir/config.ini'

# realpath: абсолютный + РАЗВОРАЧИВАЕТ симлинки (обращается к ФС)
os.path.realpath("/var/log")            # например '/private/var/log' на macOS

# normpath: только нормализует СТРОКУ, не трогает ФС, не делает абсолютным
os.path.normpath("a/b/../c/./d")        # 'a/c/d'
os.path.normpath("//a///b")             # '/a/b'  (но '//' в начале POSIX может сохраниться)

# expanduser: разворачивает ~ в домашнюю директорию
os.path.expanduser("~/.config")         # '/home/user/.config'

# expandvars: подставляет переменные окружения
os.path.expandvars("$HOME/data")        # '/home/user/data'
```

Сравнение трёх «нормализаторов»:
- `normpath` — чисто строковая нормализация (`..`, `.`, слеши), без ФС, без абсолютизации.
- `abspath` = `normpath(join(cwd, path))` — абсолютный, но симлинки НЕ разворачивает.
- `realpath` — абсолютный И разворачивает симлинки (читает ФС).

### Прочие полезные функции

```python
import os.path

os.path.getsize("file.bin")     # размер в байтах (бросает OSError если нет файла)
os.path.getmtime("file.bin")    # время модификации (unix timestamp)
os.path.getctime("file.bin")    # время создания (или смены метаданных на Unix)
os.path.getatime("file.bin")    # время доступа

os.path.commonpath(["/a/b/c", "/a/b/d"])    # '/a/b'  — общий префикс как валидный путь
os.path.commonprefix(["/a/bc", "/a/bd"])    # '/a/b'  — ПОБУКВЕННЫЙ префикс (может быть невалиден!)

os.path.relpath("/a/b/c", "/a/b")           # 'c'  — относительный путь
os.path.samefile("a.txt", "link_to_a.txt")  # True — указывают на один inode?
os.path.isabs("/etc")                        # True — абсолютный ли путь
```

## Частые вопросы на собеседовании

**Q1: Что вернёт `os.path.join("/a", "/b")`?**
`'/b'`. Абсолютный аргумент сбрасывает всё, что было до него. Частый баг: если в `join` случайно попадёт строка, начинающаяся с `/`, путь «перепрыгнет» в корень.

**Q2: В чём разница между `normpath`, `abspath` и `realpath`?**
`normpath` — чисто строковая нормализация (убирает `..`, `.`, дубли слешей), не обращается к ФС. `abspath` — делает абсолютным относительно cwd + нормализует, но симлинки не разворачивает. `realpath` — абсолютный и разворачивает симлинки (читает диск).

**Q3: Что вернёт `os.path.splitext("archive.tar.gz")`?**
`('archive.tar', '.gz')` — отделяется только **последнее** расширение. Чтобы убрать оба, нужно вызвать дважды или работать с `pathlib.Path.suffixes`.

**Q4: Как `splitext` обрабатывает скрытые файлы вроде `.bashrc`?**
`('.bashrc', '')` — ведущая точка не считается началом расширения. А `os.path.splitext("dir.with.dot/.bashrc")` тоже даст пустое расширение для имени.

**Q5: Чем `split` отличается от `splitext`?**
`split` режет по последнему разделителю каталогов на (`dirname`, `basename`). `splitext` режет имя на (`корень`, `расширение`). Это разные операции.

**Q6: `exists` vs `lexists` для битого симлинка?**
`exists` идёт по ссылке и вернёт `False`, если цель не существует. `lexists` проверяет саму ссылку и вернёт `True` даже для битой.

**Q7: Чем `commonpath` отличается от `commonprefix`?**
`commonprefix` сравнивает строки **посимвольно** и может вернуть невалидный путь (`/a/b` для `/a/bc` и `/a/bd`). `commonpath` работает с компонентами пути и возвращает корректный общий каталог; бросает `ValueError` при смешивании абсолютных и относительных путей или разных дисков.

**Q8: Эти функции читают диск?**
`join/split/splitext/basename/dirname/normpath/isabs` — нет, чисто строковые. `exists/isfile/isdir/getsize/realpath/abspath(частично через cwd)/samefile` — да, обращаются к ФС.

**Q9: Как развернуть `~` и переменные окружения в пути?**
`os.path.expanduser("~/x")` для `~`, `os.path.expandvars("$VAR/x")` для переменных. Часто комбинируют: `expanduser(expandvars(p))`.

**Q10: Почему `os.path` кроссплатформенный?**
Это алиас на `posixpath` или `ntpath` в зависимости от ОС. Поэтому он использует правильный `os.sep`. При этом можно импортировать `posixpath`/`ntpath` напрямую, чтобы разбирать пути «чужой» ОС.

## Подводные камни (gotchas)

1. **Абсолютный аргумент в `join` сбрасывает путь.** `join("/base", user_input)` опасен, если `user_input` начинается с `/` (или диска на Windows).

2. **`splitext` берёт только последнее расширение.** Для `.tar.gz` нужно два вызова или `pathlib`.

3. **`commonprefix` возвращает строковый, а не путевой префикс.** Может дать невалидный путь. Используйте `commonpath`, если нужен реальный каталог.

4. **`normpath` не разворачивает симлинки** и может изменить смысл пути с симлинками/`..`. Для безопасности используйте `realpath`.

5. **`abspath` зависит от текущего cwd.** Результат меняется, если изменился рабочий каталог (`os.chdir`). Это источник трудноуловимых багов.

6. **`getsize`/`getmtime` бросают `OSError`/`FileNotFoundError`**, если файла нет — в отличие от `exists`, который просто вернёт `False`.

7. **`normpath` на Windows меняет `/` на `\`.** Результаты различаются между платформами — не хардкодьте ожидаемую строку в тестах.

8. **`expandvars` молча оставляет нераспознанные переменные.** `$NOPE/x` останется как есть, ошибки не будет.

9. **`normpath("//a")` на POSIX** может сохранить двойной слеш в начале (это разрешено стандартом POSIX для специальных путей).

10. **Смешивание `str` и `bytes`.** Нельзя передать в одну функцию и `str`, и `bytes` — будет `TypeError`.

## Лучшие практики

- В новом коде предпочитайте `pathlib`; `os.path` — для легаси и точечных операций.
- Никогда не делайте `join(base, untrusted)` без проверки, что второй аргумент относительный и не выходит за пределы (защита от path traversal).
- Используйте `realpath` + проверку `commonpath`, чтобы убедиться, что путь внутри разрешённого каталога.
- Для нормализации перед сравнением путей используйте `realpath` (учитывает симлинки), а не только `normpath`.
- Указывайте абсолютные пути там, где возможно, чтобы не зависеть от `cwd`.
- Используйте `os.sep`/`os.pathsep` вместо хардкода `/` и `:`.
- Для разбора `PATH` используйте `os.pathsep`, не `:`.

## Шпаргалка

```python
import os.path as p

# Объединение/разбор (строковые, без ФС)
p.join("a", "b", "c")          # 'a/b/c'  (абсолютный сбрасывает!)
p.split("/a/b/c")              # ('/a/b', 'c')
p.dirname("/a/b/c")            # '/a/b'
p.basename("/a/b/c")          # 'c'
p.splitext("f.tar.gz")        # ('f.tar', '.gz')   (последнее расширение)
p.splitdrive("C:/x")          # ('C:', '/x')  (Windows)

# Нормализация
p.normpath("a/b/../c")        # 'a/c'   (только строка)
p.abspath("x")                # cwd + x, нормализ., без симлинков
p.realpath("x")               # абс. + разворот симлинков
p.expanduser("~/x")           # домашняя
p.expandvars("$HOME/x")       # переменные окружения
p.relpath("/a/b/c", "/a")     # 'b/c'
p.commonpath(["/a/b", "/a/c"])# '/a'

# ФС-проверки (читают диск)
p.exists(x); p.isfile(x); p.isdir(x); p.islink(x); p.lexists(x)
p.isabs(x); p.samefile(a, b)
p.getsize(x); p.getmtime(x); p.getatime(x); p.getctime(x)

# Платформа
import os
os.sep        # '/' или '\\'
os.pathsep    # ':' или ';'
```
