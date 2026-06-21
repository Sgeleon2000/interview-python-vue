# shutil — подготовка к собеседованию

## Что это и зачем

`shutil` (shell utilities) — модуль стандартной библиотеки для **высокоуровневых операций с файлами и коллекциями файлов**: копирование, перемещение, удаление деревьев, архивация, информация о диске. Это «питоновский аналог» команд `cp`, `mv`, `rm -rf`, `tar`, `df`, `which`.

Где `os`/`pathlib` дают атомарные операции (создать файл, удалить один файл), `shutil` оперирует **целыми деревьями и потоками**: скопировать папку рекурсивно, переместить через границу файловых систем, удалить непустую директорию, упаковать каталог в архив.

```python
import shutil

shutil.copy2("src.txt", "dst.txt")            # копия с метаданными
shutil.copytree("project", "backup")           # рекурсивная копия дерева
shutil.move("old/dir", "new/dir")              # перемещение
shutil.rmtree("temp_dir")                       # рекурсивное удаление
shutil.make_archive("backup", "zip", "project")# создать backup.zip
```

## Ключевые концепции

- **Высокоуровневость.** `shutil` строится поверх `os` и работает с файловыми объектами/деревьями целиком.
- **Сохранение метаданных.** Разные функции копирования сохраняют разный объём метаданных (права, время, флаги). `copy` < `copy2`.
- **Кроссплатформенность с оговорками.** Метаданные (права, владелец, ACL) сохраняются по-разному на разных ОС; что-то теряется.
- **Перемещение через ФС.** `move` умно выбирает: `rename` (быстро, в пределах одной ФС) или копирование+удаление (между ФС).
- **Безопасность удаления.** `rmtree` необратимо удаляет всё дерево — крайне опасная операция, требует осторожности.

## Основные функции/классы/методы

### Копирование файлов: copyfile, copy, copy2, copymode, copystat

```python
import shutil

# copyfileobj — копирует из одного открытого файлового объекта в другой (потоково)
with open("src.bin", "rb") as fsrc, open("dst.bin", "wb") as fdst:
    shutil.copyfileobj(fsrc, fdst, length=64 * 1024)  # length — размер буфера

# copyfile — копирует ТОЛЬКО содержимое (dst должен быть именем файла, не папкой)
shutil.copyfile("src.txt", "dst.txt")

# copy — содержимое + права доступа (mode). dst может быть директорией.
shutil.copy("src.txt", "target_dir/")     # создаст target_dir/src.txt

# copy2 — содержимое + ВСЕ метаданные (права, время доступа/модификации и т.д.)
shutil.copy2("src.txt", "target_dir/")    # «максимально полная» копия — предпочтительна для бэкапов

# copymode / copystat — копируют только метаданные с одного файла на другой
shutil.copymode("ref.txt", "dst.txt")     # только права
shutil.copystat("ref.txt", "dst.txt")     # права + время + флаги
```

Иерархия «полноты»: `copyfile` (только данные) < `copy` (+ права) < `copy2` (+ время и прочее).

### Копирование дерева: copytree

```python
import shutil

# Рекурсивно копирует всё дерево из src в dst
shutil.copytree("project", "project_backup")

# dirs_exist_ok=True (Python 3.8+) — не падать, если dst существует (сливает)
shutil.copytree("project", "existing_dst", dirs_exist_ok=True)

# ignore — функция-фильтр; ignore_patterns создаёт такой фильтр
shutil.copytree(
    "project",
    "backup",
    ignore=shutil.ignore_patterns("*.pyc", "__pycache__", ".git"),
)

# copy_function — какой функцией копировать файлы (по умолчанию copy2)
shutil.copytree("a", "b", copy_function=shutil.copy)

# symlinks=True — копировать симлинки как симлинки, а не их содержимое
shutil.copytree("a", "b", symlinks=True)
```

### Перемещение: move

```python
import shutil

# Перемещает файл или дерево. Если на той же ФС — быстрый rename;
# если между ФС — копирование (copy2) + удаление источника.
shutil.move("report.txt", "archive/report.txt")
shutil.move("src_dir", "dst_dir")

# Если dst — существующая директория, источник переместится ВНУТРЬ неё
shutil.move("file.txt", "existing_dir/")   # -> existing_dir/file.txt
```

### Удаление дерева: rmtree

