# ipaddress — подготовка к собеседованию

## Что это и зачем

`ipaddress` — модуль стандартной библиотеки (с Python 3.3) для работы с **IP-адресами и сетями IPv4 и IPv6**. Предоставляет объектную модель для адресов, сетей и диапазонов, заменяя ручные манипуляции со строками и битами.

Зачем нужен:
- Валидация и нормализация IP-адресов из пользовательского ввода/конфигов.
- Проверка, входит ли адрес в подсеть (ACL, файрволы, allow/deny-листы, geo/IP-фильтры).
- Расчёт подсетей, маски, broadcast, диапазонов, числа хостов (планирование сетей, IPAM).
- Определение типа адреса: приватный, публичный, loopback, multicast, link-local.
- Корректная и единообразная работа с IPv4 и IPv6 через общий интерфейс.

Главное преимущество — не парсить адреса регулярками, а получить надёжные объекты со сравнением, арифметикой и богатыми свойствами.

## Ключевые концепции

- **Address** (`IPv4Address`/`IPv6Address`) — один адрес. Поддерживает сравнение, арифметику (`+`/`-`), приведение к `int`/`str`/`packed`-байтам.
- **Network** (`IPv4Network`/`IPv6Network`) — сеть/подсеть: адрес сети + префикс (`/24`). По умолчанию сеть должна быть «строгой» (хостовые биты нулевые), иначе `ValueError`.
- **Interface** (`IPv4Interface`/`IPv6Interface`) — адрес ВМЕСТЕ с его сетью (как `192.168.1.10/24`): «адрес хоста + к какой сети относится».
- **Префикс/маска.** `/24` ≡ `255.255.255.0`. CIDR-нотация компактно задаёт размер сети.
- **Фабрики** `ip_address()`, `ip_network()`, `ip_interface()` сами определяют v4/v6.
- **Проверка вхождения** — оператор `in`: `addr in network`.
- **Классификация** — свойства `is_private`, `is_global`, `is_loopback`, `is_multicast`, `is_link_local`, `is_reserved`, `is_unspecified`.

## Основные функции/классы/методы

### Адреса

```python
import ipaddress

# Фабрика сама определяет версию
a = ipaddress.ip_address("192.168.0.1")       # IPv4Address
b = ipaddress.ip_address("2001:db8::1")        # IPv6Address
c = ipaddress.ip_address(3232235521)           # из int -> 192.168.0.1

print(a.version)        # 4
print(int(a))           # 3232235521
print(a.packed)         # b'\xc0\xa8\x00\x01' — 4 байта
print(a.reverse_pointer)# '1.0.168.192.in-addr.arpa' (для PTR DNS)

# Арифметика и сравнение
print(a + 1)            # 192.168.0.2
print(a < ipaddress.ip_address("192.168.0.5"))  # True

# Классификация
print(a.is_private)     # True
print(a.is_global)      # False
print(ipaddress.ip_address("8.8.8.8").is_global)     # True
print(ipaddress.ip_address("127.0.0.1").is_loopback) # True
print(ipaddress.ip_address("224.0.0.1").is_multicast)# True
print(ipaddress.ip_address("169.254.0.1").is_link_local)  # True
```

### Сети

```python
import ipaddress

net = ipaddress.ip_network("192.168.1.0/24")
print(net.network_address)     # 192.168.1.0
print(net.broadcast_address)   # 192.168.1.255
print(net.netmask)             # 255.255.255.0
print(net.hostmask)            # 0.0.0.255
print(net.prefixlen)           # 24
print(net.num_addresses)       # 256 (включая network и broadcast)

# Строгий режим: хостовые биты должны быть нулевыми
ipaddress.ip_network("192.168.1.10/24")              # ValueError!
ipaddress.ip_network("192.168.1.10/24", strict=False)# OK -> 192.168.1.0/24

# Перебор адресов
for ip in ipaddress.ip_network("192.168.1.0/30"):
    print(ip)        # .0, .1, .2, .3

# Только хосты (без network и broadcast для IPv4)
list(ipaddress.ip_network("192.168.1.0/30").hosts())  # [.1, .2]
```

### Проверка вхождения адреса в сеть

```python
import ipaddress

net = ipaddress.ip_network("10.0.0.0/8")
addr = ipaddress.ip_address("10.1.2.3")

print(addr in net)                 # True
print(ipaddress.ip_address("11.0.0.1") in net)  # False

# Несколько разрешённых сетей (allow-list)
ALLOWED = [ipaddress.ip_network(c) for c in ("10.0.0.0/8", "192.168.0.0/16")]

def is_allowed(ip_str: str) -> bool:
    ip = ipaddress.ip_address(ip_str)
    return any(ip in net for net in ALLOWED)

print(is_allowed("192.168.5.5"))   # True
print(is_allowed("8.8.8.8"))       # False
```

