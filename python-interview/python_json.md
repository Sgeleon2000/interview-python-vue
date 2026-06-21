# json — подготовка к собеседованию

## Что это и зачем

`json` — стандартный модуль для работы с форматом **JSON** (JavaScript Object Notation): сериализация Python-объектов в JSON-строку и обратно. JSON — это текстовый, человекочитаемый, кросс-языковой формат обмена данными (RFC 8259). Это де-факто стандарт для REST API, конфигов, логов и обмена между сервисами.

Модуль реализует кодирование/декодирование между типами Python и JSON:

| Python | JSON |
|---|---|
| `dict` | object |
| `list`, `tuple` | array |
| `str` | string |
| `int`, `float` | number |
| `True` / `False` | `true` / `false` |
| `None` | `null` |

Обратите внимание: `tuple` сериализуется в массив, но при десериализации обратно станет `list`. `set`, `datetime`, `Decimal`, `bytes` **не** поддерживаются «из коробки» — нужны кастомные энкодеры.

## Ключевые концепции

- **Сериализация (encode/dump)** — Python → JSON-текст.
- **Десериализация (decode/load)** — JSON-текст → Python.
- **`default`** — функция-callback для объектов, которые стандартный энкодер не умеет сериализовать.
- **`JSONEncoder`** — класс энкодера; переопределяют метод `default()` для кастомных типов.
- **`object_hook` / `object_pairs_hook`** — callback при декодировании каждого JSON-объекта; превращает dict в нужный тип.
- **`ensure_ascii`** — экранировать ли не-ASCII символы в `\uXXXX`.
- **`indent` / `separators` / `sort_keys`** — управление форматированием вывода.
- **`cls`** — кастомный класс энкодера/декодера.

## Основные функции/классы/методы

### dump / dumps / load / loads

```python
import json

data = {"name": "Андрей", "age": 30, "active": True, "tags": ["a", "b"]}

# dumps -> str (сериализация в строку)
s: str = json.dumps(data)
print(s)  # {"name": "Андрей", ...}  (ensure_ascii=True по умолчанию!)

# loads -> объект (десериализация из строки/bytes)
obj = json.loads(s)
print(obj == data)  # True

# dump -> запись в файл (текстовый режим, в отличие от pickle!)
with open("data.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)

# load -> чтение из файла
with open("data.json", "r", encoding="utf-8") as f:
    obj2 = json.load(f)
```

Важно: в отличие от pickle, файлы для json открываются в **текстовом** режиме (`'w'`/`'r'`), а не бинарном. `loads` умеет принимать и `bytes`/`bytearray` (с автоопределением кодировки UTF-8/16/32).

### Форматирование: indent, separators, sort_keys

```python
import json

data = {"b": 2, "a": 1, "list": [1, 2, 3]}

# Компактный вывод (убираем лишние пробелы)
print(json.dumps(data, separators=(",", ":")))
# {"b":2,"a":1,"list":[1,2,3]}

# Красивый вывод с отступами и сортировкой ключей
print(json.dumps(data, indent=2, sort_keys=True))
# {
#   "a": 1,
#   "b": 2,
#   "list": [
#     1,
#     2,
#     3
#   ]
# }
```

`separators=(item_sep, key_sep)`. По умолчанию `(', ', ': ')`, но при заданном `indent` — `(',', ': ')`. Для максимально компактного JSON: `(",", ":")`.

### ensure_ascii

```python
import json

data = {"city": "Москва"}

# По умолчанию ensure_ascii=True -> не-ASCII экранируются
print(json.dumps(data))                     # {"city": "Москва"}

# ensure_ascii=False -> человекочитаемый UTF-8
print(json.dumps(data, ensure_ascii=False)) # {"city": "Москва"}
```

`ensure_ascii=True` (дефолт) делает вывод безопасным для любых ASCII-каналов, но раздувает размер и ухудшает читаемость для кириллицы/эмодзи. Для UTF-8 API почти всегда ставят `ensure_ascii=False` (и не забывают `encoding="utf-8"` при записи в файл).

