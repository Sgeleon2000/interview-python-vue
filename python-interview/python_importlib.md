# importlib — подготовка к собеседованию

## Что это и зачем

`importlib` — пакет стандартной библиотеки, реализующий механизм импорта Python (тот самый, что стоит за инструкцией `import`) и предоставляющий программный API для него. Начиная с Python 3.3, сам оператор `import` реализован поверх `importlib`.

**Зачем нужен:**
- Динамический импорт по имени, известному только в рантайме (`importlib.import_module("pkg.mod")`).
- Перезагрузка модулей без перезапуска процесса (`importlib.reload`) — плагины, hot-reload.
- Доступ к метаданным установленных пакетов: версия, точки входа, зависимости (`importlib.metadata`).
- Чтение ресурсов внутри пакетов (`importlib.resources`).
- Кастомизация импорта: свои загрузчики (loaders) и искатели (finders) — например, импорт из БД, из зашифрованных файлов, по сети.
- Понимание и отладка циклических импортов, `sys.modules`, путей поиска.

## Ключевые концепции

1. **Механизм импорта** состоит из трёх частей:
   - **Finders (искатели)** — находят модуль, возвращают «спецификацию» (`ModuleSpec`). Перечислены в `sys.meta_path`.
   - **Loaders (загрузчики)** — создают и исполняют объект модуля.
   - **Importers** — объекты, совмещающие роли finder + loader.

2. **`sys.modules`** — кэш уже импортированных модулей (словарь `имя → модуль`). При повторном `import` модуль берётся отсюда, а не загружается заново. Это ключ к пониманию циклических импортов и `reload`.

3. **`sys.path` и `sys.meta_path`.**
   - `sys.path` — список путей файловой системы для поиска модулей.
   - `sys.meta_path` — список искателей (path-based finder, frozen, builtin, и ваши кастомные).

4. **`ModuleSpec`** — описание того, как загрузить модуль (имя, loader, origin/путь, является ли пакетом).

5. **Этапы импорта `import x`:**
   1. Проверить `sys.modules` — если есть, вернуть.
   2. Пройти `sys.meta_path` искателями, получить spec.
   3. Создать модуль, поместить его в `sys.modules` **до** исполнения тела.
   4. Исполнить код модуля.
   5. Привязать имя в текущем пространстве имён.

6. **`importlib.metadata`** — доступ к информации о *дистрибутивах* (установленных пакетах): `version`, `metadata`, `entry_points`, `files`, `requires`.

## Основные функции/классы/методы

### import_module — динамический импорт

```python
import importlib

# Эквивалент `import json`, но имя — строка (можно вычислить в рантайме)
json = importlib.import_module("json")
print(json.dumps({"a": 1}))

# Импорт вложенного модуля
mod = importlib.import_module("collections.abc")
print(mod.Mapping)

# Относительный импорт (требует указать package)
# importlib.import_module(".submodule", package="mypackage")

# Типичный паттерн «фабрика по имени из конфига»
def load_plugin(dotted_path):
    """dotted_path вида 'package.module.ClassName'."""
    module_path, _, attr = dotted_path.rpartition(".")
    module = importlib.import_module(module_path)
    return getattr(module, attr)

# handler = load_plugin("logging.handlers.RotatingFileHandler")
```

### reload — перезагрузка модуля

```python
import importlib
import mymodule              # допустим, он уже импортирован

# ... файл mymodule.py изменился на диске ...
importlib.reload(mymodule)   # перечитать и переисполнить код модуля
```

Важные нюансы `reload`:
- Модуль исполняется заново в **том же** объекте модуля (тот же `id`), поэтому существующие ссылки `import mymodule` увидят обновления.
- Но объекты, импортированные через `from mymodule import func`, **не** обновятся — они указывают на старый объект.
- Экземпляры старых классов сохраняют старый класс (их `__class__` не меняется автоматически).

```python
import importlib, sys

# Полная «чистая» перезагрузка пакета иногда требует удаления из sys.modules
def hard_reload(name):
    for key in list(sys.modules):
        if key == name or key.startswith(name + "."):
            del sys.modules[key]
    return importlib.import_module(name)
```

