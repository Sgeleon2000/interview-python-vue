# Модуль `unicodedata`, Unicode и кодировки — подготовка к собеседованию

## Что это и зачем

`unicodedata` — модуль стандартной библиотеки, дающий доступ к **базе данных Unicode Character Database (UCD)**: имена символов, категории, числовые значения, формы нормализации и т.д.

Тема в целом охватывает фундамент работы Python с текстом:

- разница между `str` (последовательность Unicode code points) и `bytes` (последовательность байтов);
- кодирование/декодирование (`encode`/`decode`) — мост между текстом и байтами;
- **нормализация** Unicode (NFC/NFD/NFKC/NFKD) — критична для сравнения, поиска, дедупликации;
- категории символов, обработка комбинирующих знаков, эмодзи, диакритики.

Понимание этого отличает middle/senior: «UnicodeDecodeError», «два визуально одинаковых имени не равны», корректная транслитерация и санитизация.

## Ключевые концепции

- **Code point** — числовой идентификатор символа (U+0041 = 'A'). `str` в Python 3 — последовательность code points.
- **`str` vs `bytes`** — `str` это абстрактный текст; `bytes` — конкретные байты в какой-то кодировке. Между ними переходят через `encode`/`decode`.
- **Кодировка (encoding)** — правило отображения code points в байты: UTF-8, UTF-16, ASCII, latin-1, cp1251 и т.д. UTF-8 — стандарт де-факто.
- **Нормализация** — приведение к канонической форме, потому что один и тот же символ может быть записан по-разному (precomposed 'é' vs 'e' + комбинирующий акут).
- **Категория** — класс символа (Lu — заглавная буква, Nd — десятичная цифра, Mn — комбинирующий знак и т.д.).
- **Канонические vs совместимостные** эквивалентности — NFC/NFD сохраняют визуальную идентичность; NFKC/NFKD дополнительно «упрощают» (ﬁ → fi, ① → 1).

## Основные функции/классы/методы

### Базовые функции `unicodedata`

```python
import unicodedata

# name() — официальное имя символа
print(unicodedata.name('A'))      # LATIN CAPITAL LETTER A
print(unicodedata.name('я'))      # CYRILLIC SMALL LETTER YA
print(unicodedata.name('€'))      # EURO SIGN
print(unicodedata.name('😀'))     # GRINNING FACE

# lookup() — символ по имени (обратная к name)
print(unicodedata.lookup('EURO SIGN'))           # €
print(unicodedata.lookup('GREEK SMALL LETTER PI')) # π

# category() — категория символа
print(unicodedata.category('A'))   # Lu (Letter, uppercase)
print(unicodedata.category('a'))   # Ll (Letter, lowercase)
print(unicodedata.category('5'))   # Nd (Number, decimal)
print(unicodedata.category(' '))   # Zs (Separator, space)
print(unicodedata.category('!'))   # Po (Punctuation, other)
print(unicodedata.category('́'))  # Mn (Mark, nonspacing) — комбинир. акут

# numeric() / digit() / decimal() — числовые значения
print(unicodedata.numeric('½'))    # 0.5
print(unicodedata.numeric('Ⅻ'))    # 12.0 (римская)
print(unicodedata.digit('7'))      # 7
print(unicodedata.decimal('٣'))    # 3 (арабская цифра)

# bidirectional, combining, mirrored — свойства
print(unicodedata.combining('́'))  # 230 (класс комбинирования)
print(unicodedata.bidirectional('A'))   # L (left-to-right)
print(unicodedata.unidata_version)       # версия базы Unicode
```

### Нормализация: NFC, NFD, NFKC, NFKD

```python
import unicodedata

# Два способа записать 'é':
s1 = 'café'                    # с precomposed é (U+00E9)
s2 = 'café'             # 'e' + комбинирующий акут (U+0301)

print(s1 == s2)               # False! Визуально одинаковы, но разные строки
print(len(s1), len(s2))       # 4 5

# NFC (Canonical Composition) — склеивает в precomposed
print(unicodedata.normalize('NFC', s1) == unicodedata.normalize('NFC', s2))
# True

# NFD (Canonical Decomposition) — раскладывает на базу + знаки
nfd = unicodedata.normalize('NFD', s1)
print(len(nfd))               # 5 (e + акут)

# NFKC / NFKD — Compatibility (более агрессивно «упрощают»)
print(unicodedata.normalize('NFKC', 'ﬁ'))   # fi  (лигатура -> 2 буквы)
print(unicodedata.normalize('NFKC', '①'))   # 1   (кружок -> цифра)
print(unicodedata.normalize('NFKC', '²'))   # 2   (надстрочная -> обычная)
print(unicodedata.normalize('NFKC', 'Ⅻ'))   # XII (римская -> буквы)
```

