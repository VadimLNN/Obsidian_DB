# Сети для DevOps — полный конспект

> Конспект составлен по блоку **«2.5 Сети»** из Roadmap DevOps и расширен до практического уровня, который нужен для Linux, Docker, Kubernetes, облаков, траблшутинга и собеседований.

---

## Содержание

1. [Что DevOps должен понимать в сетях](#1-что-devops-должен-понимать-в-сетях)
2. [Модель OSI](#2-модель-osi)
3. [TCP/IP и инкапсуляция](#3-tcpip-и-инкапсуляция)
4. [Ethernet, MAC, ARP и VLAN](#4-ethernet-mac-arp-и-vlan)
5. [IP, маски, CIDR и подсети](#5-ip-маски-cidr-и-подсети)
6. [Маршрутизация и шлюз](#6-маршрутизация-и-шлюз)
7. [NAT](#7-nat)
8. [ICMP](#8-icmp)
9. [TCP](#9-tcp)
10. [UDP](#10-udp)
11. [Порты, сокеты и соединения](#11-порты-сокеты-и-соединения)
12. [DNS](#12-dns)
13. [HTTP и HTTPS](#13-http-и-https)
14. [TLS/SSL](#14-tlsssl)
15. [SSH](#15-ssh)
16. [Основные сетевые утилиты Linux](#16-основные-сетевые-утилиты-linux)
17. [tcpdump и Wireshark](#17-tcpdump-и-wireshark)
18. [Firewall: iptables / nftables](#18-firewall-iptables--nftables)
19. [Nginx, reverse proxy и балансировка](#19-nginx-reverse-proxy-и-балансировка)
20. [HAProxy](#20-haproxy)
21. [Docker Networking](#21-docker-networking)
22. [Kubernetes Networking](#22-kubernetes-networking)
23. [Сети в облаках](#23-сети-в-облаках)
24. [Алгоритм сетевого траблшутинга](#24-алгоритм-сетевого-траблшутинга)
25. [Что происходит после ввода URL в браузере](#25-что-происходит-после-ввода-url-в-браузере)
26. [Популярные вопросы на собеседовании](#26-популярные-вопросы-на-собеседовании)
27. [Практика для закрепления](#27-практика-для-закрепления)
28. [Мини-шпаргалка](#28-мини-шпаргалка)
29. [Чек-лист готовности](#29-чек-лист-готовности)

---

# 1. Что DevOps должен понимать в сетях

Сети нужны DevOps почти везде:

- сервер не доступен по SSH;
- приложение не может подключиться к БД;
- контейнер не видит другой контейнер;
- Kubernetes Service не направляет трафик в Pod;
- DNS не резолвит имя;
- Nginx возвращает `502 Bad Gateway`;
- TLS-сертификат не проходит проверку;
- порт открыт локально, но недоступен извне;
- firewall блокирует соединение;
- сервис доступен по IP, но не по доменному имени;
- есть высокий latency или packet loss.

Главная цель — не выучить все RFC, а уметь ответить на три вопроса:

1. **Куда должен идти пакет?**
2. **По какому пути он реально идёт?**
3. **На каком уровне ломается связь?**

---

# 2. Модель OSI

OSI — концептуальная модель из 7 уровней.

| № | Уровень | Что делает | Примеры |
|---|---|---|---|
| 7 | Application | Протоколы приложений | HTTP, DNS, SSH, SMTP |
| 6 | Presentation | Форматы, кодирование, шифрование | TLS, JSON, UTF-8 |
| 5 | Session | Управление сессией | концептуально RPC/session mechanisms |
| 4 | Transport | Доставка между процессами | TCP, UDP |
| 3 | Network | Маршрутизация между сетями | IPv4, IPv6, ICMP |
| 2 | Data Link | Передача внутри локального сегмента | Ethernet, MAC, VLAN |
| 1 | Physical | Сигнал и физическая среда | кабель, оптика, радио |

Для DevOps особенно важны:

- **L3** — IP, маршрутизация, подсети;
- **L4** — TCP/UDP, порты;
- **L7** — HTTP, DNS, TLS, reverse proxy.

## Как запомнить

При отправке HTTP-запроса:

```text
HTTP request
↓
TCP segment
↓
IP packet
↓
Ethernet frame
↓
физическая передача
```

На принимающей стороне всё происходит в обратном порядке.

---

# 3. TCP/IP и инкапсуляция

В реальной жизни чаще используют модель TCP/IP:

| TCP/IP | Примерное соответствие OSI |
|---|---|
| Application | OSI 5–7 |
| Transport | OSI 4 |
| Internet | OSI 3 |
| Link | OSI 1–2 |

Каждый уровень добавляет свой заголовок.

Пример:

```text
Ethernet
└── IP
    └── TCP
        └── HTTP
```

Это называется **инкапсуляцией**.

При анализе `tcpdump` или Wireshark ты буквально видишь эти уровни друг внутри друга.

---

# 4. Ethernet, MAC, ARP и VLAN

## MAC-адрес

MAC — адрес сетевого интерфейса на канальном уровне.

Пример:

```text
52:54:00:12:34:56
```

IP нужен для доставки между сетями, MAC — для доставки кадра внутри локального L2-сегмента.

Посмотреть интерфейсы:

```bash
ip link
```

или кратко:

```bash
ip -br link
```

---

## ARP

ARP связывает:

```text
IPv4 → MAC
```

Например, компьютер знает IP шлюза:

```text
192.168.1.1
```

Но Ethernet-кадр нужно отправить на MAC.

Тогда выполняется ARP-запрос:

```text
Who has 192.168.1.1?
```

Шлюз отвечает своим MAC.

Посмотреть ARP/neighbor table:

```bash
ip neigh
```

Пример:

```text
192.168.1.1 dev wlan0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

> Для IPv6 вместо ARP используется Neighbor Discovery Protocol.

---

## VLAN

VLAN позволяет логически разделить одну физическую сеть на несколько L2-сетей.

Пример:

```text
VLAN 10 — frontend
VLAN 20 — backend
VLAN 30 — database
```

Устройства из разных VLAN обычно не общаются напрямую без маршрутизатора/L3-коммутатора.

Стандарт:

```text
IEEE 802.1Q
```

---

# 5. IP, маски, CIDR и подсети

## IPv4

IPv4 состоит из 32 бит.

Пример:

```text
192.168.1.10
```

Каждая часть — 8 бит:

```text
192 . 168 . 1 . 10
```

---

## Частные IPv4-сети

Используются внутри локальных сетей и обычно не маршрутизируются напрямую в Интернет:

```text
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

Loopback:

```text
127.0.0.0/8
```

Чаще всего:

```text
127.0.0.1
```

---

## CIDR

Запись:

```text
192.168.1.0/24
```

`/24` означает, что первые 24 бита — часть сети.

Маска:

```text
255.255.255.0
```

Количество IPv4-адресов:

```text
2^(32-prefix)
```

Для `/24`:

```text
2^(32-24) = 256 адресов
```

Обычно:

```text
192.168.1.0     network address
192.168.1.1     host
...
192.168.1.254   host
192.168.1.255   broadcast
```

---

## Частые CIDR

| CIDR | Маска | Всего адресов |
|---|---|---:|
| /8 | 255.0.0.0 | 16 777 216 |
| /16 | 255.255.0.0 | 65 536 |
| /24 | 255.255.255.0 | 256 |
| /25 | 255.255.255.128 | 128 |
| /26 | 255.255.255.192 | 64 |
| /27 | 255.255.255.224 | 32 |
| /28 | 255.255.255.240 | 16 |
| /29 | 255.255.255.248 | 8 |
| /30 | 255.255.255.252 | 4 |
| /32 | 255.255.255.255 | 1 |

---

## Пример подсети

Адрес:

```text
192.168.10.70/26
```

`/26` = 64 адреса на подсеть.

Диапазоны:

```text
192.168.10.0   - 192.168.10.63
192.168.10.64  - 192.168.10.127
192.168.10.128 - 192.168.10.191
192.168.10.192 - 192.168.10.255
```

`70` попадает во второй диапазон.

Значит:

```text
Network:   192.168.10.64
Hosts:     192.168.10.65 - 192.168.10.126
Broadcast: 192.168.10.127
```

---

## IPv6

IPv6 — 128-битные адреса.

Пример:

```text
2001:db8:abcd::10
```

Loopback:

```text
::1
```

Преимущества:

- огромное адресное пространство;
- меньше потребности в NAT;
- встроенная поддержка современных механизмов автонастройки;
- активно используется в облаках и крупных сетях.

Для Junior DevOps достаточно:

- понимать формат IPv6;
- знать `::1`;
- знать, что `/64` — типичный размер IPv6-подсети;
- уметь увидеть IPv6 в `ip addr`, `ss`, DNS `AAAA`.

---

# 6. Маршрутизация и шлюз

Маршрутизация отвечает на вопрос:

> Куда отправить IP-пакет дальше?

Посмотреть таблицу маршрутов:

```bash
ip route
```

Пример:

```text
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.20
```

Это означает:

- сеть `192.168.1.0/24` доступна напрямую через `eth0`;
- весь остальной трафик идёт через шлюз `192.168.1.1`.

---

## Default gateway

Маршрут:

```text
default via 192.168.1.1
```

используется, если более специфического маршрута нет.

---

## Longest Prefix Match

Если есть:

```text
10.0.0.0/8      via A
10.10.0.0/16    via B
10.10.5.0/24    via C
```

для адреса:

```text
10.10.5.42
```

будет выбран `/24`, потому что это самый специфичный маршрут.

---

# 7. NAT

NAT = Network Address Translation.

Изменяет IP-адреса и/или порты в пакетах.

## SNAT

Меняется source address.

Типичный случай:

```text
private host → Internet
```

Было:

```text
192.168.1.10 → 8.8.8.8
```

После NAT:

```text
203.0.113.5 → 8.8.8.8
```

---

## DNAT

Меняется destination address.

Используется для проброса портов.

Например:

```text
203.0.113.5:443
↓
10.0.0.10:8443
```

---

## PAT / Masquerading

Несколько внутренних машин используют один публичный IP благодаря разным source ports.

Так работает большинство домашних роутеров и Docker bridge networking.

---

# 8. ICMP

ICMP используется для служебных сообщений IP-сети.

Например:

- Echo Request / Echo Reply — `ping`;
- Destination Unreachable;
- Time Exceeded — используется `traceroute`.

Проверка:

```bash
ping -c 4 8.8.8.8
```

Важно:

> Если `ping` не проходит, это не всегда значит, что сервер недоступен.

ICMP может быть заблокирован firewall'ом, в то время как TCP/443 работает.

---

# 9. TCP

TCP — транспортный протокол с установлением соединения.

Свойства:

- connection-oriented;
- гарантирует порядок;
- повторно отправляет потерянные данные;
- контролирует поток;
- контролирует перегрузку;
- работает с портами.

---

## TCP 3-way handshake

Установка соединения:

```text
Client                         Server

SYN -------------------------->
    <---------------------- SYN + ACK
ACK -------------------------->

Connection established
```

1. Клиент отправляет `SYN`.
2. Сервер отвечает `SYN-ACK`.
3. Клиент отправляет `ACK`.

---

## Завершение TCP

Обычно используется обмен `FIN/ACK`.

Упрощённо:

```text
FIN →
← ACK
← FIN
ACK →
```

---

## TCP flags

Часто встречаются:

```text
SYN — начало соединения
ACK — подтверждение
FIN — корректное завершение
RST — аварийный сброс соединения
PSH — передать данные приложению быстрее
```

---

## Retransmission

Если TCP-сегмент потерялся, он отправляется повторно.

Большое количество retransmissions часто означает:

- packet loss;
- проблемы канала;
- перегрузку;
- неисправное сетевое оборудование.

---

# 10. UDP

UDP — транспортный протокол без установления соединения.

Свойства:

- нет handshake;
- нет встроенной гарантии доставки;
- нет гарантии порядка;
- меньше overhead;
- подходит для latency-sensitive задач.

Используется, например, в:

- DNS;
- VoIP;
- streaming;
- games;
- QUIC / HTTP/3.

---

## TCP vs UDP

| TCP | UDP |
|---|---|
| Есть соединение | Нет соединения |
| Есть подтверждения | Нет встроенных подтверждений |
| Гарантирует порядок | Не гарантирует |
| Повторяет потерянное | Сам не повторяет |
| Больше overhead | Меньше overhead |
| HTTP/1.1, HTTP/2, SSH | DNS, VoIP, QUIC |

---

# 11. Порты, сокеты и соединения

Порт идентифицирует сетевое приложение на машине.

Диапазон:

```text
0–65535
```

Известные порты:

| Порт | Протокол |
|---:|---|
| 22 | SSH |
| 25 | SMTP |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 8080 | часто HTTP-приложения |
| 9090 | часто Prometheus / custom services |

---

## Socket

Для TCP-соединения важна комбинация:

```text
source IP
source port
destination IP
destination port
protocol
```

Например:

```text
192.168.1.20:53014 → 142.250.74.78:443 TCP
```

Поэтому много клиентов могут одновременно подключаться к одному `server:443`.

---

## Listen socket

Сервис может слушать:

```text
127.0.0.1:8080
```

Тогда он доступен только локально.

Если слушает:

```text
0.0.0.0:8080
```

то принимает IPv4-соединения на всех локальных интерфейсах.

Проверить:

```bash
ss -lntp
```

---

# 12. DNS

DNS переводит доменные имена в адреса и хранит другие типы данных.

Пример:

```text
example.com → 93.184.216.34
```

---

## Основные DNS records

| Тип | Что хранит |
|---|---|
| A | IPv4 |
| AAAA | IPv6 |
| CNAME | alias на другое имя |
| MX | почтовый сервер |
| NS | authoritative DNS server |
| TXT | произвольный текст |
| PTR | reverse DNS |
| SRV | сервис + порт |

---

## Как работает DNS

Например:

```text
app.example.com
```

Клиент сначала смотрит:

1. локальный DNS cache;
2. `/etc/hosts`;
3. системный resolver;
4. рекурсивный DNS resolver.

Если ответа нет в cache, resolver может пройти:

```text
Root DNS
↓
TLD .com
↓
authoritative DNS example.com
↓
A/AAAA answer
```

---

## `/etc/resolv.conf`

Традиционно содержит DNS-серверы:

```text
nameserver 1.1.1.1
nameserver 8.8.8.8
```

На systemd-based системах DNS может управляться через:

```bash
resolvectl
```

Проверка:

```bash
resolvectl status
```

---

## Полезные команды

```bash
dig example.com
```

Кратко:

```bash
dig +short example.com
```

Определённый тип записи:

```bash
dig MX example.com
dig AAAA example.com
```

Проверить через конкретный DNS:

```bash
dig @8.8.8.8 example.com
```

Reverse DNS:

```bash
dig -x 8.8.8.8
```

---

# 13. HTTP и HTTPS

HTTP — протокол прикладного уровня.

Типичный запрос:

```http
GET /api/users HTTP/1.1
Host: example.com
User-Agent: curl/8.x
Accept: */*
```

Ответ:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 123
```

---

## Основные методы

```text
GET      получить ресурс
POST     создать/отправить
PUT      заменить ресурс
PATCH    частично изменить
DELETE   удалить
HEAD     получить только headers
OPTIONS  узнать доступные возможности
```

---

## Частые status codes

### 2xx

```text
200 OK
201 Created
204 No Content
```

### 3xx

```text
301 Moved Permanently
302 Found
304 Not Modified
```

### 4xx

```text
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
405 Method Not Allowed
429 Too Many Requests
```

### 5xx

```text
500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

---

## Как интерпретировать 502/503/504

### 502 Bad Gateway

Reverse proxy смог принять запрос, но получил некорректный ответ от upstream.

Часто:

- backend упал;
- неправильный порт;
- connection refused;
- TLS mismatch;
- backend закрыл соединение.

### 503 Service Unavailable

Сервис временно недоступен.

Например:

- нет healthy backends;
- maintenance;
- приложение перегружено.

### 504 Gateway Timeout

Proxy не дождался ответа upstream.

Возможные причины:

- backend завис;
- network latency;
- firewall;
- слишком маленький timeout.

---

## curl

Headers:

```bash
curl -I https://example.com
```

Подробный вывод:

```bash
curl -v https://example.com
```

Показать только HTTP status:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://example.com
```

Проверить localhost с нужным Host:

```bash
curl -H 'Host: app.example.com' http://127.0.0.1
```

---

# 14. TLS/SSL

Правильнее говорить **TLS**. SSL — старое семейство протоколов.

HTTPS:

```text
HTTP over TLS
```

---

## Упрощённый TLS handshake

Для обычного HTTPS поверх TCP:

```text
1. TCP handshake
2. ClientHello
3. ServerHello
4. Сервер отправляет сертификат
5. Клиент проверяет сертификат
6. Стороны вычисляют общий session key
7. Finished
8. Дальше HTTP идёт в зашифрованном виде
```

В TLS 1.3 handshake оптимизирован и использует современные key-exchange механизмы.

---

## Что проверяется в сертификате

Клиент проверяет:

- срок действия;
- имя домена;
- цепочку сертификатов;
- доверенный CA;
- подпись;
- иногда revocation status.

---

## SNI

SNI позволяет передать имя сайта во время TLS handshake.

Это нужно, потому что один IP может обслуживать:

```text
example.com
api.example.com
shop.example.com
```

с разными сертификатами.

---

## Проверка TLS через curl

```bash
curl -v https://example.com
```

Игнорировать проверку сертификата:

```bash
curl -k https://example.com
```

> `-k` полезен для диагностики, но не должен быть нормальным решением проблем сертификата.

---

## Проверка через OpenSSL

```bash
openssl s_client -connect example.com:443 -servername example.com
```

---

# 15. SSH

SSH используется для защищённого удалённого доступа.

Обычно:

```text
TCP/22
```

Подключение:

```bash
ssh user@server
```

---

## Ключевая аутентификация

Создать ключ:

```bash
ssh-keygen -t ed25519
```

Скопировать публичный ключ:

```bash
ssh-copy-id user@server
```

Файл клиента:

```text
~/.ssh/config
```

Пример:

```sshconfig
Host prod
    HostName 203.0.113.10
    User deploy
    IdentityFile ~/.ssh/id_ed25519
```

После этого:

```bash
ssh prod
```

---

## Port forwarding

Local forwarding:

```bash
ssh -L 15432:db.internal:5432 user@server
```

После этого локальный:

```text
localhost:15432
```

будет туннелироваться к:

```text
db.internal:5432
```

через SSH-сервер.

---

# 16. Основные сетевые утилиты Linux

Roadmap отдельно выделяет:

```text
ping
traceroute / mtr
netstat / ss
telnet / curl
nc
nmap
tcpdump
wireshark
iftop
iptables
```

Ниже — что реально нужно уметь.

---

## `ip`

Современная базовая утилита вместо старых `ifconfig`/`route`.

Интерфейсы:

```bash
ip addr
```

Кратко:

```bash
ip -br addr
```

Маршруты:

```bash
ip route
```

Соседи / ARP:

```bash
ip neigh
```

---

## `ping`

Проверить базовую IP-доступность:

```bash
ping -c 4 8.8.8.8
```

Проверить DNS + сеть:

```bash
ping -c 4 example.com
```

Если IP пингуется, а домен нет — подозрение на DNS.

---

## `traceroute`

Показывает путь до узла:

```bash
traceroute example.com
```

Полезно при:

- routing problem;
- packet loss;
- неожиданном маршруте.

---

## `mtr`

Комбинация `ping + traceroute`.

```bash
mtr example.com
```

Report mode:

```bash
mtr -rw example.com
```

---

## `ss`

Главная утилита для sockets.

Все listening TCP:

```bash
ss -lnt
```

С процессами:

```bash
sudo ss -lntp
```

UDP:

```bash
sudo ss -lnup
```

Все TCP connections:

```bash
ss -tan
```

Порт 8080:

```bash
ss -lntp | grep ':8080'
```

---

## `netstat`

Старая утилита.

Пример:

```bash
netstat -lntp
```

Но в современных системах обычно предпочтительнее:

```bash
ss
```

---

## `curl`

Проверяет HTTP/HTTPS и много других протоколов.

```bash
curl -v http://server:8080/health
```

---

## `telnet`

Можно использовать для простой проверки TCP-порта:

```bash
telnet server 443
```

Но чаще удобнее `nc`.

---

## `nc` / netcat

Проверить TCP-порт:

```bash
nc -vz server 443
```

Проверить несколько портов:

```bash
nc -vz server 22 80 443
```

Слушать порт:

```bash
nc -l 9000
```

Подключиться:

```bash
nc 127.0.0.1 9000
```

---

## `nmap`

Сканирование портов.

```bash
nmap 192.168.1.10
```

Конкретные порты:

```bash
nmap -p 22,80,443 192.168.1.10
```

TCP connect scan:

```bash
nmap -sT 192.168.1.10
```

> Использовать только на своих системах или там, где есть разрешение.

---

## `iftop`

Показывает сетевой трафик по соединениям в реальном времени:

```bash
sudo iftop
```

Интерфейс:

```bash
sudo iftop -i eth0
```

---

# 17. tcpdump и Wireshark

## tcpdump

Один из самых важных инструментов сетевого траблшутинга.

Показать весь трафик:

```bash
sudo tcpdump -ni any
```

Только порт 443:

```bash
sudo tcpdump -ni any port 443
```

Только конкретный host:

```bash
sudo tcpdump -ni any host 10.0.0.15
```

TCP SYN:

```bash
sudo tcpdump -ni any 'tcp[tcpflags] & tcp-syn != 0'
```

DNS:

```bash
sudo tcpdump -ni any port 53
```

Записать в файл:

```bash
sudo tcpdump -ni any -w capture.pcap
```

---

## Как читать проблему с TCP

Если клиент отправляет:

```text
SYN
SYN
SYN
SYN
```

а ответа нет:

- пакет не доходит;
- firewall silently drops;
- проблема маршрутизации;
- сервер не отвечает.

Если видишь:

```text
SYN →
← RST
```

то хост доступен, но порт обычно закрыт или сервис не слушает.

Если:

```text
SYN →
← SYN-ACK
ACK →
```

TCP установился — проблема уже выше L4.

---

## Wireshark

GUI-анализатор пакетов.

Полезные фильтры:

```text
tcp
udp
dns
http
tls
ip.addr == 10.0.0.15
tcp.port == 443
```

---

# 18. Firewall: iptables / nftables

Firewall фильтрует сетевой трафик.

Важно понимать направления:

```text
INPUT   → трафик к текущей машине
OUTPUT  → трафик от текущей машины
FORWARD → трафик через текущую машину
```

---

## iptables

Посмотреть правила:

```bash
sudo iptables -L -n -v
```

NAT:

```bash
sudo iptables -t nat -L -n -v
```

---

## nftables

В современных Linux всё чаще используется `nftables`.

Посмотреть ruleset:

```bash
sudo nft list ruleset
```

---

## Stateful firewall

Linux firewall умеет учитывать состояние соединения через conntrack.

Пример состояний:

```text
NEW
ESTABLISHED
RELATED
INVALID
```

Частый принцип:

```text
разрешить ESTABLISHED,RELATED
разрешить нужные NEW
остальное запретить
```

---

# 19. Nginx, reverse proxy и балансировка

Nginx может работать как:

- web server;
- reverse proxy;
- load balancer;
- TLS termination point;
- static file server.

---

## Forward proxy vs Reverse proxy

### Forward proxy

```text
Client → Proxy → Internet
```

Proxy представляет клиента.

### Reverse proxy

```text
Client → Reverse Proxy → Backend
```

Proxy представляет серверную сторону.

---

## Пример reverse proxy

```nginx
server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

Трафик:

```text
Client
  ↓ :80
Nginx
  ↓ :8080
Backend
```

---

## Балансировка

```nginx
upstream backend {
    server 10.0.0.11:8080;
    server 10.0.0.12:8080;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
    }
}
```

---

## L4 vs L7 Load Balancing

### L4

Балансировка по:

```text
IP + TCP/UDP port
```

Не нужно понимать HTTP.

### L7

Балансировщик понимает:

```text
Host
URL
headers
cookies
HTTP methods
```

Может направить:

```text
/api → backend A
/static → backend B
```

---

# 20. HAProxy

HAProxy — специализированный proxy/load balancer.

Умеет:

- TCP load balancing;
- HTTP load balancing;
- health checks;
- sticky sessions;
- ACL;
- TLS termination;
- connection limits.

Важно понимать не конкретный синтаксис наизусть, а концепции:

```text
frontend
backend
server
health check
balancing algorithm
```

---

# 21. Docker Networking

Roadmap требует понимать:

```text
bridge
host
none
overlay
macvlan
```

---

## Как Docker изолирует сеть

Контейнеры используют Linux:

- network namespaces;
- virtual Ethernet (`veth`);
- bridge;
- routing;
- NAT;
- firewall rules.

---

## bridge

Типичная сеть Docker на одном host.

```text
container
   |
  veth
   |
docker bridge
   |
 host
   |
 Internet
```

Посмотреть:

```bash
docker network ls
```

```bash
docker network inspect bridge
```

---

## Проброс порта

```bash
docker run -p 8080:80 nginx
```

Означает:

```text
host:8080 → container:80
```

---

## user-defined bridge

Создать:

```bash
docker network create appnet
```

Запустить контейнер:

```bash
docker run --network appnet --name web nginx
```

В user-defined network контейнеры могут обращаться друг к другу по именам благодаря Docker DNS.

---

## host

```bash
docker run --network host ...
```

Контейнер использует network namespace хоста.

Плюсы:

- меньше сетевой изоляции;
- нет обычного bridge/NAT пути.

Минус:

- теряется изоляция портов.

---

## none

```bash
docker run --network none ...
```

Контейнер практически без внешней сети.

---

## overlay

Используется для multi-host networking, например в Docker Swarm.

Создаёт виртуальную сеть поверх нескольких узлов.

---

## macvlan

Контейнер получает отдельный MAC и выглядит в локальной сети почти как отдельное физическое устройство.

Используется реже, когда контейнер должен быть полноценным участником L2-сети.

---

# 22. Kubernetes Networking

Roadmap выделяет:

- CNI;
- kube-proxy;
- CoreDNS;
- сеть между нодами;
- Service;
- Ingress;
- NetworkPolicy.

---

## Основная идея

Kubernetes предполагает:

> Каждый Pod получает собственный IP.

Упрощённо:

```text
Pod A → Pod B
```

должны иметь возможность общаться без ручного NAT между Pod'ами.

---

## CNI

CNI = Container Network Interface.

Плагин отвечает за создание сетевой связности Pod'ов.

Примеры:

```text
Calico
Cilium
Flannel
Weave Net
```

CNI может отвечать за:

- Pod IP;
- routes;
- encapsulation;
- network policy;
- eBPF networking.

---

## CoreDNS

DNS внутри кластера.

Например Service:

```text
backend.default.svc.cluster.local
```

обычно резолвится через CoreDNS.

---

## Service

Service даёт стабильную точку доступа к набору Pod'ов.

Основные типы:

### ClusterIP

Доступен внутри кластера.

### NodePort

Открывает порт на каждой Node.

### LoadBalancer

Просит внешний/cloud load balancer.

### ExternalName

DNS alias на внешний hostname.

---

## kube-proxy

Исторически/обычно реализует Service networking через правила:

- iptables;
- IPVS;
- в некоторых современных решениях часть логики может заменяться eBPF.

Основная задача:

```text
Service IP:port
↓
один из backend Pod'ов
```

---

## Ingress

Ingress описывает HTTP/HTTPS routing.

Пример:

```text
example.com/api → api-service
example.com/web → web-service
```

Но сам объект Ingress ничего не делает без:

```text
Ingress Controller
```

Например:

- Nginx Ingress Controller;
- Traefik;
- HAProxy Ingress;
- cloud-specific controller.

---

## NetworkPolicy

Описывает, кому разрешено общаться с Pod'ом.

Пример идеи:

```text
frontend → backend : allowed
backend → database : allowed
other → database : denied
```

Важно:

> NetworkPolicy работает только если CNI plugin её поддерживает.

---

## Kubernetes troubleshooting

Посмотреть Pod:

```bash
kubectl get pods -o wide
```

Service:

```bash
kubectl get svc
```

Endpoints:

```bash
kubectl get endpoints
```

DNS из Pod:

```bash
kubectl exec -it <pod> -- nslookup kubernetes.default
```

Проверить HTTP:

```bash
kubectl exec -it <pod> -- curl http://backend:8080
```

---

# 23. Сети в облаках

Roadmap требует базово понимать:

- VPC;
- subnets;
- security groups;
- подключение к VM;
- managed services.

---

## VPC

VPC — виртуальная приватная сеть в облаке.

Пример:

```text
10.0.0.0/16
```

Внутри неё создаются подсети:

```text
10.0.1.0/24 — public
10.0.2.0/24 — private
10.0.3.0/24 — database
```

---

## Public subnet

Обычно имеет маршрут к Internet Gateway.

---

## Private subnet

Не имеет прямого маршрута из Интернета.

Для исходящего доступа часто используется:

```text
NAT Gateway
```

---

## Security Group

Виртуальный stateful firewall для VM/интерфейса/сервиса.

Например:

```text
allow TCP 22 from office IP
allow TCP 443 from Internet
allow TCP 5432 only from backend subnet
```

---

## Route Table

Определяет, куда направлять трафик.

Например:

```text
10.0.0.0/16 → local
0.0.0.0/0   → Internet Gateway
```

---

# 24. Алгоритм сетевого траблшутинга

Самый полезный навык — проверять сеть слоями.

## Шаг 1. Есть ли интерфейс?

```bash
ip -br link
ip -br addr
```

Проверить:

- интерфейс `UP`;
- IP назначен;
- правильная подсеть.

---

## Шаг 2. Есть ли маршрут?

```bash
ip route
```

Проверить default gateway и маршрут до нужной сети.

Узнать, какой маршрут будет выбран:

```bash
ip route get 8.8.8.8
```

---

## Шаг 3. Доступен ли gateway?

```bash
ping -c 4 <gateway>
```

---

## Шаг 4. Доступен ли удалённый IP?

```bash
ping -c 4 8.8.8.8
```

Если IP работает, но доменное имя нет:

```text
подозрение на DNS
```

---

## Шаг 5. Работает ли DNS?

```bash
dig example.com
```

или:

```bash
resolvectl query example.com
```

---

## Шаг 6. Открыт ли TCP-порт?

```bash
nc -vz server 443
```

или:

```bash
curl -v https://server
```

---

## Шаг 7. Слушает ли локальный сервис?

На сервере:

```bash
sudo ss -lntp
```

Критично проверить:

```text
127.0.0.1:8080
```

или:

```text
0.0.0.0:8080
```

---

## Шаг 8. Firewall

```bash
sudo nft list ruleset
```

или:

```bash
sudo iptables -L -n -v
```

---

## Шаг 9. Посмотреть пакеты

```bash
sudo tcpdump -ni any host <ip>
```

Ты сразу видишь:

- приходит ли SYN;
- отвечает ли сервер;
- есть ли RST;
- есть ли DNS;
- идёт ли TLS.

---

## Шаг 10. Проверить приложение

```bash
curl -v http://127.0.0.1:8080
```

Если локально работает, а извне нет — проблема скорее:

```text
bind address / routing / NAT / firewall / load balancer
```

---

# 25. Что происходит после ввода URL в браузере

Классический вопрос на собеседовании.

Пусть пользователь вводит:

```text
https://example.com/path
```

Упрощённая последовательность:

1. Браузер разбирает URL.
2. Проверяет cache/HSTS/локальные данные.
3. Нужно определить IP `example.com`.
4. Выполняется DNS resolution.
5. ОС выбирает маршрут до IP.
6. Если next-hop в локальной сети — определяется MAC через ARP/NDP.
7. Пакет отправляется через gateway.
8. Для HTTPS обычно устанавливается TCP-соединение с `:443`.
9. Происходит TCP 3-way handshake.
10. Происходит TLS handshake.
11. Проверяется сертификат.
12. Формируется защищённый канал.
13. Браузер отправляет HTTP request.
14. Запрос может пройти через:
    - CDN;
    - WAF;
    - Load Balancer;
    - reverse proxy;
    - Ingress;
    - backend.
15. Backend формирует HTTP response.
16. Response возвращается клиенту.
17. Браузер получает HTML.
18. Загружает CSS/JS/images/fonts.
19. Выполняет parsing/rendering.
20. Пользователь видит страницу.

Для HTTP/3 часть отличается:

```text
HTTP/3 → QUIC → UDP
```

и TLS встроен в QUIC.

---

# 26. Популярные вопросы на собеседовании

## 1. Назови уровни OSI

```text
1 Physical
2 Data Link
3 Network
4 Transport
5 Session
6 Presentation
7 Application
```

Главное для DevOps:

```text
L3 IP
L4 TCP/UDP
L7 HTTP/DNS
```

---

## 2. Чем TCP отличается от UDP?

TCP:

- connection-oriented;
- гарантирует порядок;
- есть retransmission;
- congestion control.

UDP:

- connectionless;
- нет встроенной гарантии доставки;
- меньше overhead;
- подходит для real-time и QUIC.

---

## 3. Что такое маска подсети?

Она определяет, какая часть IP относится к сети, а какая к host.

Пример:

```text
192.168.1.20/24
```

Сеть:

```text
192.168.1.0/24
```

---

## 4. Как работает DNS?

Клиент обращается к resolver'у.

Если ответа нет в cache, recursive resolver идёт к:

```text
Root → TLD → Authoritative DNS
```

и возвращает нужную запись.

---

## 5. Как происходит TLS handshake?

Кратко:

```text
ClientHello
ServerHello
certificate
certificate validation
key exchange
Finished
encrypted application data
```

---

## 6. Что такое ARP?

Механизм поиска MAC-адреса по IPv4 внутри локального L2-сегмента.

---

## 7. Что такое default gateway?

Маршрутизатор, которому отправляется пакет, если нет более специфичного маршрута.

---

## 8. Что такое NAT?

Трансляция IP-адресов/портов.

Основные варианты:

```text
SNAT
DNAT
PAT/MASQUERADE
```

---

## 9. Что происходит при `ping`?

Отправляется ICMP Echo Request, ожидается Echo Reply.

---

## 10. Почему сайт доступен по IP, но не по имени?

Частые причины:

- DNS;
- неправильная DNS-запись;
- resolver;
- `/etc/hosts`;
- split DNS;
- неправильный SNI/virtual host.

---

## 11. Почему сервис работает на localhost, но не доступен извне?

Проверить:

- слушает ли `127.0.0.1` вместо `0.0.0.0`;
- firewall;
- routing;
- security group;
- NAT/port forwarding;
- reverse proxy.

---

## 12. В чём разница 502 и 504?

```text
502 — proxy получил плохой/невалидный ответ от upstream
504 — proxy не дождался upstream
```

---

## 13. Чем reverse proxy отличается от load balancer?

Reverse proxy — общая роль посредника перед backend.

Load balancing — одна из возможных функций reverse proxy.

---

## 14. Что такое stateful firewall?

Firewall запоминает состояние соединений и может автоматически разрешать ответный трафик для `ESTABLISHED`.

---

## 15. Что такое MTU?

Maximum Transmission Unit — максимальный размер L3-пакета, который можно передать через интерфейс без фрагментации/дополнительной обработки.

Типичный Ethernet MTU:

```text
1500 bytes
```

Проблемы MTU часто проявляются как:

- часть сайтов работает;
- маленькие запросы проходят;
- большие TLS/HTTP передачи зависают.

---

# 27. Практика для закрепления

## Лаба 1. Диагностика своей машины

Выполнить:

```bash
ip -br addr
ip route
ip neigh
ss -lntup
resolvectl status
```

Ответить:

- какой IP;
- какая маска;
- какой gateway;
- какие DNS;
- какие порты слушаются.

---

## Лаба 2. DNS

```bash
dig github.com
dig A github.com
dig AAAA github.com
dig MX gmail.com
dig @1.1.1.1 github.com
```

Разобраться:

- какой resolver;
- какой TTL;
- какие записи получены.

---

## Лаба 3. TCP

Запустить:

```bash
nc -l 9000
```

Во втором терминале:

```bash
nc 127.0.0.1 9000
```

Параллельно:

```bash
sudo tcpdump -ni lo port 9000
```

Найти:

```text
SYN
SYN-ACK
ACK
```

---

## Лаба 4. HTTP

Поднять:

```bash
python -m http.server 8080
```

Проверить:

```bash
curl -v http://127.0.0.1:8080
```

Посмотреть:

```bash
ss -lntp
```

---

## Лаба 5. Nginx reverse proxy

Поднять backend на:

```text
127.0.0.1:8080
```

Настроить Nginx на:

```text
:80 → :8080
```

Проверить:

```bash
curl -v http://localhost
```

---

## Лаба 6. Docker network

Создать сеть:

```bash
docker network create labnet
```

Запустить два контейнера в одной сети и проверить обращение по container name.

Посмотреть:

```bash
docker network inspect labnet
```

---

## Лаба 7. Firewall

Поднять сервис на `:8080`.

Проверить доступ.

Добавить firewall-rule, блокирующий порт.

Проверить снова.

Посмотреть через `tcpdump`, что меняется.

---

## Лаба 8. Kubernetes

В локальном Minikube/Kind:

1. Deployment с web-приложением.
2. Service ClusterIP.
3. Проверить DNS Service.
4. Масштабировать Deployment.
5. Посмотреть endpoints.
6. Создать Ingress.
7. Добавить NetworkPolicy.

---

# 28. Мини-шпаргалка

## IP

```bash
ip -br addr
ip route
ip route get 8.8.8.8
ip neigh
```

## DNS

```bash
dig example.com
dig +short example.com
resolvectl status
```

## TCP/UDP

```bash
ss -lntup
ss -tan
nc -vz host 443
```

## HTTP

```bash
curl -v https://example.com
curl -I https://example.com
curl -s -o /dev/null -w '%{http_code}\n' https://example.com
```

## Маршрут

```bash
ping -c 4 host
traceroute host
mtr -rw host
```

## Пакеты

```bash
sudo tcpdump -ni any
sudo tcpdump -ni any port 443
sudo tcpdump -ni any host 10.0.0.10
```

## Firewall

```bash
sudo nft list ruleset
sudo iptables -L -n -v
```

## TLS

```bash
openssl s_client -connect example.com:443 -servername example.com
```

---

# 29. Чек-лист готовности

Перед переходом дальше по DevOps roadmap ты должен уметь объяснить без подсказки:

- [ ] 7 уровней OSI.
- [ ] Разницу L2 / L3 / L4 / L7.
- [ ] Что такое MAC.
- [ ] Что делает ARP.
- [ ] Что такое IPv4 и IPv6.
- [ ] Что означает `/24`, `/16`, `/32`.
- [ ] Как вычислить простую подсеть.
- [ ] Что такое default gateway.
- [ ] Как работает routing table.
- [ ] Что такое SNAT и DNAT.
- [ ] Для чего нужен ICMP.
- [ ] TCP three-way handshake.
- [ ] TCP vs UDP.
- [ ] Что такое порт и socket.
- [ ] Как работает DNS.
- [ ] Основные DNS records.
- [ ] Что происходит при HTTP-запросе.
- [ ] Основные HTTP status codes.
- [ ] Как работает TLS handshake.
- [ ] Что такое сертификат и CA.
- [ ] Как работает SSH.
- [ ] `ip addr`, `ip route`, `ip neigh`.
- [ ] `ping`, `traceroute`, `mtr`.
- [ ] `ss`.
- [ ] `curl`.
- [ ] `nc`.
- [ ] `nmap`.
- [ ] `tcpdump`.
- [ ] базово Wireshark.
- [ ] firewall: INPUT / OUTPUT / FORWARD.
- [ ] iptables / nftables на базовом уровне.
- [ ] reverse proxy.
- [ ] L4 vs L7 load balancing.
- [ ] Nginx.
- [ ] HAProxy — что это и зачем.
- [ ] Docker `bridge`.
- [ ] Docker `host`.
- [ ] Docker `none`.
- [ ] Docker `overlay`.
- [ ] Docker `macvlan`.
- [ ] Kubernetes CNI.
- [ ] CoreDNS.
- [ ] kube-proxy.
- [ ] Service.
- [ ] Ingress и Ingress Controller.
- [ ] NetworkPolicy.
- [ ] VPC.
- [ ] public/private subnet.
- [ ] Security Group.
- [ ] Алгоритм сетевого траблшутинга.
- [ ] Что происходит после ввода URL в браузере.

---

# Главное для DevOps

Если оставить пять наиболее важных навыков:

1. **IP + CIDR + routing.**
2. **TCP/UDP + ports + sockets.**
3. **DNS + HTTP + TLS.**
4. **Linux troubleshooting: `ip`, `ss`, `curl`, `dig`, `tcpdump`.**
5. **Понимание сети Docker/Kubernetes на базе Linux networking.**

Не нужно сначала становиться сетевым инженером. Для DevOps важнее уметь быстро определить, где проблема:

```text
DNS?
routing?
firewall?
port?
TCP?
TLS?
reverse proxy?
application?
Docker/Kubernetes networking?
```

Если умеешь последовательно проверить каждый слой — сетевой фундамент для дальнейшего DevOps уже есть.
