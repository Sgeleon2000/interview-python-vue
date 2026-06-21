# selectors / select — подготовка к собеседованию

## Что это и зачем

Модули `select` и `selectors` дают **мультиплексирование ввода-вывода (I/O multiplexing)** — возможность одним потоком следить за **множеством** файловых дескрипторов (сокетов, пайпов) и реагировать, когда какой-то из них **готов** к чтению или записи.

Зачем нужно:
- Обслуживать **тысячи соединений** в одном потоке без блокировки и без потока-на-соединение.
- Это фундамент **событийно-ориентированных** серверов (event-driven), на котором построены asyncio, Twisted, Tornado, nginx.

Идея: вместо того чтобы блокироваться на `recv()` каждого сокета по очереди, мы спрашиваем у ОС «какие из этих N сокетов готовы прямо сейчас?» и обрабатываем только их.

- **`select`** — низкоуровневый модуль с прямыми обёртками над системными вызовами `select()`, `poll()`, `epoll()`, `kqueue()`.
- **`selectors`** — высокоуровневая обёртка над `select`: единый кросс-платформенный API, который сам выбирает **наиболее эффективный** механизм ОС.

## Ключевые концепции

### Готовность дескриптора (readiness)
Дескриптор «готов на чтение», если `recv`/`accept` не заблокируется (есть данные или входящее соединение / EOF). «Готов на запись» — если `send` не заблокируется (есть место в буфере отправки). Мультиплексор сообщает о готовности, а само чтение/запись делает приложение неблокирующими вызовами.

### Событийная модель (event loop)
```
┌──────────────────────────────────────────────┐
│ 1. Регистрируем сокеты в селекторе (READ/WRITE)│
│ 2. selector.select() -> блокируемся, пока     │
│    хоть один сокет не станет готов             │
│ 3. Для каждого готового сокета -> обработчик   │
│    (accept нового / recv данных / send ответа) │
│ 4. Возврат к шагу 2                            │
└──────────────────────────────────────────────┘
```

### Механизмы ОС: select / poll / epoll / kqueue
```
┌─────────┬──────────────┬───────────────┬──────────────────────────┐
│ Механизм│ Платформа    │ Сложность     │ Ограничения              │
├─────────┼──────────────┼───────────────┼──────────────────────────┤
│ select  │ везде        │ O(N) на вызов │ лимит FD_SETSIZE (~1024) │
│ poll    │ Unix         │ O(N) на вызов │ нет жёсткого лимита FD    │
│ epoll   │ Linux        │ O(1)*         │ только Linux             │
│ kqueue  │ BSD/macOS    │ O(1)*         │ только BSD/macOS         │
└─────────┴──────────────┴───────────────┴──────────────────────────┘
* масштабируется к десяткам тысяч соединений
```
`select`/`poll` каждый раз перебирают все дескрипторы — O(N). `epoll`/`kqueue` используют событийную нотификацию ядра и не зависят линейно от числа дескрипторов — поэтому масштабируются.

### Уровневая (level-triggered) и краевая (edge-triggered) нотификация
- **Level-triggered** (по умолчанию у select/poll/epoll): сообщает о готовности, **пока** условие истинно (пока есть непрочитанные данные).
- **Edge-triggered** (epoll с `EPOLLET`): сообщает **один раз** при переходе состояния — нужно вычитывать все данные за раз, иначе пропустишь событие. Сложнее, но эффективнее.

## Основные функции/классы/методы

### Высокоуровневый API: selectors
```python
import selectors

sel = selectors.DefaultSelector()   # автоматически epoll/kqueue/select по ОС

# Регистрация: дескриптор, интересующие события, произвольные данные (data)
sel.register(sock, selectors.EVENT_READ, data=handler)
sel.modify(sock, selectors.EVENT_READ | selectors.EVENT_WRITE, data=handler)
sel.unregister(sock)

events = sel.select(timeout=None)   # ждать готовности (None = бесконечно)
# events: список пар (key, mask)
#   key.fileobj — сам сокет
#   key.data    — то, что передали при register
#   mask        — какие события сработали (READ/WRITE)
```