```python
import shutil

# Рекурсивно удаляет директорию со всем содержимым (как rm -rf). НЕОБРАТИМО!
shutil.rmtree("temp_dir")

# ignore_errors=True — игнорировать ошибки (например, нет прав)
shutil.rmtree("temp_dir", ignore_errors=True)

# onexc (3.12+) / onerror (<3.12) — обработчик ошибок (например, снять read-only)
import os, stat
def handle_remove_readonly(func, path, exc):
    os.chmod(path, stat.S_IWRITE)   # снимаем флаг "только чтение"
    func(path)                       # повторяем операцию
shutil.rmtree("temp_dir", onexc=handle_remove_readonly)
```

### Архивация: make_archive, unpack_archive

```python
import shutil

# make_archive(base_name, format, root_dir) -> путь к созданному архиву
# Упакует содержимое root_dir в base_name.<ext>
archive = shutil.make_archive(
    base_name="backups/project_2026",   # имя БЕЗ расширения
    format="zip",                        # 'zip', 'tar', 'gztar', 'bztar', 'xztar'
    root_dir="project",                  # что архивировать
)
print(archive)   # 'backups/project_2026.zip'

# Список доступных форматов
print(shutil.get_archive_formats())     # [('bztar', ...), ('gztar', ...), ('zip', ...), ...]

# Распаковка (формат определяется по расширению или задаётся явно)
shutil.unpack_archive("project_2026.zip", "restore_dir")
shutil.unpack_archive("data.tar.gz", "out", format="gztar")
```

### Информация о диске и поиск программ

```python
import shutil

# disk_usage возвращает namedtuple(total, used, free) в БАЙТАХ
usage = shutil.disk_usage("/")
print(usage.total, usage.used, usage.free)
print(f"Свободно: {usage.free / 1e9:.1f} ГБ")

# which — путь к исполняемому файлу (аналог Unix-команды which); None если не найден
print(shutil.which("python3"))     # '/usr/bin/python3' или None
print(shutil.which("git"))

# get_terminal_size — размеры терминала (columns, lines)
print(shutil.get_terminal_size())  # os.terminal_size(columns=80, lines=24)

# chown — сменить владельца/группу (только Unix)
# shutil.chown("file.txt", user="www-data", group="www-data")
```

## Частые вопросы на собеседовании

**Q1: В чём разница между `copy`, `copy2` и `copyfile`?**
`copyfile` копирует только содержимое (dst обязан быть путём файла). `copy` копирует содержимое + права (mode), dst может быть директорией. `copy2` дополнительно сохраняет все доступные метаданные (время доступа/модификации, флаги). Для бэкапов используют `copy2`.

**Q2: Как `move` ведёт себя между разными файловыми системами?**
В пределах одной ФС `move` использует быстрый `os.rename`. Между разными ФС `rename` невозможен, поэтому `move` копирует файл/дерево (через `copy2`) и затем удаляет источник. Это медленнее и не атомарно.

**Q3: Чем `rmtree` отличается от `os.rmdir` и `pathlib.Path.rmdir`?**
`os.rmdir`/`Path.rmdir` удаляют только **пустую** директорию. `shutil.rmtree` удаляет директорию **рекурсивно** со всем содержимым (как `rm -rf`). Операция необратима.

**Q4: Что произойдёт с `copytree`, если целевая папка уже существует?**
До Python 3.8 — `FileExistsError`. С 3.8 можно передать `dirs_exist_ok=True`, тогда содержимое сольётся в существующую папку.

**Q5: Как исключить файлы при `copytree`?**
Через параметр `ignore`, обычно `ignore=shutil.ignore_patterns("*.pyc", "__pycache__", ".git")`. Это функция, возвращающая имена для пропуска.

**Q6: Как создать zip/tar архив каталога одной командой?**
`shutil.make_archive("name", "zip", "dir")` создаст `name.zip`. Поддерживаемые форматы: `zip`, `tar`, `gztar`, `bztar`, `xztar`. Имя задаётся **без** расширения — оно добавится автоматически.

**Q7: Как программно узнать свободное место на диске?**
`shutil.disk_usage(path)` возвращает namedtuple с полями `total`, `used`, `free` в байтах.

**Q8: Зачем нужен `shutil.which`?**
Находит полный путь к исполняемому файлу в `PATH` (как Unix `which`). Возвращает `None`, если не найден. Полезно для проверки наличия внешних программ перед `subprocess`.

