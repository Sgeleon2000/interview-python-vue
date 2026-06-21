# unittest — подготовка к собеседованию

## Что это и зачем

`unittest` — встроенный в стандартную библиотеку Python фреймворк для модульного тестирования (xUnit-семейство, вдохновлён JUnit). Он поставляется вместе с интерпретатором, поэтому не требует установки сторонних пакетов.

Зачем нужен:
- **Регрессия.** Тесты фиксируют ожидаемое поведение и ломаются, когда код меняется непреднамеренно.
- **Документация.** Тест показывает, как должен использоваться код.
- **Рефакторинг.** С тестами можно смело менять реализацию, проверяя, что контракт не нарушен.
- **CI/CD.** Тесты автоматически запускаются на сервере перед мержем.

Ключевые сущности: класс `TestCase` (контейнер для тестов), `TestSuite` (набор тестов), `TestLoader` (поиск и загрузка тестов), `TestRunner` (запуск и вывод результата), `TestResult` (накопитель результатов).

## Ключевые концепции

- **Тест-метод** — метод класса, имя которого начинается с `test`. Каждый такой метод выполняется изолированно.
- **Фикстуры (fixtures)** — подготовка окружения до теста (`setUp`) и очистка после (`tearDown`). На уровне класса — `setUpClass`/`tearDownClass`, на уровне модуля — `setUpModule`/`tearDownModule`.
- **Ассерты (assertions)** — методы вида `assertEqual`, `assertTrue`, проверяющие условие и фиксирующие падение с понятным сообщением.
- **Изоляция.** Для каждого тест-метода создаётся *новый* экземпляр `TestCase`, поэтому атрибуты, выставленные в одном тесте, не «протекают» в другой.
- **Test discovery** — автоматический поиск тестов по соглашениям об именах (`test*.py`).
- **Skip / expectedFailure** — управление пропуском тестов.
- **subTest** — несколько проверок в одном тесте без остановки на первой ошибке.

## Основные функции/классы/методы

### Минимальный пример

```python
import unittest


def add(a, b):
    return a + b


class TestAdd(unittest.TestCase):
    def test_positive(self):
        # Проверяем сложение положительных чисел
        self.assertEqual(add(2, 3), 5)

    def test_negative(self):
        self.assertEqual(add(-1, -1), -2)


if __name__ == "__main__":
    # Запуск всех тестов модуля при прямом вызове файла
    unittest.main()
```

Запуск:

```bash
python -m unittest test_module.py    # конкретный файл
python -m unittest                    # discovery в текущей папке
python -m unittest -v                 # подробный вывод
```

### Основные ассерты

```python
class TestAsserts(unittest.TestCase):
    def test_equality(self):
        self.assertEqual(1 + 1, 2)            # a == b
        self.assertNotEqual(1, 2)             # a != b

    def test_bool(self):
        self.assertTrue(bool([1]))            # bool(x) is True
        self.assertFalse(bool([]))            # bool(x) is False

    def test_identity(self):
        a = object()
        self.assertIs(a, a)                   # a is b
        self.assertIsNot(a, object())         # a is not b
        self.assertIsNone(None)               # x is None
        self.assertIsNotNone(0)               # x is not None

    def test_membership(self):
        self.assertIn(2, [1, 2, 3])           # a in b
        self.assertNotIn(5, [1, 2, 3])        # a not in b

    def test_types(self):
        self.assertIsInstance(1, int)         # isinstance(a, b)
        self.assertNotIsInstance("s", int)

    def test_compare(self):
        self.assertGreater(3, 2)              # a > b
        self.assertGreaterEqual(3, 3)
        self.assertLess(2, 3)
        self.assertLessEqual(2, 2)

    def test_approx(self):
        # Сравнение float с точностью; places — число знаков после запятой
        self.assertAlmostEqual(0.1 + 0.2, 0.3, places=7)
        self.assertNotAlmostEqual(0.1, 0.2)
        # Альтернатива через delta — допустимая абсолютная разница
        self.assertAlmostEqual(100, 102, delta=5)

    def test_collections(self):
        # Сравнение списков с учётом порядка
        self.assertListEqual([1, 2], [1, 2])
        # Сравнение без учёта порядка и с учётом повторов
        self.assertCountEqual([1, 2, 2], [2, 1, 2])
        self.assertDictEqual({"a": 1}, {"a": 1})
        self.assertSetEqual({1, 2}, {2, 1})
        # Проверка подмножества элементов словаря
        # (assertDictContainsSubset устарел; используйте subset-проверку вручную)

    def test_strings(self):
        self.assertRegex("abc123", r"\d+")            # совпадение по regex
        self.assertNotRegex("abc", r"\d+")
        # Удобная разница строк при падении
        self.assertMultiLineEqual("line1\nline2", "line1\nline2")
```