Сравнение форм:

| Форма | Что делает | Когда использовать |
|-------|------------|--------------------|
| NFC | Каноническая композиция (склеивает) | Хранение, сравнение, обмен (рекомендуется W3C) |
| NFD | Каноническое разложение | Удаление диакритики, посимвольный анализ |
| NFKC | Совместимостная композиция | Поиск, идентификаторы (агрессивная нормализация) |
| NFKD | Совместимостное разложение | Транслитерация, очистка от форматирования |

### Удаление диакритики (транслитерация)

```python
import unicodedata

def remove_accents(text: str) -> str:
    # 1. Разложить на базу + комбинирующие знаки (NFD)
    nfd = unicodedata.normalize('NFD', text)
    # 2. Убрать все комбинирующие знаки (category Mn)
    return ''.join(c for c in nfd if unicodedata.category(c) != 'Mn')

print(remove_accents('café résumé naïve'))  # cafe resume naive
print(remove_accents('Müller'))              # Muller
```

### `str` vs `bytes`, `encode` / `decode`

```python
# str — текст (Unicode code points)
text = "Привет, мир! €"
print(type(text), len(text))  # <class 'str'> 14

# encode: str -> bytes (нужна кодировка)
data = text.encode('utf-8')
print(type(data), len(data))  # <class 'bytes'> 25 (кириллица 2 байта, € 3 байта)
print(data)                   # b'\xd0\x9f\xd1\x80...'

# decode: bytes -> str
back = data.decode('utf-8')
print(back == text)           # True

# Разные кодировки -> разные байты
print("é".encode('utf-8'))    # b'\xc3\xa9'  (2 байта)
print("é".encode('latin-1'))  # b'\xe9'      (1 байт)
# print("€".encode('latin-1')) # UnicodeEncodeError: нет € в latin-1

# UTF-16 с BOM
print("A".encode('utf-16'))   # b'\xff\xfeA\x00' (BOM + 2 байта)
```

### Обработка ошибок кодирования/декодирования

```python
# Параметр errors управляет поведением при невозможности конвертации
text = "Привет €"

# strict (по умолчанию) — бросает исключение
# text.encode('ascii')  # UnicodeEncodeError

# ignore — пропускает проблемные символы
print(text.encode('ascii', errors='ignore'))   # b' '

# replace — заменяет на ? (encode) или U+FFFD (decode)
print(text.encode('ascii', errors='replace'))  # b'??????? ?'

# xmlcharrefreplace — XML-сущности (только encode)
print("€".encode('ascii', errors='xmlcharrefreplace'))  # b'&#8364;'

# backslashreplace — \u-экранирование
print("€".encode('ascii', errors='backslashreplace'))   # b'\\u20ac'

# Декодирование битых байтов
broken = b'\xff\xfeABC'
print(broken.decode('utf-8', errors='replace'))  # '��ABC'
# surrogateescape — обратимо «прячет» битые байты (для путей файлов)
print(broken.decode('utf-8', errors='surrogateescape'))
```

### Категории — практическое применение

```python
import unicodedata

def is_letter(c): return unicodedata.category(c).startswith('L')
def is_digit(c):  return unicodedata.category(c).startswith('N')

text = "abc123 πφ ٣ 😀"
letters = [c for c in text if is_letter(c)]
digits  = [c for c in text if is_digit(c)]
print(letters)  # ['a', 'b', 'c', 'π', 'φ', '٣' попадёт? нет — Nd]
print(digits)   # ['1', '2', '3', '٣']

# Удаление непечатаемых/управляющих символов (категория C*)
def clean(text):
    return ''.join(c for c in text if not unicodedata.category(c).startswith('C'))
print(repr(clean("текст\x00\x07с\tмусором")))  # 'текстс\tмусором'? \t это Cc!
```