### importlib.util — низкоуровневые утилиты

```python
import importlib.util
import sys

# Импорт модуля из произвольного файла по пути
def import_from_path(module_name, file_path):
    spec = importlib.util.spec_from_file_location(module_name, file_path)
    module = importlib.util.module_from_spec(spec)
    sys.modules[module_name] = module      # важно: до exec_module
    spec.loader.exec_module(module)        # исполнить код модуля
    return module

# mod = import_from_path("config", "/etc/app/config.py")

# Проверить, найдётся ли модуль, не импортируя его
spec = importlib.util.find_spec("numpy")
print("numpy установлен:" , spec is not None)
```

### Ленивый импорт через LazyLoader

```python
import importlib.util
import sys

def lazy_import(name):
    """Модуль реально загрузится при первом обращении к его атрибутам."""
    spec = importlib.util.find_spec(name)
    loader = importlib.util.LazyLoader(spec.loader)
    spec.loader = loader
    module = importlib.util.module_from_spec(spec)
    sys.modules[name] = module
    loader.exec_module(module)
    return module

# np = lazy_import("numpy")   # быстрый старт, загрузка отложена
```

### importlib.metadata — метаданные пакетов

```python
from importlib import metadata

# Версия установленного дистрибутива
print(metadata.version("pip"))

# Полные метаданные (PKG-INFO)
meta = metadata.metadata("pip")
print(meta["Name"], meta["Summary"])

# Зависимости
print(metadata.requires("requests"))      # список requirement-строк

# Точки входа (плагины, console_scripts)
eps = metadata.entry_points()
# В 3.10+ удобный select по группе:
for ep in eps.select(group="console_scripts"):
    print(ep.name, "->", ep.value)

# Список всех установленных дистрибутивов
for dist in metadata.distributions():
    print(dist.metadata["Name"], dist.version)
```

### importlib.resources — доступ к файлам внутри пакетов

```python
from importlib import resources

# Прочитать файл-ресурс, лежащий внутри пакета (работает даже из zip/egg)
# data = resources.files("mypackage").joinpath("data/config.json").read_text()
```

### Кастомный загрузчик и искатель (продвинутое)

```python
import importlib.abc
import importlib.util
import sys

class StringLoader(importlib.abc.Loader):
    """Загружает модуль из строки исходного кода."""
    def __init__(self, source):
        self.source = source

    def create_module(self, spec):
        return None                         # использовать стандартное создание

    def exec_module(self, module):
        exec(compile(self.source, module.__name__, "exec"), module.__dict__)

class StringFinder(importlib.abc.MetaPathFinder):
    """Отдаёт модули из заранее заданного словаря {имя: исходник}."""
    def __init__(self, sources):
        self.sources = sources

    def find_spec(self, fullname, path, target=None):
        if fullname in self.sources:
            return importlib.util.spec_from_loader(
                fullname, StringLoader(self.sources[fullname])
            )
        return None

# Регистрируем искатель в начало meta_path
sys.meta_path.insert(0, StringFinder({"virtual_mod": "VALUE = 42\n"}))

import virtual_mod                          # импортируется из строки!
print(virtual_mod.VALUE)                    # 42
```

### Циклические импорты — разбор

Циклический импорт — когда модуль A импортирует B, а B импортирует A (прямо или через цепочку). Поскольку модуль кладётся в `sys.modules` **до** полного исполнения, второй импорт получит **частично инициализированный** модуль.

```python
# --- module_a.py ---
import module_b                # B начнёт импортироваться, дойдёт до import module_a

def func_a():
    return module_b.func_b()

VALUE_A = "A"

# --- module_b.py ---
import module_a                # module_a уже в sys.modules, но НЕ доисполнен!

def func_b():
    return module_a.VALUE_A    # ОК: используется отложенно, внутри функции

# print(module_a.VALUE_A)      # БЫЛО БЫ ОШИБКОЙ на уровне модуля:
#                              # VALUE_A ещё не определён в момент импорта
```

Опасный случай — `from`-импорт на уровне модуля:

```python
# module_b.py
from module_a import VALUE_A   # ImportError: cannot import name 'VALUE_A'
#                              # (A ещё не дошёл до строки VALUE_A = "A")
```

**Способы решения циклических импортов:**

```python
# 1. Отложить импорт внутрь функции (lazy import) — самый частый приём
def func_b():
    import module_a            # импорт случится при вызове, когда A уже готов
    return module_a.VALUE_A

# 2. Импортировать модуль целиком (import module_a), а не имена из него,
#    и обращаться через module_a.X — связывание отложенное.

# 3. Реорганизовать код: вынести общую часть в третий модуль (common.py),
#    разорвав цикл.

# 4. Для аннотаций типов использовать TYPE_CHECKING:
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from module_a import SomeClass   # импорт только для type checker, не в рантайме

def handler(x: "SomeClass") -> None: ...
```

## Частые вопросы на собеседовании

**Q1. Что такое `sys.modules` и какова его роль?**
A: Кэш импортированных модулей (`имя → объект модуля`). При импорте Python сначала проверяет `sys.modules`; если модуль там — возвращает его без повторной загрузки. Это объясняет, почему модуль исполняется один раз, и лежит в основе механизма циклических импортов и `reload`.

**Q2. Из чего состоит механизм импорта?**
A: Из искателей (finders в `sys.meta_path`), которые находят модуль и возвращают `ModuleSpec`, и загрузчиков (loaders), которые создают и исполняют объект модуля. Этапы: проверка кэша → поиск spec → создание модуля → запись в `sys.modules` → исполнение тела → привязка имени.

**Q3. Чем `import_module` лучше `__import__`?**
A: `importlib.import_module` — рекомендованный высокоуровневый API: корректно обрабатывает вложенные имена (`a.b.c` возвращает `c`, а не `a`), поддерживает относительные импорты через `package`. `__import__` — низкоуровневая функция за оператором `import`, неудобна напрямую.

**Q4. Что происходит при циклическом импорте и как его решить?**
A: Так как модуль попадает в `sys.modules` до полного исполнения, при цикле второй импорт получает частично инициализированный модуль. Если на уровне модуля обратиться к ещё не определённому имени (особенно через `from x import y`) — будет `ImportError`/`AttributeError`. Решения: отложенный импорт внутри функции, импорт модуля целиком вместо `from`, рефакторинг общей логики в отдельный модуль, `TYPE_CHECKING` для аннотаций.

**Q5. Почему `from module import name` особенно проблематичен в циклах?**
A: `from`-импорт требует, чтобы имя уже существовало в момент импорта. При цикле целевой модуль может быть не доисполнен, и имени ещё нет — мгновенный `ImportError`. `import module` лишь связывает имя модуля (отложенный доступ к атрибутам), поэтому безопаснее.

**Q6. Как работает `reload` и в чём его ограничения?**
A: `importlib.reload(mod)` переисполняет код в том же объекте модуля. Ссылки `import mod` увидят изменения, но имена, импортированные через `from mod import x`, останутся старыми; существующие экземпляры сохранят старые классы. Для «чистой» перезагрузки иногда удаляют записи из `sys.modules`.

**Q7. Как импортировать модуль из произвольного файла по пути?**
A: Через `importlib.util.spec_from_file_location` + `module_from_spec` + `spec.loader.exec_module`. Желательно поместить модуль в `sys.modules` до `exec_module` (важно для корректных внутренних импортов и циклов).

**Q8. Зачем нужен `importlib.metadata`?**
A: Для доступа к метаданным установленных дистрибутивов без их импорта: версия (`version`), полные метаданные, зависимости (`requires`), точки входа (`entry_points` — основа системы плагинов и `console_scripts`). Заменяет устаревший `pkg_resources`.

**Q9. Что такое `sys.meta_path` и `sys.path`?**
A: `sys.meta_path` — список искателей, которые умеют находить модули (builtin, frozen, path-based, ваши кастомные). `sys.path` — список каталогов/архивов для поиска модулей файловым искателем. Чтобы добавить нестандартный источник импорта, регистрируют свой finder в `sys.meta_path`.