**Q9: Что делает `copyfileobj` и когда он полезен?**
Копирует данные между двумя уже открытыми файловыми объектами потоково, буфером. Полезен, когда у вас есть file-like объекты (например, сетевой поток, `BytesIO`, ответ HTTP), а не пути на диске.

**Q10: Как `rmtree` обработать файлы «только для чтения» на Windows?**
По умолчанию `rmtree` упадёт на read-only файлах. Нужно передать обработчик `onexc` (3.12+) / `onerror` (раньше), который снимает флаг через `os.chmod(path, stat.S_IWRITE)` и повторяет операцию.

**Q11: Сохраняет ли `copy2` владельца файла?**
Нет. `copy2` сохраняет права (mode), время и флаги, но **не** владельца/группу. Для смены владельца есть отдельный `shutil.chown` (только Unix, требует прав).

## Подводные камни (gotchas)

1. **`rmtree` необратим и опасен.** Ошибка в пути (например, пустая переменная → `rmtree("")` или `rmtree("/")`) может удалить лишнее. Всегда валидируйте путь.

2. **`copytree` падает на существующей цели** без `dirs_exist_ok=True` (до 3.8 такого параметра нет вовсе).

3. **`make_archive` принимает имя БЕЗ расширения.** Если передать `"backup.zip"`, получите `backup.zip.zip`.

4. **`move` в существующую директорию** перемещает источник **внутрь** неё, а не заменяет. Поведение зависит от того, существует ли `dst`.

5. **`move` не атомарен между ФС** — при сбое можно получить частично скопированные данные.

6. **`copy` не сохраняет время**, только права. Если важны метаданные — берите `copy2`.

7. **`rmtree(ignore_errors=True)` молча проглатывает ошибки** — можно подумать, что удаление прошло, хотя часть файлов осталась.

8. **`disk_usage` на несуществующем пути** бросает `FileNotFoundError`.

9. **`copytree` с симлинками.** По умолчанию (`symlinks=False`) копирует **содержимое** цели симлинка. Битый симлинк вызовет ошибку, если не задан `ignore_dangling_symlinks=True`.

10. **`make_archive` меняет cwd внутри** (через `root_dir`/`base_dir`) — будьте внимательны с относительными путями в многопоточном коде.

11. **`which` зависит от `PATH` и расширений (`PATHEXT` на Windows)** — результат платформозависим.

## Лучшие практики

- Для копий, где важны метаданные/время, используйте `copy2`, а не `copy`.
- Перед `rmtree` явно проверяйте, что путь не пустой, абсолютный и внутри ожидаемого каталога.
- Для перемещения между потенциально разными ФС используйте `shutil.move`, а не `os.rename`.
- В `copytree` всегда задавайте `ignore_patterns` для мусора (`__pycache__`, `.git`, `*.pyc`).
- Передавайте в `make_archive` имя без расширения и явно указывайте формат.
- Проверяйте наличие внешних утилит через `shutil.which(...)` перед запуском `subprocess`.
- Для больших файлов используйте `copyfileobj` с разумным `length` (например, 1 МБ) для контроля памяти.
- На Windows готовьте обработчик `onexc`/`onerror` для `rmtree` (read-only файлы).
- Логируйте операции удаления/перемещения — они необратимы.

## Шпаргалка

```python
import shutil

# Копирование файлов
shutil.copyfile(src, dst)               # только содержимое
shutil.copy(src, dst)                   # + права (dst может быть папкой)
shutil.copy2(src, dst)                  # + все метаданные (для бэкапов)
shutil.copyfileobj(fsrc, fdst, length)  # между открытыми файлами

# Дерево
shutil.copytree(src, dst, dirs_exist_ok=True,
                ignore=shutil.ignore_patterns("*.pyc", ".git"))
shutil.move(src, dst)                   # перемещение (умно между ФС)
shutil.rmtree(path, onexc=handler)      # рекурсивное удаление (опасно!)

# Архивы
shutil.make_archive("name", "zip", "dir")   # -> name.zip (имя без расширения!)
shutil.unpack_archive("a.tar.gz", "out")
shutil.get_archive_formats()

# Система
shutil.disk_usage("/")                  # (total, used, free) в байтах
shutil.which("git")                     # путь к программе или None
shutil.get_terminal_size()              # (columns, lines)
```