### Безопасное сравнение строк (casefold + нормализация)

```python
import unicodedata

def normalize_for_comparison(s: str) -> str:
    # casefold агрессивнее lower (немецкая ß -> ss)
    return unicodedata.normalize('NFC', s.casefold())

print('ß'.casefold())        # ss
print('STRASSE'.casefold() == normalize_for_comparison('straße').upper().casefold())
# Сравнение имён пользователей независимо от формы Unicode:
print(normalize_for_comparison('café') == normalize_for_comparison('café'))
# True
```

## Частые вопросы на собеседовании

**Q1: В чём разница между `str` и `bytes` в Python 3?**
A: `str` — неизменяемая последовательность Unicode code points (абстрактный текст). `bytes` — неизменяемая последовательность байтов (0–255). Между ними нет неявного преобразования: `str.encode(encoding)` → `bytes`, `bytes.decode(encoding)` → `str`. Смешивать их в операциях (`"a" + b"b"`) — `TypeError`.

**Q2: Зачем нужна нормализация Unicode?**
A: Один и тот же символ может иметь разные представления (precomposed 'é' U+00E9 vs 'e'+U+0301). Без нормализации визуально идентичные строки не равны (`==` вернёт False), ломая сравнение, поиск, дедупликацию, ключи словарей. Нормализация приводит к канонической форме.

**Q3: Различие NFC/NFD/NFKC/NFKD?**
A: NFC — каноническая **композиция** (склеивает в precomposed). NFD — каноническое **разложение** (база + комбинирующие знаки). NFK* — **совместимостные** формы: дополнительно «упрощают» символы (лигатура ﬁ→fi, ①→1, ²→2), теряя форматирование. C=composed, D=decomposed, K=compatibility. Для хранения/обмена — NFC; для удаления диакритики — NFD; для поиска/идентификаторов — NFKC.

**Q4: Когда NFKC/NFKD опасны?**
A: Совместимостная нормализация необратима и меняет семантику: '²' станет '2', '½' разложится, лигатуры распадутся, а полноширинные символы сожмутся. Нельзя применять к данным, где важна точная форма (пароли, криптоключи), но полезно для нормализации идентификаторов/поиска.

**Q5: Почему `len("é")` может быть 1 или 2?**
A: Зависит от представления. Precomposed U+00E9 — длина 1. Разложенное 'e' + U+0301 (комбинирующий акут) — длина 2, хотя визуально один «символ» (графема). Python считает code points, а не графемы. Для подсчёта графем нужны сторонние библиотеки (`grapheme`, `regex` с `\X`).

**Q6: Что такое UnicodeDecodeError и как его обработать?**
A: Возникает, когда байты нельзя декодировать в указанной кодировке (например невалидный UTF-8). Решения: указать правильную кодировку, либо `errors='replace'` (→ U+FFFD), `errors='ignore'` (пропуск), `errors='surrogateescape'` (обратимое сохранение битых байтов). Угадывать кодировку помогает `chardet`/`charset-normalizer`.

**Q7: Чем `casefold()` отличается от `lower()`?**
A: `casefold` агрессивнее и предназначен для caseless-сравнения: немецкая 'ß' → 'ss', обрабатывает больше Unicode-случаев, чем `lower`. Для регистронезависимого сравнения текста используют `casefold`, а не `lower`.

**Q8: Как удалить диакритические знаки из текста?**
A: Нормализовать в NFD (отделит комбинирующие знаки), затем отфильтровать символы категории `Mn` (Mark, nonspacing) через `unicodedata.category(c) != 'Mn'`, и при желании собрать обратно (NFC).

**Q9: Что такое BOM и когда он появляется?**
A: Byte Order Mark (U+FEFF) — маркер порядка байтов в начале файла. UTF-16/UTF-32 используют его для определения endianness. `encode('utf-8-sig')` добавляет BOM; `'utf-8'` — нет. При чтении файлов с BOM используют `utf-8-sig`, чтобы BOM не попал в текст.

**Q10: Почему сравнение имён файлов/пользователей требует нормализации?**
A: macOS хранит имена файлов в NFD, Linux обычно в NFC — одно имя в разных формах не совпадёт побайтно. Аналогично для логинов: атакующий может зарегистрировать визуально идентичный логин в другой форме. Нормализация (часто NFKC + casefold) защищает от этого.