### Подсети и суперсети

```python
import ipaddress

net = ipaddress.ip_network("192.168.0.0/24")

# Разбить /24 на четыре /26
subnets = list(net.subnets(prefixlen_diff=2))   # /26 каждая
# или указать целевой префикс:
subnets = list(net.subnets(new_prefix=26))
print(subnets)  # [192.168.0.0/26, .64/26, .128/26, .192/26]

# Объединить в более крупную сеть (суперсеть)
print(net.supernet(prefixlen_diff=1))           # 192.168.0.0/23

# Проверка отношений
small = ipaddress.ip_network("192.168.0.0/26")
print(small.subnet_of(net))     # True
print(net.supernet_of(small))   # True
print(net.overlaps(ipaddress.ip_network("192.168.0.128/25")))  # True
```

### Интерфейсы (адрес + сеть)

```python
import ipaddress

iface = ipaddress.ip_interface("192.168.1.10/24")
print(iface.ip)        # 192.168.1.10   (адрес хоста)
print(iface.network)   # 192.168.1.0/24 (сеть, к которой принадлежит)
print(iface.netmask)   # 255.255.255.0
print(iface.with_prefixlen)  # '192.168.1.10/24'
```

### Суммаризация и работа с диапазонами

```python
import ipaddress

# Покрыть произвольный диапазон минимальным набором CIDR-сетей
nets = ipaddress.summarize_address_range(
    ipaddress.ip_address("192.168.1.0"),
    ipaddress.ip_address("192.168.1.130"),
)
print(list(nets))   # [192.168.1.0/25, 192.168.1.128/31, 192.168.1.130/32]

# Свернуть список сетей в минимальный (объединить смежные)
collapsed = ipaddress.collapse_addresses([
    ipaddress.ip_network("192.168.0.0/25"),
    ipaddress.ip_network("192.168.0.128/25"),
])
print(list(collapsed))   # [192.168.0.0/24]
```

### IPv6 особенности

```python
import ipaddress

v6 = ipaddress.ip_address("2001:0db8:0000:0000:0000:0000:0000:0001")
print(v6.compressed)    # '2001:db8::1'  (сжатая форма)
print(v6.exploded)      # полная форма с нулями

# IPv4-mapped IPv6
m = ipaddress.ip_address("::ffff:192.168.0.1")
print(m.ipv4_mapped)    # 192.168.0.1
```

### Валидация ввода

```python
import ipaddress

def parse_ip(value: str):
    try:
        return ipaddress.ip_address(value)
    except ValueError:
        return None   # некорректный адрес

parse_ip("192.168.0.1")   # IPv4Address
parse_ip("999.1.1.1")     # None
parse_ip("not-ip")        # None
```

## Частые вопросы на собеседовании

**Q: В чём разница между Address, Network и Interface?**
A: Address — один IP (`192.168.1.10`). Network — сеть с префиксом и нулевыми хостовыми битами (`192.168.1.0/24`). Interface — адрес хоста вместе с его сетью (`192.168.1.10/24`), то есть «кто я и в какой сети».

**Q: Почему `ip_network("192.168.1.10/24")` бросает ошибку?**
A: По умолчанию `strict=True` требует, чтобы хостовые биты были нулевыми (адрес сети, а не хоста). `192.168.1.10` имеет ненулевые хостовые биты. Решение — `strict=False` (нормализует к `192.168.1.0/24`) или использовать `ip_interface`.

**Q: Как проверить, что адрес входит в подсеть?**
A: `ip_address(addr) in ip_network(net)`. Для нескольких сетей — `any(ip in n for n in networks)`.

**Q: Чем `is_private` отличается от `is_global`?**
A: `is_private` — адрес из приватных диапазонов (RFC 1918: 10/8, 172.16/12, 192.168/16, плюс loopback, link-local и т.п.). `is_global` — публично маршрутизируемый адрес. Обычно они взаимоисключающи для обычных адресов.

**Q: Сколько хостов в /24? Чем `num_addresses` отличается от `hosts()`?**
A: `/24` = 256 адресов (`num_addresses`), но `.hosts()` для IPv4 исключает адрес сети и broadcast → 254 пригодных для хостов. Для очень малых сетей (`/31`, `/32`) правила особые.

**Q: Как разбить сеть на подсети?**
A: `net.subnets(new_prefix=26)` или `net.subnets(prefixlen_diff=2)`. Обратно — `net.supernet()`.

**Q: Чем `subnet_of`/`supernet_of` отличаются от `overlaps`?**
A: `subnet_of`/`supernet_of` проверяют строгое отношение вложенности. `overlaps` — есть ли вообще пересечение диапазонов (необязательно вложенность).