### Проверка исключений и предупреждений

```python
class TestExceptions(unittest.TestCase):
    def test_raises_context(self):
        # Контекст-менеджер: тело должно бросить ZeroDivisionError
        with self.assertRaises(ZeroDivisionError):
            1 / 0

    def test_raises_message(self):
        # Проверяем тип И текст сообщения через regex
        with self.assertRaisesRegex(ValueError, r"invalid literal"):
            int("not a number")

    def test_raises_inspect(self):
        # Доступ к самому исключению через атрибут .exception
        with self.assertRaises(ValueError) as ctx:
            raise ValueError("boom")
        self.assertEqual(str(ctx.exception), "boom")

    def test_callable_form(self):
        # Альтернативная форма: assertRaises(Exc, callable, *args)
        self.assertRaises(KeyError, lambda: {}["missing"])

    def test_warns(self):
        import warnings
        with self.assertWarns(UserWarning):
            warnings.warn("deprecated", UserWarning)

    def test_logs(self):
        import logging
        # Проверяем, что в лог записано нужное сообщение
        with self.assertLogs("myapp", level="INFO") as cm:
            logging.getLogger("myapp").info("hello")
        self.assertIn("hello", cm.output[0])
```

### setUp / tearDown / setUpClass / tearDownClass

```python
class TestWithFixtures(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        # Выполняется ОДИН раз перед всеми тестами класса.
        # Подходит для дорогих ресурсов: коннект к БД, поднятие сервера.
        print("setUpClass")
        cls.shared_resource = expensive_connection()

    @classmethod
    def tearDownClass(cls):
        # ОДИН раз после всех тестов класса
        cls.shared_resource.close()
        print("tearDownClass")

    def setUp(self):
        # Перед КАЖДЫМ тест-методом. Здесь готовим изолированное состояние.
        self.data = [1, 2, 3]

    def tearDown(self):
        # После КАЖДОГО теста (даже если тест упал). Очистка.
        self.data = None

    def test_one(self):
        self.assertEqual(len(self.data), 3)

    def test_two(self):
        self.data.append(4)
        self.assertEqual(len(self.data), 4)
        # В test_one self.data снова будет [1, 2, 3] — изоляция!


def expensive_connection():
    class Conn:
        def close(self):
            pass
    return Conn()
```

Порядок выполнения для класса с двумя тестами:
```
setUpModule (если есть)
  setUpClass
    setUp -> test_one -> tearDown
    setUp -> test_two -> tearDown
  tearDownClass
tearDownModule (если есть)
```

### addCleanup — надёжная очистка

```python
class TestCleanup(unittest.TestCase):
    def test_resource(self):
        f = open("/tmp/data.txt", "w")
        # addCleanup гарантирует вызов даже если тест упадёт ниже.
        # Вызовы выполняются в обратном порядке (LIFO), как стек.
        self.addCleanup(f.close)
        f.write("data")
        self.assertTrue(True)
```

`addCleanup` надёжнее `tearDown`, потому что регистрируется только после успешного создания ресурса и не зависит от того, упал ли `setUp`.