**Q10. Как сделать собственный загрузчик модулей?**
A: Реализовать `MetaPathFinder.find_spec`, возвращающий `ModuleSpec` с кастомным `Loader`, у которого определён `exec_module` (и при необходимости `create_module`). Затем зарегистрировать finder в `sys.meta_path`. Так делают импорт из БД, зашифрованных файлов, по сети.

**Q11. Где кэшируется скомпилированный байткод и при чём тут импорт?**
A: В `.pyc` файлах в каталоге `__pycache__` рядом с модулем. При импорте Python сверяет временную метку/хеш исходника и переиспользует `.pyc`, если он актуален, ускоряя загрузку.

**Q12. Можно ли импортировать модуль один раз, а потом «забыть» его?**
A: Удалить из `sys.modules` (`del sys.modules["name"]`) — следующий импорт перечитает файл. Но существующие ссылки на старый объект продолжат жить; это источник трудноуловимых багов при «горячей» перезагрузке.

## Подводные камни (gotchas)

- **`from x import y` в циклах падает**, тогда как `import x` чаще выживает — выбирайте форму осознанно.
- **`reload` не обновляет `from`-импорты и существующие экземпляры** — частая причина «почему мои изменения не подхватились».
- **Модуль кладётся в `sys.modules` до исполнения тела** — при цикле увидите неполный модуль; обращение к ещё не определённым именам падает.
- **Изменение `sys.path`/`sys.modules` глобально** влияет на весь процесс — побочные эффекты в тестах и плагинах.
- **`importlib.import_module("a.b.c")` возвращает `c`** (а не `a`), в отличие от `__import__`. Частая путаница.
- **Имя дистрибутива ≠ имя пакета для импорта.** `metadata.version("Pillow")`, но `import PIL`. Не путайте имя в PyPI и имя модуля.
- **`spec_from_file_location` без записи в `sys.modules`** ломает внутренние относительные импорты и циклы загружаемого модуля.
- **Устаревший `pkg_resources`** медленный — для метаданных используйте `importlib.metadata`.

## Лучшие практики

- Для динамического импорта — `importlib.import_module`, а не `__import__` и не `eval`.
- Разрывайте циклы через отложенный импорт внутри функций, импорт модуля целиком и `TYPE_CHECKING` для аннотаций; в идеале — рефакторингом архитектуры.
- При импорте из файла кладите модуль в `sys.modules` перед `exec_module`.
- Для метаданных и плагинов используйте `importlib.metadata` (`entry_points`), для файлов внутри пакетов — `importlib.resources` (а не `__file__` + пути).
- `reload` применяйте осторожно и осознанно (dev/hot-reload), помня про `from`-импорты и старые экземпляры.
- Кастомные finders/loaders документируйте и регистрируйте аккуратно, помня о глобальности `sys.meta_path`.

## Шпаргалка

```python
import importlib
import importlib.util
from importlib import metadata
import sys

# Динамический импорт
mod = importlib.import_module("pkg.sub")          # возвращает pkg.sub
importlib.import_module(".rel", package="pkg")    # относительный

# Перезагрузка
importlib.reload(mod)                              # тот же объект, переисполнение

# Импорт из файла
spec = importlib.util.spec_from_file_location("m", "/path/m.py")
m = importlib.util.module_from_spec(spec)
sys.modules["m"] = m                               # ДО exec_module
spec.loader.exec_module(m)

# Проверка наличия модуля
importlib.util.find_spec("numpy") is not None

# Метаданные пакетов (заменяет pkg_resources)
metadata.version("requests")
metadata.requires("requests")
metadata.entry_points().select(group="console_scripts")

# Кэш и пути
sys.modules        # имя -> модуль (кэш импортов)
sys.path           # каталоги поиска
sys.meta_path      # искатели (finders), сюда вставляют кастомные

# Циклический импорт: лечится отложенным импортом в функции,
# `import module` вместо `from module import name`, TYPE_CHECKING для аннотаций,
# выносом общей логики в отдельный модуль.
```