**Q: Как сравнить или отсортировать список IP?**
A: Объекты `IPv4Address`/`IPv6Address` поддерживают сравнение и сортировку напрямую (`sorted(addrs)`), в отличие от строк, где «10» < «9» лексикографически — некорректно.

**Q: Что такое CIDR и префикс?**
A: CIDR — нотация `адрес/префикс`, где префикс — число старших бит, задающих сеть. `/24` = 24 сетевых бита, 8 хостовых → маска `255.255.255.0`.

**Q: Как обрабатывать одновременно IPv4 и IPv6?**
A: Использовать фабрики `ip_address`/`ip_network`, которые сами определяют версию, и работать через общий интерфейс. Проверять версию — через `.version`.

**Q: Как свернуть список смежных сетей в минимальный набор?**
A: `ipaddress.collapse_addresses([...])` объединяет смежные/перекрывающиеся, `summarize_address_range(start, end)` покрывает диапазон минимальным набором CIDR.

## Подводные камни (gotchas)

- **Строгий режим сетей.** `ip_network("x.x.x.10/24")` падает; помните про `strict=False` или `ip_interface`.
- **`num_addresses` включает network и broadcast**, а `.hosts()` — нет. Не путать при расчёте «сколько устройств влезет».
- **Перебор больших сетей опасен.** `list(ip_network("10.0.0.0/8"))` создаст 16 млн объектов — память/время. Итерируйте лениво и с осторожностью.
- **Сравнение разных версий.** Нельзя сравнивать IPv4 с IPv6 напрямую (`TypeError`); проверяйте `.version`.
- **`in` требует правильных типов.** `"10.0.0.1" in net` (строка) не сработает корректно — сначала `ip_address(...)`.
- **`is_global` для IPv6** учитывает свои диапазоны; не предполагайте поведение по аналогии с IPv4.
- **Ведущие нули в IPv4.** `ip_address("010.0.0.1")` в современных версиях Python — `ValueError` (раньше некоторые системы трактовали как octal — источник уязвимостей SSRF).
- **IPv4-mapped IPv6.** `::ffff:1.2.3.4` — это IPv6-объект; для фильтрации по IPv4-правилам извлекайте `.ipv4_mapped`.
- **`ip_address` бросает `ValueError`** на мусоре — оборачивайте при работе с внешним вводом.

## Лучшие практики

- Для пользовательского ввода — всегда `try/except ValueError` вокруг фабрик.
- Используйте фабрики `ip_address`/`ip_network`/`ip_interface` вместо явных классов, чтобы код работал с обеими версиями.
- Проверку вхождения делайте через объекты (`addr in net`), не через строки/регулярки.
- Для allow/deny-листов предсоздавайте объекты `ip_network` один раз и переиспользуйте.
- Сортируйте и сравнивайте IP как объекты, а не как строки.
- Не материализуйте крупные сети в список; итерируйте лениво или работайте через свойства (`network_address`, `num_addresses`).
- Учитывайте IPv4-mapped IPv6 и нормализацию (`.compressed`) при сравнении/хранении IPv6.
- При проверках безопасности (SSRF/allow-list) полагайтесь на `is_private`/`is_global` и парсинг `ipaddress`, а не на самописные регулярки.

## Шпаргалка

```python
import ipaddress as ip

# Фабрики (сами определяют v4/v6)
a = ip.ip_address("192.168.0.1")      # IPv4Address / IPv6Address
n = ip.ip_network("10.0.0.0/8")       # сеть (strict=True по умолчанию!)
i = ip.ip_interface("192.168.1.10/24")# адрес + сеть

# Сеть с хостовыми битами:
ip.ip_network("192.168.1.10/24", strict=False)   # -> 192.168.1.0/24

# Вхождение
a in n                                 # True/False
any(a in net for net in allowed_nets)  # allow-list

# Свойства адреса
a.version; int(a); a.packed; a.is_private; a.is_global; a.is_loopback
a.is_multicast; a.is_link_local

# Свойства сети
n.network_address; n.broadcast_address; n.netmask; n.prefixlen
n.num_addresses                        # с network+broadcast
list(n.hosts())                        # без network+broadcast (IPv4)

# Подсети/суперсети/отношения
list(n.subnets(new_prefix=26))
n.supernet(prefixlen_diff=1)
small.subnet_of(n); n.overlaps(other)

# Диапазоны
list(ip.summarize_address_range(a1, a2))
list(ip.collapse_addresses([n1, n2]))

# Валидация
try: ip.ip_address(value)
except ValueError: ...                 # некорректный IP

# IPv6: .compressed / .exploded / .ipv4_mapped
# Не сравнивать v4 и v6; не материализовать огромные сети в list().
```