**Q11: Сколько байт занимает символ в UTF-8?**
A: От 1 до 4. ASCII (U+0000–U+007F) — 1 байт; кириллица/латиница с диакритикой — 2; большинство CJK — 3; эмодзи и редкие символы — 4. Поэтому `len(str) != len(str.encode('utf-8'))` для не-ASCII.

## Подводные камни (gotchas)

```python
import unicodedata

# 1. Визуально равные строки не равны без нормализации
a, b = 'é', 'é'
print(a == b)  # False!
print(unicodedata.normalize('NFC', a) == unicodedata.normalize('NFC', b))  # True

# 2. len считает code points, не графемы
print(len('👨‍👩‍👧‍👦'))  # 7 (семья = несколько code points + ZWJ), не 1

# 3. Срез может разорвать символ из суррогатной пары / комбинацию
s = 'é'  # é разложенное
print(s[0])    # 'e' — акут отделился!

# 4. encode/decode с неверной кодировкой -> мусор или ошибка
data = "Привет".encode('utf-8')
# print(data.decode('cp1251'))  # 'РџСЂРёРІРµС'... — мохибэйк

# 5. NFKC меняет данные необратимо
print(unicodedata.normalize('NFKC', '①②③'))  # '123' — потеря информации

# 6. Открытие файла без указания encoding зависит от локали ОС!
# open('f.txt')              # кодировка платформо-зависима
# open('f.txt', encoding='utf-8')  # всегда указывайте явно

# 7. str.isdigit() != unicodedata.decimal
print('²'.isdigit())   # True (надстрочная 2)
# unicodedata.decimal('²')  # ValueError — не decimal!
print('²'.isnumeric())  # True
```

## Лучшие практики

- **Всегда указывайте `encoding='utf-8'`** при открытии файлов — не полагайтесь на локаль ОС.
- **Нормализуйте на границах системы**: входящий текст приводите к NFC (хранение) или NFKC (сравнение/идентификаторы).
- Для **caseless-сравнения** используйте `unicodedata.normalize('NFC', s.casefold())`.
- Для **удаления диакритики** — NFD + фильтрация категории `Mn`.
- При декодировании внешних данных задавайте политику `errors` осознанно (`replace`/`surrogateescape`).
- Используйте `utf-8-sig` для файлов с возможным BOM.
- Не применяйте NFKC/NFKD к данным, где важна точная форма (пароли, ключи).
- Для работы с графемами/эмодзи берите `regex` (`\X`) или `grapheme`, а не срезы по индексам.
- Для определения неизвестной кодировки — `charset-normalizer`/`chardet`.

## Шпаргалка

```python
import unicodedata

# Информация о символе
unicodedata.name(c)          # имя символа
unicodedata.lookup(name)     # символ по имени
unicodedata.category(c)      # категория (Lu, Ll, Nd, Mn, ...)
unicodedata.numeric(c)       # числовое значение
unicodedata.combining(c)     # класс комбинирования
unicodedata.unidata_version  # версия Unicode БД

# Нормализация
unicodedata.normalize('NFC',  s)  # композиция (хранение)
unicodedata.normalize('NFD',  s)  # разложение
unicodedata.normalize('NFKC', s)  # совместимостная композиция (поиск)
unicodedata.normalize('NFKD', s)  # совместимостное разложение

# str <-> bytes
s.encode('utf-8')                  # str -> bytes
b.decode('utf-8')                  # bytes -> str
# errors: strict | ignore | replace | xmlcharrefreplace
#         backslashreplace | surrogateescape

# Кодировки
'utf-8'      # стандарт, 1-4 байта
'utf-8-sig'  # UTF-8 с BOM
'utf-16'     # с BOM, 2/4 байта
'ascii'      # только U+0000..U+007F
'latin-1'    # 1 байт, U+0000..U+00FF
'cp1251'     # кириллица Windows

# Категории (первая буква)
# L — буквы, N — числа, P — пунктуация, S — символы
# M — знаки (Mn — комбинирующие), Z — разделители, C — управляющие

# Идиомы
# Удалить диакритику:
''.join(c for c in unicodedata.normalize('NFD', s)
        if unicodedata.category(c) != 'Mn')
# Caseless сравнение:
unicodedata.normalize('NFC', s.casefold())
```