### Пропуск тестов (skip)

```python
import sys


class TestSkips(unittest.TestCase):
    @unittest.skip("временно отключён, баг JIRA-123")
    def test_skipped_always(self):
        self.fail("этот код не выполнится")

    @unittest.skipIf(sys.platform == "win32", "не работает на Windows")
    def test_unix_only(self):
        self.assertTrue(True)

    @unittest.skipUnless(sys.version_info >= (3, 11), "нужен Python 3.11+")
    def test_new_feature(self):
        self.assertTrue(True)

    def test_conditional_skip(self):
        # Пропуск изнутри теста по условию, известному только в рантайме
        if not has_gpu():
            self.skipTest("нет GPU")
        self.assertTrue(True)

    @unittest.expectedFailure
    def test_known_bug(self):
        # Тест ОЖИДАЕМО падает. Если вдруг пройдёт — будет "unexpected success".
        self.assertEqual(1, 2)


def has_gpu():
    return False
```

### subTest — параметризация без остановки

```python
class TestSubTests(unittest.TestCase):
    def test_even_numbers(self):
        for i in range(0, 6):
            # Каждая итерация — отдельный "субтест".
            # Если упадёт i=3, проверки i=4,5 ВСЁ РАВНО выполнятся,
            # и в отчёте будут все упавшие значения, а не только первое.
            with self.subTest(i=i):
                self.assertEqual(i % 2, 0)
        # Без subTest цикл упал бы на первом нечётном и скрыл остальные.
```

### Test discovery

```bash
# Ищет файлы по шаблону test*.py начиная с текущей папки
python -m unittest discover

# С явными параметрами:
python -m unittest discover -s tests -p "*_test.py" -t .
#   -s стартовая директория
#   -p паттерн имён файлов
#   -t топ-уровень проекта (для импортов)
```

Соглашения для discovery: каталоги должны быть импортируемыми (наличие `__init__.py` в старых версиях; для namespace-пакетов не обязательно), имена файлов — по паттерну, классы наследуют `TestCase`, методы начинаются с `test`.

### Программный запуск (suite, runner)

```python
import unittest


def make_suite():
    suite = unittest.TestSuite()
    # Добавляем отдельные тесты
    suite.addTest(TestAdd("test_positive"))
    # Или загружаем все тесты класса
    loader = unittest.TestLoader()
    suite.addTests(loader.loadTestsFromTestCase(TestAsserts))
    return suite


if __name__ == "__main__":
    runner = unittest.TextTestRunner(verbosity=2)
    result = runner.run(make_suite())
    # result.wasSuccessful() -> bool, result.failures, result.errors
```

## Частые вопросы на собеседовании

**Q1. В чём разница между failure и error в unittest?**
*Failure* — провалена ассерция (`AssertionError`), то есть проверка не прошла, но тест отработал штатно. *Error* — в тесте возникло непредвиденное исключение (например, `KeyError`, `TypeError`), не связанное с ассертом. В отчёте они считаются раздельно: `FAILED (failures=1, errors=2)`.

**Q2. Создаётся ли один экземпляр TestCase на все тесты или на каждый?**
На *каждый* тест-метод создаётся новый экземпляр класса. Поэтому атрибуты `self.x`, выставленные в одном тесте, не видны в другом. Общее состояние нужно класть в атрибуты класса через `setUpClass` (с осторожностью — оно НЕ изолируется).

**Q3. Чем отличаются setUp/tearDown от setUpClass/tearDownClass?**
`setUp`/`tearDown` запускаются перед/после *каждого* тест-метода и обеспечивают изоляцию. `setUpClass`/`tearDownClass` — *один* раз на класс (методы класса, декорированы `@classmethod`), для дорогих общих ресурсов. Минус `setUpClass`: состояние общее, тесты могут влиять друг на друга.