### Кастомный энкодер через `default`

`default(obj)` вызывается для объектов, которые энкодер не знает; должен вернуть сериализуемый объект или поднять `TypeError`.

```python
import json
from datetime import datetime, date
from decimal import Decimal

def custom_default(obj):
    if isinstance(obj, (datetime, date)):
        return obj.isoformat()           # datetime -> строка ISO 8601
    if isinstance(obj, Decimal):
        return float(obj)                # Decimal -> float (с потерей точности!)
    if isinstance(obj, set):
        return list(obj)                 # set -> list
    if isinstance(obj, bytes):
        return obj.decode("utf-8")
    raise TypeError(f"Тип {type(obj)} не сериализуется в JSON")

data = {"ts": datetime(2026, 6, 20, 12, 0), "price": Decimal("9.99"), "tags": {"x", "y"}}
print(json.dumps(data, default=custom_default, ensure_ascii=False))
# {"ts": "2026-06-20T12:00:00", "price": 9.99, "tags": ["x", "y"]}
```

### Кастомный энкодер через подкласс `JSONEncoder`

Класс удобнее для переиспользования и передачи через `cls=`.

```python
import json
from datetime import datetime
from decimal import Decimal

class ExtendedEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return {"__datetime__": obj.isoformat()}  # помечаем тип для round-trip
        if isinstance(obj, Decimal):
            return {"__decimal__": str(obj)}          # str сохраняет точность!
        if isinstance(obj, set):
            return list(obj)
        return super().default(obj)  # делегируем стандартной обработке -> TypeError

data = {"ts": datetime(2026, 6, 20), "price": Decimal("9.99")}
print(json.dumps(data, cls=ExtendedEncoder))
# {"ts": {"__datetime__": "2026-06-20T00:00:00"}, "price": {"__decimal__": "9.99"}}
```

### Кастомная десериализация: object_hook

`object_hook(d)` вызывается для каждого декодированного JSON-объекта (dict); возвращаемое значение подставляется вместо dict. Позволяет восстанавливать типы.

```python
import json
from datetime import datetime
from decimal import Decimal

def custom_object_hook(d):
    if "__datetime__" in d:
        return datetime.fromisoformat(d["__datetime__"])
    if "__decimal__" in d:
        return Decimal(d["__decimal__"])
    return d

s = '{"ts": {"__datetime__": "2026-06-20T00:00:00"}, "price": {"__decimal__": "9.99"}}'
obj = json.loads(s, object_hook=custom_object_hook)
print(obj["ts"], type(obj["ts"]))     # 2026-06-20 00:00:00 <class 'datetime.datetime'>
print(obj["price"], type(obj["price"]))  # 9.99 <class 'decimal.Decimal'>
```

### object_pairs_hook — сохранение порядка/дубликатов ключей

`object_pairs_hook(pairs)` получает **список** пар `(ключ, значение)` ДО создания dict. Важно, когда нужно: обнаружить дублирующиеся ключи или использовать особый тип контейнера.

```python
import json
from collections import OrderedDict

# Обнаружение дублирующихся ключей (обычный loads молча берёт последний!)
def reject_duplicates(pairs):
    seen = {}
    for key, value in pairs:
        if key in seen:
            raise ValueError(f"Дублирующийся ключ: {key!r}")
        seen[key] = value
    return seen

print(json.loads('{"a": 1, "b": 2}', object_pairs_hook=reject_duplicates))  # {'a': 1, 'b': 2}
try:
    json.loads('{"a": 1, "a": 2}', object_pairs_hook=reject_duplicates)
except ValueError as e:
    print(e)  # Дублирующийся ключ: 'a'
```

### parse_float / parse_int — управление числами

Полезно для финансов: парсить числа как `Decimal`, чтобы не терять точность.