### Эхо-сервер на selectors (один поток, много клиентов)
```python
import selectors
import socket
import types

sel = selectors.DefaultSelector()

def accept(sock):
    conn, addr = sock.accept()          # принимаем нового клиента
    print("Подключился:", addr)
    conn.setblocking(False)             # ОБЯЗАТЕЛЬНО неблокирующий
    # Храним состояние клиента в key.data
    data = types.SimpleNamespace(addr=addr, inb=b"", outb=b"")
    # Следим и за чтением, и за записью
    sel.register(conn, selectors.EVENT_READ | selectors.EVENT_WRITE, data=data)

def service(key, mask):
    sock = key.fileobj
    data = key.data
    if mask & selectors.EVENT_READ:     # сокет готов к чтению
        recv = sock.recv(1024)
        if recv:
            data.outb += recv           # эхо: накапливаем для отправки
        else:
            print("Отключился:", data.addr)
            sel.unregister(sock)
            sock.close()
            return
    if mask & selectors.EVENT_WRITE and data.outb:   # готов к записи и есть что слать
        sent = sock.send(data.outb)
        data.outb = data.outb[sent:]    # убираем отправленное

# --- Слушающий сокет ---
lsock = socket.socket()
lsock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
lsock.bind(("127.0.0.1", 9000))
lsock.listen()
lsock.setblocking(False)
sel.register(lsock, selectors.EVENT_READ, data=None)   # data=None => слушающий
print("Сервер на :9000")

# --- Событийный цикл ---
while True:
    events = sel.select(timeout=None)        # блокируемся до готовности
    for key, mask in events:
        if key.data is None:                 # слушающий сокет -> новое соединение
            accept(key.fileobj)
        else:                                # клиентский сокет -> обслуживаем
            service(key, mask)
```

### Низкоуровневый API: select.select()
```python
import select
import socket

server = socket.socket()
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(("127.0.0.1", 9001))
server.listen()
server.setblocking(False)

inputs = [server]    # за какими сокетами следим на чтение
outputs = []         # за какими — на запись

while inputs:
    # select возвращает три списка: готовые на чтение / запись / с ошибкой
    readable, writable, exceptional = select.select(inputs, outputs, inputs, 1.0)

    for s in readable:
        if s is server:
            conn, addr = s.accept()
            conn.setblocking(False)
            inputs.append(conn)         # начинаем следить за новым клиентом
        else:
            data = s.recv(1024)
            if data:
                s.sendall(data)         # эхо
            else:                       # клиент закрыл соединение
                inputs.remove(s)
                s.close()

    for s in exceptional:               # сокеты с ошибкой
        inputs.remove(s)
        s.close()
```

### select.poll() (Unix, без лимита FD_SETSIZE)
```python
import select

poller = select.poll()
poller.register(sock.fileno(), select.POLLIN)   # следим на чтение
events = poller.poll(1000)                       # таймаут в миллисекундах
for fd, event in events:
    if event & select.POLLIN:
        ...   # дескриптор fd готов на чтение
poller.unregister(sock.fileno())
```

### select.epoll() (только Linux)
```python
import select

epoll = select.epoll()
epoll.register(sock.fileno(), select.EPOLLIN)    # level-triggered по умолчанию
# Краевая нотификация: select.EPOLLIN | select.EPOLLET
events = epoll.poll(timeout=1)                    # таймаут в секундах
for fd, event in events:
    ...
epoll.close()
```

## Частые вопросы на собеседовании

**Q1. Что такое мультиплексирование I/O и зачем оно?**
Это слежение за множеством дескрипторов одним потоком: ОС сообщает, какие из них готовы к чтению/записи, и приложение обрабатывает только их. Позволяет обслуживать тысячи соединений без блокировки и без потока на каждое.

**Q2. В чём разница между `select` и `selectors`?**
`select` — низкоуровневый модуль с прямыми обёртками над системными вызовами (`select`, `poll`, `epoll`, `kqueue`). `selectors` — высокоуровневая кросс-платформенная обёртка, которая через `DefaultSelector` сама выбирает наиболее эффективный механизм ОС. В прикладном коде рекомендуют `selectors`.

**Q3. Чем `epoll` лучше `select`?**
`select`/`poll` каждый вызов линейно перебирают все дескрипторы — O(N), а `select` ещё и ограничен `FD_SETSIZE` (~1024). `epoll` (Linux) и `kqueue` (BSD/macOS) используют событийную нотификацию ядра, масштабируются к десяткам тысяч соединений и не имеют такого лимита.

**Q4. Почему сокеты должны быть неблокирующими при мультиплексировании?**
Селектор лишь сообщает о **готовности**, но между сигналом и вызовом `recv`/`send` ситуация может измениться, и блокирующий вызов «подвесит» весь поток, остановив обслуживание всех остальных соединений. Неблокирующие сокеты гарантируют, что один клиент не заблокирует event loop.

**Q5. Что возвращает `select.select()`?**
Три списка: дескрипторы, готовые на чтение, готовые на запись, и с ошибочными условиями (`readable, writable, exceptional`). Передаются три входных списка дескрипторов и опциональный таймаут.