**Q4. Что делает assertRaises и какие формы у него есть?**
Проверяет, что блок кода бросает указанное исключение. Две формы: контекст-менеджер `with self.assertRaises(Exc):` и вызываемая `self.assertRaises(Exc, func, *args)`. Через `as ctx` можно получить само исключение (`ctx.exception`). Для проверки текста — `assertRaisesRegex`.

**Q5. Зачем нужен subTest?**
Чтобы в одном тест-методе прогнать несколько наборов данных и при падении одного не прерывать остальные. Без него цикл с ассертом остановится на первой ошибке. `with self.subTest(param=value)` помечает каждую итерацию, и в отчёте видно все упавшие наборы с их параметрами.

**Q6. Чем skip отличается от expectedFailure?**
`skip`/`skipIf`/`skipUnless` тест *не запускают вообще* (помечается как skipped). `expectedFailure` — тест *запускается*, но его падение считается ожидаемым (expected failure). Если такой тест внезапно пройдёт — это «unexpected success», тоже сигнализирующий о проблеме.

**Q7. Как работает test discovery и какие требования к структуре?**
`python -m unittest discover` рекурсивно ищет файлы по паттерну (`test*.py` по умолчанию), импортирует их и собирает все `TestCase`. Требуется, чтобы каталоги были импортируемыми и имена файлов/методов соответствовали соглашениям. Параметры `-s`, `-p`, `-t` управляют стартовой папкой, паттерном и top-level директорией.

**Q8. В каком порядке выполняются тесты?**
По умолчанию — в *алфавитном* порядке имён методов внутри класса (через `TestLoader.sortTestMethodsUsing`). Полагаться на порядок — антипаттерн: тесты должны быть независимыми. Порядок классов и модулей определяется порядком загрузки.

**Q9. Что такое addCleanup и чем он лучше tearDown?**
`addCleanup(func, *args)` регистрирует функции очистки, выполняемые в обратном порядке (LIFO) после теста, даже при падении. Преимущество: регистрируется *после* успешного создания ресурса (в самом тесте), поэтому не пытается чистить то, что не создалось, и работает даже если `setUp` упал частично.

**Q10. Как сравнить два float в тестах?**
Не через `assertEqual` (из-за погрешности представления), а через `assertAlmostEqual(a, b, places=N)` или `assertAlmostEqual(a, b, delta=D)`. `places` — округление до N знаков, `delta` — допустимая абсолютная разница.

**Q11. Чем unittest отличается от pytest?**
unittest — встроенный, классовый, многословный (`self.assertEqual`). pytest — сторонний, использует обычный `assert` с интроспекцией, фикстуры через декораторы и dependency injection, мощную параметризацию `@pytest.mark.parametrize`, плагины. pytest умеет запускать тесты, написанные на unittest, поэтому миграция плавная.

**Q12. Можно ли запускать unittest-тесты через pytest?**
Да. pytest обнаруживает классы-наследники `TestCase` и их `test*` методы, корректно вызывает `setUp/tearDown/setUpClass`. Это позволяет постепенно мигрировать, оставляя старые тесты как есть.

**Q13. Как параметризовать тесты в чистом unittest?**
Несколькими способами: цикл с `subTest`; генерация методов в метаклассе/динамически; сторонние пакеты (`parameterized`). Нативной декоративной параметризации, как в pytest, в unittest нет — отсюда популярность `subTest`.

**Q14. Что вернёт unittest, если тест ничего не проверяет (нет ассертов)?**
Тест считается *прошедшим*. unittest не требует наличия ассертов; «зелёный» тест без проверок — частая скрытая проблема (тест ничего не валидирует). Линтеры и code review должны это отлавливать.

## Подводные камни (gotchas)

