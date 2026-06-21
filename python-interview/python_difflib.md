# Модуль `difflib` — подготовка к собеседованию

## Что это и зачем

`difflib` — модуль стандартной библиотеки для **сравнения последовательностей** (строк, списков) и вычисления различий между ними. Применяется для:

- генерации diff-ов между версиями текста/файлов (как `diff` в Unix);
- поиска похожих строк (автодополнение, «вы имели в виду...?»);
- вычисления степени схожести (ratio);
- подсветки изменений в текстах.

В основе — алгоритм поиска самой длинной общей подпоследовательности (вариация Ratcliff/Obershelp, «gestalt pattern matching»).

## Ключевые концепции

- **`SequenceMatcher`** — ядро модуля: сравнивает две последовательности, находит совпадающие блоки, вычисляет ratio и opcodes.
- **`Differ`** — генерирует человекочитаемый построчный diff с маркерами `+`/`-`/`?`.
- **`unified_diff` / `context_diff`** — функции, дающие diff в форматах, привычных по git/patch.
- **`get_close_matches`** — нечёткий поиск похожих строк из списка.
- **Opcodes** — список операций (`replace`, `delete`, `insert`, `equal`), превращающих одну последовательность в другую.
- **`autojunk`** — эвристика, помечающая частые элементы как «мусор» (ускоряет, но может искажать ratio на больших текстах).

## Основные функции/классы/методы

### `SequenceMatcher`

```python
from difflib import SequenceMatcher

# Первый аргумент isjunk (None = нет junk-функции)
sm = SequenceMatcher(None, "питон", "пиктон")

# ratio() — мера схожести от 0.0 до 1.0
print(sm.ratio())  # 0.909... (очень похоже)

# Формула: 2.0 * M / T, где M — число совпавших символов, T — суммарная длина
sm2 = SequenceMatcher(None, "abcd", "abce")
print(sm2.ratio())  # 0.75  (2*3 / 8)

# Сравнение списков, не только строк
sm3 = SequenceMatcher(None, [1, 2, 3, 4], [1, 2, 4, 5])
print(sm3.ratio())  # 0.75
```

Поиск совпадающих блоков и opcodes:

```python
from difflib import SequenceMatcher

a = "qabxcd"
b = "abycdf"
sm = SequenceMatcher(None, a, b)

# get_matching_blocks() -> [Match(a_idx, b_idx, size), ...]
for block in sm.get_matching_blocks():
    print(block)
# Match(a=1, b=0, size=2)  -> 'ab'
# Match(a=4, b=3, size=2)  -> 'cd'
# Match(a=6, b=6, size=0)  -> терминатор

# get_opcodes() -> операции трансформации a в b
for tag, i1, i2, j1, j2 in sm.get_opcodes():
    print(f"{tag:7} a[{i1}:{i2}]={a[i1:i2]!r} b[{j1}:{j2}]={b[j1:j2]!r}")
# delete  a[0:1]='q'   b[0:0]=''
# equal   a[1:3]='ab'  b[0:2]='ab'
# replace a[3:4]='x'   b[2:3]='y'
# equal   a[4:6]='cd'  b[3:5]='cd'
# insert  a[6:6]=''    b[5:6]='f'

# find_longest_match — самая длинная общая подстрока
match = sm.find_longest_match(0, len(a), 0, len(b))
print(match)  # Match(a=1, b=0, size=2) -> 'ab'

# quick_ratio() / real_quick_ratio() — быстрые верхние оценки ratio
print(sm.quick_ratio())       # быстрая оценка (>= ratio)
print(sm.real_quick_ratio())  # ещё быстрее, грубее
```

Оптимизация при сравнении одной строки со многими:

```python
from difflib import SequenceMatcher

sm = SequenceMatcher()
sm.set_seq2("эталон")           # b кешируется (автоматическая индексация b2j)
for candidate in ["эталог", "талон", "эталон!"]:
    sm.set_seq1(candidate)       # меняем только a — быстрее
    print(candidate, round(sm.ratio(), 3))
```

### `Differ` — построчный diff

```python
from difflib import Differ

text1 = ["строка один\n", "строка два\n", "строка три\n"]
text2 = ["строка один\n", "строка ДВА\n", "строка три\n", "строка четыре\n"]

differ = Differ()
result = differ.compare(text1, text2)
print("".join(result))
#   строка один
# - строка два
# ?         ^^^
# + строка ДВА
# ?         ^^^
#   строка три
# + строка четыре

# Маркеры:
#   '  ' — без изменений
#   '- ' — есть только в первом
#   '+ ' — есть только во втором
#   '? ' — подсказка, какие символы изменились
```