**Q6. Что такое level-triggered и edge-triggered?**
Level-triggered (по умолчанию): событие сообщается, пока условие истинно (пока есть непрочитанные данные). Edge-triggered (epoll + `EPOLLET`): сообщается один раз при изменении состояния — нужно вычитать все данные сразу, иначе событие потеряется. Edge эффективнее, но сложнее в реализации.

**Q7. Как мультиплексирование связано с asyncio?**
asyncio под капотом использует `selectors` (на Linux — epoll, на macOS — kqueue). Event loop регистрирует сокеты в селекторе, ждёт готовности и возобновляет соответствующие корутины. То есть asyncio — это удобная корутинная надстройка над тем же механизмом.

**Q8. Зачем `EVENT_WRITE`, ведь обычно мы читаем?**
Запись тоже может блокироваться, если буфер отправки ОС заполнен (медленный/перегруженный клиент). Регистрируясь на `EVENT_WRITE`, мы дожидаемся, когда в буфере появится место, и отправляем остаток данных, не блокируя поток. Часто на запись подписываются динамически — только когда есть что отправлять.

**Q9. Какой лимит у `select` и как его обойти?**
`select` ограничен `FD_SETSIZE` (обычно 1024 дескриптора) и линейно деградирует с их числом. Обходится переходом на `poll` (нет жёсткого лимита) либо `epoll`/`kqueue` (масштабируемые, событийные).

**Q10. Какой механизм выберет `DefaultSelector`?**
Наиболее эффективный для платформы: `EpollSelector` на Linux, `KqueueSelector` на BSD/macOS, иначе `PollSelector` или `SelectSelector`. Это скрыто от прикладного кода за единым API.

## Подводные камни (gotchas)

- **Забыли `setblocking(False)`** — блокирующий `recv`/`send` подвесит весь event loop при ложной готовности.
- **Лимит `FD_SETSIZE`** у `select` (~1024) — не используйте `select` для тысяч соединений, берите `epoll`/`kqueue`/`selectors`.
- **Постоянная подписка на `EVENT_WRITE`** — сокет почти всегда готов к записи, цикл будет крутиться вхолостую; подписывайтесь только когда есть данные на отправку.
- **`recv()` вернул `b""`** — соединение закрыто, нужно `unregister` и `close`, иначе закрытый сокет будет вечно «готов» и забьёт цикл.
- **Edge-triggered без полного вычитывания** — потеряете данные: при `EPOLLET` нужно читать в цикле до `EAGAIN`/`BlockingIOError`.
- **`poll`/`epoll` таймаут в разных единицах** — `poll` принимает миллисекунды, `epoll` — секунды; легко перепутать.
- **`epoll` только Linux, `kqueue` только BSD/macOS** — для кросс-платформенности используйте `selectors.DefaultSelector`.
- **Не сняли с регистрации закрытый сокет** — `unregister` перед `close`.

## Лучшие практики

- В прикладном коде предпочитайте **`selectors.DefaultSelector`** — кросс-платформенно и эффективно.
- Всегда переводите сокеты в **неблокирующий режим** перед регистрацией.
- Храните **состояние соединения** в `key.data` (буферы чтения/записи, адрес).
- Подписывайтесь на **`EVENT_WRITE` динамически** — только при наличии данных для отправки.
- Корректно **снимайте с регистрации и закрывайте** сокеты при EOF/ошибке.
- Для очень высоких нагрузок и удобства — берите **asyncio**, который сам управляет селектором.
- Помните про **разные единицы таймаута** у `poll` (мс) и `epoll` (с).
- При edge-triggered читайте/пишите **до `BlockingIOError`**.

## Шпаргалка

```python
import selectors, socket, types

sel = selectors.DefaultSelector()           # epoll/kqueue/select автоматически

sel.register(sock, selectors.EVENT_READ, data=...)              # подписка
sel.modify(sock, selectors.EVENT_READ | selectors.EVENT_WRITE)  # изменить интерес
sel.unregister(sock)                                            # отписка

for key, mask in sel.select(timeout=None):  # ждать готовности
    if mask & selectors.EVENT_READ:  ...    # готов к чтению
    if mask & selectors.EVENT_WRITE: ...    # готов к записи
    # key.fileobj — сокет, key.data — пользовательские данные

# --- Низкоуровнево ---
import select
r, w, e = select.select(inputs, outputs, inputs, timeout)  # списки готовых

p = select.poll(); p.register(fd, select.POLLIN); p.poll(ms)   # Unix
ep = select.epoll(); ep.register(fd, select.EPOLLIN); ep.poll(sec)  # Linux

# Обязательно: sock.setblocking(False) перед регистрацией!
```