```python
import json
from decimal import Decimal

s = '{"price": 0.1, "qty": 3}'
obj = json.loads(s, parse_float=Decimal)
print(obj["price"], type(obj["price"]))  # 0.1 <class 'decimal.Decimal'>  (точно!)
```

## Частые вопросы на собеседовании

**Q1. Чем `dumps` отличается от `dump` (и `loads` от `load`)?**
`dumps`/`loads` работают со строкой (`str`/`bytes`) в памяти; `dump`/`load` — с файловым объектом. Файлы открываются в текстовом режиме (в отличие от pickle).

**Q2. Какие Python-типы НЕ сериализуются стандартным json и как с этим бороться?**
`set`, `tuple` (точнее, теряет тип → list), `datetime`/`date`, `Decimal`, `bytes`, экземпляры классов, `complex`. Решение: `default`-функция или подкласс `JSONEncoder` для кодирования, `object_hook`/`object_pairs_hook` для восстановления.

**Q3. Как сериализовать `datetime`?**
JSON не имеет типа даты. Конвертируют в строку ISO 8601 через `obj.isoformat()` в `default`, а при чтении парсят `datetime.fromisoformat(...)` в `object_hook`. Для round-trip помечают значение спец-ключом (`__datetime__`).

**Q4. Как корректно работать с `Decimal`?**
Стандартный json не умеет `Decimal`. Сериализовать через `str(obj)` (сохраняет точность) или `float(obj)` (теряет точность). При чтении — `parse_float=Decimal`, чтобы все числа с точкой парсились как Decimal без потерь.

**Q5. Что делает `ensure_ascii` и зачем его отключать?**
При `True` (дефолт) не-ASCII символы экранируются в `\uXXXX`. Отключают (`ensure_ascii=False`) для читаемого UTF-8 вывода кириллицы/эмодзи и меньшего размера. При записи в файл также указывают `encoding="utf-8"`.

**Q6. В чём разница `object_hook` и `object_pairs_hook`?**
`object_hook` получает уже готовый `dict`; `object_pairs_hook` получает список пар ДО создания dict (имеет приоритет, если заданы оба). `object_pairs_hook` нужен для контроля порядка и обнаружения дубликатов ключей.

**Q7. Что происходит с дублирующимися ключами в JSON-объекте?**
Стандартный `loads` молча оставляет **последнее** значение. Чтобы поймать дубликаты — используйте `object_pairs_hook` с проверкой.

**Q8. json vs pickle — когда что?**
json — для обмена с внешним миром (API, конфиги, межъязыковой обмен), безопасен, читаем, но только базовые типы. pickle — для внутреннего Python: сериализует почти всё, быстрее на сложных объектах, но небезопасен для чужих данных и непереносим. См. таблицу.

**Q9. Безопасен ли `json.loads`?**
В плане выполнения кода — да (в отличие от pickle, код не исполняется). Но есть риски DoS: глубокая вложенность (стек), огромные числа, гигантские строки. Для недоверенного входа ограничивайте размер и используйте потоковый/строгий парсер.

**Q10. Как получить компактный JSON без лишних пробелов?**
`json.dumps(data, separators=(",", ":"))` — убирает пробелы после запятых и двоеточий, экономит размер для сети.

**Q11. Сохраняется ли порядок ключей?**
Да, начиная с Python 3.7 dict упорядочен, и `json` сохраняет порядок вставки. Для сортировки — `sort_keys=True`.

**Q12. Что вернёт `json.dumps((1, 2))` и `json.loads('[1, 2]')`?**
Кортеж сериализуется в `"[1, 2]"`, но при загрузке вернётся **список** `[1, 2]` — информация о том, что это был tuple, теряется.

## Подводные камни (gotchas)