### `unified_diff` — формат git/patch

```python
from difflib import unified_diff

a = "одна\nдве\nтри\n".splitlines(keepends=True)
b = "одна\nДВЕ\nтри\nчетыре\n".splitlines(keepends=True)

diff = unified_diff(a, b, fromfile="старый.txt", tofile="новый.txt", lineterm="\n")
print("".join(diff))
# --- старый.txt
# +++ новый.txt
# @@ -1,3 +1,4 @@
#  одна
# -две
# +ДВЕ
#  три
# +четыре

# n= задаёт число строк контекста (по умолчанию 3)
diff2 = unified_diff(a, b, n=0)  # без контекста
```

### `context_diff` — контекстный формат

```python
from difflib import context_diff

a = "a\nb\nc\n".splitlines(keepends=True)
b = "a\nX\nc\n".splitlines(keepends=True)

print("".join(context_diff(a, b, "f1", "f2")))
# *** f1
# --- f2
# ***************
# *** 1,3 ****
#   a
# ! b
#   c
# --- 1,3 ----
#   a
# ! X
#   c
```

### `get_close_matches` — нечёткий поиск

```python
from difflib import get_close_matches

words = ["apple", "apply", "ape", "orange", "application"]

# Найти n самых похожих на word (cutoff — минимальный ratio)
print(get_close_matches("appel", words))
# ['apple', 'apply', 'ape']

print(get_close_matches("appel", words, n=1))      # ['apple']
print(get_close_matches("appel", words, cutoff=0.8))  # ['apple']

# Практика: "вы имели в виду...?" в CLI
commands = ["install", "uninstall", "list", "search", "update"]
user_input = "instal"
suggestions = get_close_matches(user_input, commands, n=3, cutoff=0.6)
print(f"Неизвестная команда. Возможно: {suggestions}")
# Неизвестная команда. Возможно: ['install', 'uninstall']
```

### `HtmlDiff` — diff в HTML

```python
from difflib import HtmlDiff

a = ["строка 1\n", "строка 2\n"]
b = ["строка 1\n", "строка 2 изменена\n"]

html = HtmlDiff().make_table(a, b, "Было", "Стало")
# Возвращает HTML-таблицу с подсветкой различий (для отчётов/веба)
print(html[:80], "...")
```

## Частые вопросы на собеседовании

**Q1: Как `SequenceMatcher.ratio()` вычисляет схожесть?**
A: По формуле `2.0 * M / T`, где `M` — суммарное число совпадающих элементов в найденных блоках, `T` — общая длина обеих последовательностей. Результат в диапазоне [0.0, 1.0]: 1.0 — идентичны, 0.0 — ничего общего.

**Q2: Чем отличаются `ratio`, `quick_ratio`, `real_quick_ratio`?**
A: `ratio()` — точное значение (дороже всего). `quick_ratio()` — верхняя оценка на основе совпадения мультимножеств символов (без учёта порядка). `real_quick_ratio()` — ещё грубее, на основе только длин. Оба «quick» дают значение `>= ratio()` и используются для быстрой отбраковки кандидатов перед точным расчётом.

**Q3: Что такое opcodes и зачем они?**
A: `get_opcodes()` возвращает список кортежей `(tag, i1, i2, j1, j2)`, описывающих, как трансформировать последовательность `a` в `b`. `tag` ∈ {`equal`, `replace`, `delete`, `insert`}. Это основа для построения diff-ов, патчей, визуализации изменений.

**Q4: В чём разница между `Differ` и `unified_diff`?**
A: `Differ.compare()` даёт подробный человекочитаемый вывод с маркерами `+`/`-`/`?` (включая подсветку внутристрочных изменений). `unified_diff` генерирует компактный формат, совместимый с `patch`/git, с заголовками `@@` и контекстом. Для машинной обработки/патчей — `unified_diff`, для наглядности — `Differ`.

**Q5: Что делает параметр `autojunk` в `SequenceMatcher`?**
A: Эвристика: автоматически помечает «популярные» элементы (встречающиеся часто, если последовательность длиннее 200) как мусор, исключая из совпадений для ускорения. Иногда искажает результаты — отключается через `SequenceMatcher(autojunk=False, ...)`.

