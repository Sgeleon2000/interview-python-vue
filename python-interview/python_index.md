# Подготовка к собеседованию по Python — оглавление

Учебные материалы по разделам стандартной библиотеки Python
([docs.python.org/3/library](https://docs.python.org/3/library/index.html)).
Каждый файл построен по единой структуре: *что это и зачем → ключевые концепции →
основные функции/классы с примерами → частые вопросы на собеседовании (Q/A) →
подводные камни (gotchas) → лучшие практики → шпаргалка*.

> Уровень материалов — middle/senior. Все примеры рабочие, с комментариями на русском.

## Встроенное ядро языка

| Тема | Файл |
|------|------|
| Встроенные функции (`print`, `len`, `map`, `sorted`, `super`, `property`…) | [python_builtin_functions.md](python_builtin_functions.md) |
| Встроенные константы (`True`, `False`, `None`, `Ellipsis`, `__debug__`) | [python_builtin_constants.md](python_builtin_constants.md) |
| Встроенные типы (числа, `list`/`tuple`/`dict`/`set`, `str`, `bytes`) | [python_stdtypes.md](python_stdtypes.md) |
| Иерархия исключений, `raise from`, `ExceptionGroup` | [python_exceptions.md](python_exceptions.md) |

## Обработка текста

| Тема | Файл |
|------|------|
| Модуль `string`, `Template`, format spec | [python_string.md](python_string.md) |
| Регулярные выражения `re` | [python_re.md](python_re.md) |
| Перенос/форматирование текста `textwrap` | [python_textwrap.md](python_textwrap.md) |
| Сравнение последовательностей `difflib` | [python_difflib.md](python_difflib.md) |
| Юникод и нормализация `unicodedata` | [python_unicodedata.md](python_unicodedata.md) |

## Типы данных и структуры

| Тема | Файл |
|------|------|
| Дата и время `datetime`/`zoneinfo` | [python_datetime.md](python_datetime.md) |
| `collections` (`deque`, `Counter`, `defaultdict`…) | [python_collections.md](python_collections.md) |
| Абстрактные базовые классы `collections.abc` | [python_collections_abc.md](python_collections_abc.md) |
| Куча/приоритетная очередь `heapq` | [python_heapq.md](python_heapq.md) |
| Бинарный поиск `bisect` | [python_bisect.md](python_bisect.md) |
| Типизированные массивы `array` | [python_array.md](python_array.md) |
| Перечисления `enum` | [python_enum.md](python_enum.md) |
| Копирование объектов `copy` | [python_copy.md](python_copy.md) |
| Слабые ссылки `weakref` | [python_weakref.md](python_weakref.md) |

## Числа и математика

| Тема | Файл |
|------|------|
| `math` | [python_math.md](python_math.md) |
| Точные десятичные `decimal` | [python_decimal.md](python_decimal.md) |
| Рациональные `fractions` | [python_fractions.md](python_fractions.md) |
| Случайные числа `random` | [python_random.md](python_random.md) |
| Статистика `statistics` | [python_statistics.md](python_statistics.md) |
| Числовая башня `numbers` | [python_numbers.md](python_numbers.md) |

## Функциональное программирование

| Тема | Файл |
|------|------|
| Итераторы `itertools` | [python_itertools.md](python_itertools.md) |
| Декораторы/мемоизация `functools` | [python_functools.md](python_functools.md) |
| Операторы как функции `operator` | [python_operator.md](python_operator.md) |

## Файлы и каталоги

| Тема | Файл |
|------|------|
| `pathlib` | [python_pathlib.md](python_pathlib.md) |
| `os.path` | [python_os_path.md](python_os_path.md) |
| `shutil` | [python_shutil.md](python_shutil.md) |
| `tempfile` | [python_tempfile.md](python_tempfile.md) |
| `glob`/`fnmatch` | [python_glob.md](python_glob.md) |
| Потоки ввода-вывода `io` | [python_io.md](python_io.md) |

## Сохранение данных

| Тема | Файл |
|------|------|
| Сериализация `pickle` | [python_pickle.md](python_pickle.md) |
| `json` | [python_json.md](python_json.md) |
| `sqlite3` | [python_sqlite3.md](python_sqlite3.md) |
| `shelve` | [python_shelve.md](python_shelve.md) |
| `csv` | [python_csv.md](python_csv.md) |
| `configparser` | [python_configparser.md](python_configparser.md) |

## Сжатие и архивы

| Тема | Файл |
|------|------|
| `zipfile`/`tarfile` | [python_zipfile.md](python_zipfile.md) |
| `gzip`/`bz2`/`lzma`/`zlib` | [python_gzip.md](python_gzip.md) |

## Криптография

| Тема | Файл |
|------|------|
| Хеширование `hashlib` | [python_hashlib.md](python_hashlib.md) |
| `hmac` | [python_hmac.md](python_hmac.md) |
| `secrets` | [python_secrets.md](python_secrets.md) |

## Сервисы ОС

| Тема | Файл |
|------|------|
| `os` | [python_os.md](python_os.md) |
| `sys` | [python_sys.md](python_sys.md) |
| `time` | [python_time.md](python_time.md) |
| `argparse` | [python_argparse.md](python_argparse.md) |
| `logging` | [python_logging.md](python_logging.md) |
| `platform` | [python_platform.md](python_platform.md) |

## Параллелизм и многозадачность

| Тема | Файл |
|------|------|
| Потоки `threading` (+ GIL) | [python_threading.md](python_threading.md) |
| Процессы `multiprocessing` | [python_multiprocessing.md](python_multiprocessing.md) |
| Пулы `concurrent.futures` | [python_concurrent_futures.md](python_concurrent_futures.md) |
| Внешние процессы `subprocess` | [python_subprocess.md](python_subprocess.md) |
| Очереди `queue` | [python_queue.md](python_queue.md) |
| Контекстные переменные `contextvars` | [python_contextvars.md](python_contextvars.md) |

## Сеть и асинхронность

| Тема | Файл |
|------|------|
| `asyncio` | [python_asyncio.md](python_asyncio.md) |
| Сокеты `socket` | [python_socket.md](python_socket.md) |
| `ssl`/TLS | [python_ssl.md](python_ssl.md) |
| Мультиплексирование `selectors` | [python_selectors.md](python_selectors.md) |
| Сигналы `signal` | [python_signal.md](python_signal.md) |

## Интернет-данные и протоколы

| Тема | Файл |
|------|------|
| `email` | [python_email.md](python_email.md) |
| `urllib` | [python_urllib.md](python_urllib.md) |
| `http` (клиент/сервер) | [python_http.md](python_http.md) |
| `base64`/`binascii` | [python_base64.md](python_base64.md) |
| `uuid` | [python_uuid.md](python_uuid.md) |
| `ipaddress` | [python_ipaddress.md](python_ipaddress.md) |

## Сервисы времени выполнения

| Тема | Файл |
|------|------|
| `dataclasses` | [python_dataclasses.md](python_dataclasses.md) |
| `contextlib` (менеджеры контекста) | [python_contextlib.md](python_contextlib.md) |
| Абстрактные классы `abc` | [python_abc.md](python_abc.md) |
| Аннотации типов `typing` | [python_typing.md](python_typing.md) |
| `warnings` | [python_warnings.md](python_warnings.md) |
| Сборка мусора `gc` | [python_gc.md](python_gc.md) |
| Интроспекция `inspect` | [python_inspect.md](python_inspect.md) |
| `traceback` | [python_traceback.md](python_traceback.md) |

## Инструменты разработки и отладки

| Тема | Файл |
|------|------|
| `unittest` | [python_unittest.md](python_unittest.md) |
| `unittest.mock` | [python_unittest_mock.md](python_unittest_mock.md) |
| `doctest` | [python_doctest.md](python_doctest.md) |
| Отладчик `pdb` | [python_pdb.md](python_pdb.md) |
| `timeit` | [python_timeit.md](python_timeit.md) |
| Профилирование `cProfile` | [python_cProfile.md](python_cProfile.md) |
| Память `tracemalloc` | [python_tracemalloc.md](python_tracemalloc.md) |

## Языковые сервисы и импорт

| Тема | Файл |
|------|------|
| AST `ast` | [python_ast.md](python_ast.md) |
| Байт-код `dis` | [python_dis.md](python_dis.md) |
| Импорт модулей `importlib` | [python_importlib.md](python_importlib.md) |
| Токенизация `tokenize` | [python_tokenize.md](python_tokenize.md) |
| Виртуальные окружения `venv` | [python_venv.md](python_venv.md) |

---

**Всего: 87 тем.** Рекомендуемый порядок для собеседования: сначала ядро языка
(`stdtypes`, `exceptions`, `builtin_functions`), затем `collections`/`itertools`/`functools`,
`dataclasses`/`typing`, параллелизм (`threading` + GIL, `asyncio`), `gc` и тестирование.