- **`tuple` → `list`.** Round-trip не сохраняет кортежи и множества.
- **Ключи объекта всегда строки.** Нечисловые/нестроковые ключи dict при `dumps` приводятся к строкам (`{1: "a"}` → `{"1": "a"}`); при чтении `1` станет `"1"`. Контролируйте через `sort_keys` и `parse_int`/object_hook при необходимости.
- **`float('nan')`, `Infinity`.** По умолчанию json **разрешает** `NaN`/`Infinity`/`-Infinity` (нестандартное расширение). Это невалидный JSON по RFC; многие парсеры других языков их не примут. Отключайте через `allow_nan=False` (поднимет `ValueError`).
- **Потеря точности float.** `0.1 + 0.2` и большие int как float теряют точность. Для денег — `Decimal` + `parse_float`.
- **`ensure_ascii` и файлы.** При `ensure_ascii=False` обязательно укажите `encoding="utf-8"` при открытии файла, иначе на Windows возможен `UnicodeEncodeError`.
- **`default` вызывается только для неизвестных типов.** Нельзя через `default` переопределить сериализацию `int`/`str` — он не будет вызван для известных типов.
- **`super().default()` обязателен** в подклассе `JSONEncoder` для неизвестных типов, иначе потеряете корректный `TypeError`.
- **Дубликаты ключей** молча схлопываются (последний выигрывает) — теряются данные.
- **DoS-вектор.** Огромная вложенность/числа из недоверенного источника. Ограничивайте размер входа.
- **`bytes` не сериализуется.** Нужно вручную (base64 или decode).

## Лучшие практики

1. Для UTF-8 API ставьте `ensure_ascii=False` и `encoding="utf-8"` при записи.
2. Для денег/точных чисел используйте `Decimal` + `parse_float=Decimal`, сериализуйте через `str()`.
3. Кастомную логику типов оформляйте подклассом `JSONEncoder` (переиспользуемо) + парный `object_hook`.
4. Для round-trip нестандартных типов помечайте их спец-ключами (`__datetime__`, `__decimal__`).
5. Для недоверенного входа: `allow_nan=False`, ограничение размера, проверка дубликатов ключей.
6. Для сети — компактные `separators=(",", ":")`; для логов/конфигов — `indent=2`.
7. Используйте json (а не pickle) для любого внешнего обмена данными — он безопасен и кросс-язычен.
8. Не кладите в JSON `tuple`/`set`, рассчитывая на round-trip.

## Шпаргалка

```python
import json

# --- Базовые операции ---
s = json.dumps(obj)                    # объект -> str
obj = json.loads(s)                    # str/bytes -> объект
json.dump(obj, file)                   # -> файл (текстовый режим!)
obj = json.load(file)                  # <- файл

# --- Форматирование ---
json.dumps(obj, indent=2, sort_keys=True)        # красиво
json.dumps(obj, separators=(",", ":"))           # компактно
json.dumps(obj, ensure_ascii=False)              # читаемый UTF-8
json.dumps(obj, allow_nan=False)                 # запретить NaN/Infinity

# --- Кастомная сериализация ---
json.dumps(obj, default=func)                    # func(o) для неизвестных типов
json.dumps(obj, cls=MyEncoder)                   # подкласс JSONEncoder.default()

# --- Кастомная десериализация ---
json.loads(s, object_hook=func)                  # dict -> кастомный тип
json.loads(s, object_pairs_hook=func)            # список пар (приоритет, дубликаты)
json.loads(s, parse_float=Decimal)               # числа -> Decimal без потерь
```

### json vs pickle — сводная таблица

| Критерий | json | pickle |
|---|---|---|
| Формат | текстовый | бинарный |
| Кросс-язык | да | нет |
| Читаемость | да | нет |
| Типы | базовые (dict/list/str/num/bool/null) | почти любые объекты |
| Безопасность загрузки | безопасно | **опасно (RCE)** |
| Режим файла | текстовый (`w`/`r`) | бинарный (`wb`/`rb`) |
| datetime/Decimal/set | нужны хуки | из коробки |
| Применение | API, конфиги, обмен | внутренний кэш, IPC |
```