**Q6: Как эффективно сравнивать одну строку с тысячами других?**
A: Создать один `SequenceMatcher`, зафиксировать общую последовательность через `set_seq2()` (она индексируется один раз), затем в цикле менять только `set_seq1()`. Дополнительно отсеивать кандидатов через `quick_ratio()`/`real_quick_ratio()` перед `ratio()`.

**Q7: Какой алгоритм лежит в основе difflib?**
A: Вариация алгоритма Ratcliff/Obershelp («gestalt pattern matching»): рекурсивно находит самую длинную общую подстроку (без junk), затем рекурсивно применяет тот же подход к частям слева и справа от неё.

**Q8: Подходит ли `get_close_matches` для опечаток/автокоррекции?**
A: Да, для простых случаев («вы имели в виду...?»). Но это не настоящее расстояние Левенштейна — основано на ratio Ratcliff/Obershelp. Для серьёзной нечёткой обработки берут специализированные библиотеки (`rapidfuzz`, `python-Levenshtein`).

**Q9: Почему важен `keepends=True` при `splitlines`?**
A: Функции вроде `unified_diff` ожидают список строк **с** символами перевода строки (`\n`). `splitlines(keepends=True)` их сохраняет; иначе вывод diff будет без переносов и параметр `lineterm` придётся настраивать вручную.

## Подводные камни (gotchas)

```python
from difflib import SequenceMatcher, unified_diff

# 1. ratio НЕ симметричен по производительности, но симметричен по значению
print(SequenceMatcher(None, "ab", "ba").ratio())  # 0.5

# 2. ratio учитывает ПОРЯДОК (в отличие от quick_ratio)
print(SequenceMatcher(None, "abc", "cba").ratio())       # 0.333
print(SequenceMatcher(None, "abc", "cba").quick_ratio()) # 1.0 (только символы!)

# 3. autojunk искажает результат на длинных текстах
long_a = "x" * 300
long_b = "x" * 300 + "y"
print(SequenceMatcher(None, long_a, long_b).ratio())              # с autojunk
print(SequenceMatcher(None, long_a, long_b, autojunk=False).ratio())

# 4. unified_diff требует список строк, не одну строку
text = "a\nb\nc"
# unified_diff(text, ...)  # сравнит ПОСИМВОЛЬНО, не построчно!
lines = text.splitlines(keepends=True)  # правильно

# 5. get_close_matches возвращает [] если ничего не прошло cutoff
print(get_close_matches("zzz", ["apple"], cutoff=0.6) if False else None)

# 6. Differ.compare принимает списки строк, результат — генератор
from difflib import Differ
result = Differ().compare(["a\n"], ["b\n"])
# print(result)  # это генератор; нужно "".join(result) или list()
```

## Лучшие практики

- Для построчного сравнения **разбивайте через `splitlines(keepends=True)`**.
- Для масштабного нечёткого поиска используйте `set_seq2` + `quick_ratio` для предфильтрации.
- На больших текстах рассмотрите `autojunk=False`, если точность важнее скорости.
- Для diff в стиле git/patch — `unified_diff`; для наглядного — `Differ`; для веба — `HtmlDiff`.
- Помните: difflib хорош для умеренных объёмов; для тяжёлой нечёткой обработки берите `rapidfuzz`.
- `get_close_matches` — отличный UX для подсказок команд/имён в CLI.
- Подбирайте `cutoff` под задачу: выше — строже, ниже — больше ложных совпадений.

## Шпаргалка

```python
from difflib import (SequenceMatcher, Differ, unified_diff,
                     context_diff, get_close_matches, HtmlDiff)

# SequenceMatcher
sm = SequenceMatcher(None, a, b)
sm.ratio()                 # точная схожесть 0..1
sm.quick_ratio()           # быстрая верхняя оценка
sm.real_quick_ratio()      # ещё быстрее
sm.get_matching_blocks()   # совпадающие блоки
sm.get_opcodes()           # операции трансформации
sm.find_longest_match(...) # длиннейшая общая подстрока
sm.set_seq1(x); sm.set_seq2(y)  # переиспользование

# Diff-функции (вход: списки строк с keepends=True)
unified_diff(a, b, fromfile=, tofile=, n=3)  # формат patch/git
context_diff(a, b, ...)                       # контекстный формат
Differ().compare(a, b)                        # маркеры + - ? (генератор)

# Нечёткий поиск
get_close_matches(word, possibilities, n=3, cutoff=0.6)

# ratio = 2*M / T  (M — совпавших, T — суммарная длина)
# opcodes tags: equal | replace | delete | insert
# Differ маркеры: '  ' '- ' '+ ' '? '
```