- **Состояние в setUpClass не изолируется.** Если тест мутирует `cls.shared`, следующий тест увидит изменение. Используйте `setUpClass` только для read-only/неизменяемых ресурсов или сбрасывайте состояние в `setUp`.
- **Зависимость от порядка тестов.** Тесты сортируются по алфавиту. Код, который «работает», только если `test_a` выполнился до `test_b`, — хрупкий. Каждый тест должен быть самодостаточным.
- **`assertTrue(a == b)` вместо `assertEqual(a, b)`.** Первый при падении выдаёт бесполезное `False is not true`, второй — показывает фактические значения. Всегда выбирайте специализированный ассерт.
- **Имя метода не с `test`.** Метод `check_something` НЕ будет найден как тест. Опечатка в префиксе — молчаливо «пропущенный» тест.
- **tearDown не вызовется, если упал setUp.** Если `setUp` бросил исключение, тело теста и `tearDown` не выполняются (но `addCleanup`, зарегистрированные ДО падения, выполнятся). Поэтому для критичной очистки предпочтительнее `addCleanup`.
- **Мутабельные атрибуты класса.** `class T(TestCase): data = []` — список общий для всех тестов и экземпляров. Инициализируйте мутабельное состояние в `setUp`.
- **assertRaises без указания, ГДЕ ждём исключение.** `self.assertRaises(ValueError, func())` — здесь `func()` вызывается *до* assertRaises и исключение не перехватывается. Нужно передавать `func` без скобок: `assertRaises(ValueError, func, arg)` или использовать `with`.
- **Долгий setUpClass на тяжёлых ресурсах.** Если в `setUpClass` падает соединение, упадёт весь класс целиком с error, а не отдельные тесты.

## Лучшие практики

- Один тест проверяет одно поведение; имя метода описывает сценарий: `test_withdraw_raises_when_insufficient_funds`.
- Используйте максимально специфичный ассерт (`assertEqual`, `assertIn`, `assertIsInstance`), а не `assertTrue` с выражением.
- Делайте тесты независимыми и идемпотентными — любой порядок и любое подмножество должны проходить.
- Дорогие ресурсы (БД, сеть) изолируйте через мок или поднимайте в `setUpClass`, лёгкое состояние — в `setUp`.
- Для очистки предпочитайте `addCleanup` вместо `tearDown`.
- Для табличных данных используйте `subTest`, чтобы видеть все падения сразу.
- Добавляйте поясняющее сообщение в ассерт при неочевидной проверке: `self.assertEqual(x, y, "баланс должен уменьшиться")`.
- Запускайте в CI с `-v` и собирайте отчёт; падающие тесты не должны мержиться.

## Шпаргалка

```python
import unittest

class T(unittest.TestCase):
    @classmethod
    def setUpClass(cls): ...        # 1 раз до всех
    @classmethod
    def tearDownClass(cls): ...     # 1 раз после всех
    def setUp(self): ...            # перед каждым тестом
    def tearDown(self): ...         # после каждого теста
    def test_x(self): ...           # метод начинается с test

# Ассерты:
assertEqual / assertNotEqual          # ==, !=
assertTrue / assertFalse              # истинность
assertIs / assertIsNot                # is, is not
assertIsNone / assertIsNotNone
assertIn / assertNotIn                # membership
assertIsInstance / assertNotIsInstance
assertGreater / assertLess / assertGreaterEqual / assertLessEqual
assertAlmostEqual(a, b, places=7)     # float
assertListEqual / assertDictEqual / assertSetEqual / assertCountEqual
assertRegex / assertRaises / assertRaisesRegex / assertWarns / assertLogs

# Пропуск:
@unittest.skip("причина")
@unittest.skipIf(cond, "причина")
@unittest.skipUnless(cond, "причина")
@unittest.expectedFailure
self.skipTest("причина")              # из тела теста

# subTest:
with self.subTest(param=v): ...

# Очистка:
self.addCleanup(func, *args)
```

```bash
python -m unittest                      # discovery
python -m unittest -v                   # подробно
python -m unittest module.Class.test    # точечно
python -m unittest discover -s tests -p "*_test.py"
python -m unittest -f                   # остановка на первой ошибке (failfast)
python -m unittest -k pattern           # фильтр по подстроке/glob имени
```
