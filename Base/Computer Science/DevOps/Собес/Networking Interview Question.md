---
tags:
  - computer-science/networking
  - career/interview
  - flashcards/devops/networks
---
## 1. Обязательные вопросы из Roadmap

Что происходит после ввода URL в браузере?  
?  
URL разбирается → DNS получает IP → ОС выбирает маршрут → устанавливается TCP или QUIC-соединение → для HTTPS выполняется TLS → отправляется HTTP-запрос → CDN/LB/reverse proxy передают его backend → приходит HTTP-ответ → браузер рендерит страницу.

Как происходит TLS handshake?  
?  
ClientHello → ServerHello → сервер передаёт сертификат → клиент проверяет hostname, срок и цепочку доверия → выполняется key exchange → стороны получают сеансовые ключи → начинается зашифрованный обмен.

Что такое маска подсети?  
?  
Она определяет сетевую и host-часть IP-адреса. Например `/24` соответствует `255.255.255.0`.

Назови 7 уровней OSI.  
?  
1 Physical; 2 Data Link; 3 Network; 4 Transport; 5 Session; 6 Presentation; 7 Application.

Чем TCP отличается от UDP?  
?  
TCP устанавливает соединение и обеспечивает порядок, подтверждения и повторную передачу; UDP не устанавливает соединение и не гарантирует доставку, зато имеет меньший overhead.

Как работает DNS?  
?  
Клиент обращается к resolver. При отсутствии ответа в cache recursive resolver проходит root → TLD → authoritative DNS, получает запись, кэширует её по TTL и возвращает клиенту.

## 2. OSI и TCP/IP

На каком уровне OSI работает IP?  
?  
L3, Network.

На каком уровне OSI работают TCP и UDP?  
?  
L4, Transport.

На каком уровне находятся HTTP и DNS?  
?  
L7, Application.

Что такое инкапсуляция?  
?  
Добавление заголовков каждого уровня: данные приложения → TCP/UDP → IP → Ethernet.

Чем TCP/IP модель отличается от OSI?  
?  
TCP/IP обычно содержит Link, Internet, Transport, Application; OSI — 7 более детальных уровней.

Что такое frame, packet и segment?  
?  
Frame обычно L2 Ethernet, packet — L3 IP, segment — TCP на L4.

## 3. IP, CIDR и подсети

Что такое IPv4?  
?  
32-битная IP-адресация, например `192.168.1.10`.

Что такое IPv6?  
?  
128-битная IP-адресация, например `2001:db8::1`.

Какие IPv4-сети являются private?  
?  
`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.

Какой IPv4 loopback?  
?  
Обычно `127.0.0.1`; loopback-сеть — `127.0.0.0/8`.

Какой IPv6 loopback?  
?  
`::1`.

Что означает CIDR `/24`?  
?  
Первые 24 бита — network prefix; остаётся 8 host-бит. Всего 256 IPv4-адресов.

Сколько адресов в `/26`?  
?  
64 адреса, потому что `2^(32-26)=64`.

Сколько адресов в `/32`?  
?  
Один IPv4-адрес.

Как вычислить число IPv4-адресов по CIDR?  
?  
`2^(32-prefix)`.

Что такое network address?  
?  
Первый адрес подсети, у которого host-биты равны 0.

Что такое broadcast address?  
?  
Последний адрес традиционной IPv4-подсети, у которого host-биты равны 1.

Что такое MTU?  
?  
Maximum Transmission Unit — максимальный размер L3-пакета для интерфейса без необходимости дополнительной фрагментации; для Ethernet часто 1500 байт.

Как может проявляться проблема MTU?  
?  
Мелкие запросы проходят, а крупные TCP/TLS/HTTP передачи зависают или теряются; часто встречается в VPN и overlay-сетях.

## 4. Маршрутизация

Что такое маршрутизация?  
?  
Выбор пути для IP-пакета к сети назначения.

Что такое routing table?  
?  
Таблица сетевых префиксов и next hop/interface, по которой ядро выбирает маршрут.

Как посмотреть маршруты в Linux?  
?  
`ip route`.

Как узнать выбранный маршрут до конкретного IP?  
?  
`ip route get <IP>`.

Что такое default gateway?  
?  
Шлюз, используемый, если нет более специфичного маршрута.

Что такое longest prefix match?  
?  
Из подходящих маршрутов выбирается маршрут с наиболее длинным и специфичным префиксом.

Что означает `default via 192.168.1.1 dev eth0`?  
?  
Весь трафик без более специфичного маршрута отправляется через `192.168.1.1` по `eth0`.

## 5. Ethernet, ARP и VLAN

Что такое MAC-адрес?  
?  
Адрес сетевого интерфейса на L2, используемый Ethernet внутри локального сегмента.

Зачем нужен ARP?  
?  
Для определения MAC-адреса по IPv4-адресу внутри локального L2-сегмента.

Как посмотреть ARP/neighbor cache?  
?  
`ip neigh`.

Что используется вместо ARP в IPv6?  
?  
Neighbor Discovery Protocol через ICMPv6.

Что такое VLAN?  
?  
Логическое разделение физической L2-сети на отдельные broadcast domains.

Зачем нужен trunk port?  
?  
Для передачи трафика нескольких VLAN по одному физическому каналу, обычно с тегами 802.1Q.

## 6. ICMP, DHCP и NAT

Что такое ICMP?  
?  
Служебный протокол L3 для диагностических и управляющих сообщений: Echo, Destination Unreachable, Time Exceeded и др.

Как работает ping?  
?  
Отправляет ICMP Echo Request и ждёт Echo Reply, измеряя RTT.

Если ping не проходит, значит ли это, что сервер недоступен?  
?  
Нет. ICMP может быть заблокирован, а TCP/22 или TCP/443 при этом работать.

Как работает traceroute?  
?  
Отправляет пакеты с увеличивающимся TTL/Hop Limit; промежуточные маршрутизаторы отвечают ICMP Time Exceeded.

Что такое DHCP?  
?  
Протокол автоматической выдачи IP, маски, gateway, DNS и других сетевых параметров.

Что такое DHCP DORA?  
?  
Discover → Offer → Request → Acknowledge.

Что такое NAT?  
?  
Изменение IP-адресов и/или портов проходящего трафика.

Что такое SNAT?  
?  
Изменение source address, обычно при выходе private-сети наружу.

Что такое DNAT?  
?  
Изменение destination address/port, например при port forwarding.

Что такое PAT/Masquerading?  
?  
Много внутренних соединений используют один внешний IP и различаются по портам.

## 7. TCP, UDP и порты

Как выглядит TCP three-way handshake?  
?  
SYN → SYN-ACK → ACK.

Зачем нужен TCP handshake?  
?  
Подтвердить двустороннюю достижимость, согласовать initial sequence numbers и создать состояние соединения.

Что означает TCP SYN?  
?  
Начало установления TCP-соединения.

Что означает TCP ACK?  
?  
Подтверждение получения сегмента/данных.

Что означает TCP FIN?  
?  
Корректное закрытие направления TCP-соединения.

Что означает TCP RST?  
?  
Немедленный сброс соединения.

Что такое TCP retransmission?  
?  
Повторная отправка сегмента, если отправитель считает, что он или ACK потерян.

Что такое flow control в TCP?  
?  
Ограничение скорости отправителя возможностями получателя через receive window.

Что такое congestion control?  
?  
Регулирование скорости передачи по состоянию сети, чтобы не создавать перегрузку.

Что означает `SYN_SENT`?  
?  
Клиент отправил SYN и ждёт SYN-ACK; длительное состояние указывает на возможный drop/routing/firewall issue.

Что означает `ESTABLISHED`?  
?  
TCP handshake завершён, соединение установлено.

Что такое `TIME_WAIT`?  
?  
Состояние после активного закрытия TCP, защищающее новое соединение от старых сегментов и позволяющее повторить финальный ACK.

Что такое UDP?  
?  
Connectionless transport без встроенной гарантии доставки, порядка и retransmission.

Почему DNS часто использует UDP?  
?  
Для небольших запросов это убирает TCP handshake и снижает overhead; при необходимости DNS использует и TCP.

Что такое ephemeral port?  
?  
Временный source port, который ОС выбирает для исходящего соединения.

Что означает `127.0.0.1:8080` в LISTEN?  
?  
Сервис принимает соединения только локально через loopback.

Что означает `0.0.0.0:8080` в LISTEN?  
?  
Сервис принимает IPv4-соединения на порту 8080 на всех локальных IPv4-интерфейсах.

## 8. DNS

Что такое recursive DNS resolver?  
?  
Сервер, который получает запрос клиента и сам ищет окончательный DNS-ответ либо берёт его из cache.

Что такое authoritative DNS server?  
?  
Источник истины для конкретной DNS-zone.

Что такое DNS record A?  
?  
Hostname → IPv4.

Что такое AAAA?  
?  
Hostname → IPv6.

Что такое CNAME?  
?  
Alias одного hostname на другой hostname.

Что такое MX?  
?  
Почтовые серверы домена.

Что такое NS?  
?  
Authoritative name servers зоны.

Что такое TXT?  
?  
Текстовая DNS-запись, часто используется для verification, SPF и служебных данных.

Что такое PTR?  
?  
Reverse DNS: IP → hostname.

Что такое SRV?  
?  
Запись расположения сервиса, включая hostname и port.

Что такое TTL в DNS?  
?  
Срок, в течение которого ответ можно хранить в cache.

Почему после изменения DNS часть пользователей видит старый IP?  
?  
Из-за cache layers, которые используют старую запись до истечения TTL.

Как проверить DNS через определённый resolver?  
?  
`dig @1.1.1.1 example.com`.

Чем `dig` отличается от `getent hosts`?  
?  
`dig` проверяет DNS напрямую; `getent hosts` использует системный NSS path, включая `/etc/hosts` и DNS, поэтому лучше показывает результат для обычного приложения.

Почему `dig` может работать, а приложение не резолвит имя?  
?  
Приложение может использовать `/etc/hosts`, другой resolver/cache, NSS, контейнерный namespace или собственную конфигурацию.

## 9. HTTP и TLS

Что такое HTTP?  
?  
Прикладной request/response протокол.

Какие HTTP-методы нужно знать?  
?  
GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS.

Что означают классы 2xx/3xx/4xx/5xx?  
?  
2xx success; 3xx redirect; 4xx client-side request error; 5xx server/proxy-side error.

Чем 401 отличается от 403?  
?  
401 требует корректной аутентификации; 403 означает, что доступ запрещён.

Что означает 429?  
?  
Too Many Requests — обычно rate limit.

Что означает 502 Bad Gateway?  
?  
Proxy/gateway не получил корректный ответ от upstream.

Что означает 503 Service Unavailable?  
?  
Сервис временно недоступен, например нет healthy backend или есть overload.

Что означает 504 Gateway Timeout?  
?  
Gateway/proxy не дождался ответа upstream.

Чем HTTP отличается от HTTPS?  
?  
HTTPS — HTTP внутри TLS, который добавляет шифрование, целостность и проверку сервера.

Что такое HTTP keep-alive?  
?  
Повторное использование существующего соединения для нескольких запросов.

Чем HTTP/2 отличается от HTTP/1.1 на базовом уровне?  
?  
HTTP/2 поддерживает multiplexing нескольких потоков в одном TCP-соединении.

На чём работает HTTP/3?  
?  
На QUIC поверх UDP, с TLS 1.3, интегрированным в QUIC.

Что такое TLS?  
?  
Transport Layer Security — криптографическая защита сетевого соединения.

Чем SSL отличается от TLS?  
?  
SSL устарел; современные защищённые соединения используют TLS, хотя термин SSL всё ещё употребляется разговорно.

Что такое TLS-сертификат?  
?  
Подписанный документ, связывающий публичный ключ с доменным именем/субъектом.

Что такое CA?  
?  
Certificate Authority — удостоверяющий центр в цепочке доверия.

Что такое certificate chain?  
?  
Server certificate → intermediate CA → trusted root CA.

Что проверяет клиент в сертификате?  
?  
Hostname/SAN, срок действия, подпись, ограничения и цепочку доверия.

Что такое SNI?  
?  
Передача hostname во время TLS handshake, чтобы сервер на одном IP выбрал правильный virtual host и сертификат.

Почему HTTPS по IP может не работать, хотя по домену работает?  
?  
Сертификат может не содержать IP и без правильного SNI сервер может выбрать другой сертификат.

Как проверить TLS из терминала?  
?  
`openssl s_client -connect example.com:443 -servername example.com` или `curl -v https://example.com`.

Почему `curl -k` не является исправлением?  
?  
Он отключает проверку сертификата и только маскирует проблему; использовать его можно лишь как диагностический тест.

## 10. Proxy и балансировка

Что такое reverse proxy?  
?  
Proxy перед backend: client → reverse proxy → backend; используется для routing, TLS termination, caching и load balancing.

Что такое forward proxy?  
?  
Proxy на стороне клиента: client → proxy → внешний сервер; используется для egress control, filtering и caching.

Чем L4 load balancer отличается от L7?  
?  
L4 принимает решения по IP/port/protocol; L7 понимает HTTP Host/path/headers/cookies.

Какие алгоритмы балансировки нужно знать?  
?  
Round robin, weighted round robin, least connections, random, IP hash/sticky approaches.

Что такое health check?  
?  
Регулярная проверка backend; unhealthy endpoint исключается из распределения трафика.

Что такое TLS termination?  
?  
Расшифровка TLS на proxy/load balancer перед передачей запроса дальше.

Что такое sticky session?  
?  
Попытка направлять последовательные запросы одного клиента на тот же backend.

Что такое connection draining?  
?  
Вывод backend из балансировки с ожиданием завершения уже активных соединений.

## 11. Linux-инструменты и firewall

Как посмотреть IP-адреса Linux?  
?  
`ip -br addr`.

Как посмотреть интерфейсы?  
?  
`ip -br link`.

Как посмотреть listening TCP-порты с процессами?  
?  
`sudo ss -lntp`.

Как проверить удалённый TCP-порт?  
?  
`nc -vz host 443`.

Как проверить HTTP endpoint?  
?  
`curl -v http://host:port/path`.

Как получить только HTTP status через curl?  
?  
`curl -s -o /dev/null -w '%{http_code}\n' URL`.

Для чего нужен mtr?  
?  
Показывает путь до узла и статистику latency/loss по hops.

Для чего нужен tcpdump?  
?  
Для захвата и анализа фактических сетевых пакетов.

Как захватить TCP/443 на всех интерфейсах?  
?  
`sudo tcpdump -ni any port 443`.

Как сохранить capture в pcap?  
?  
`sudo tcpdump -ni any -w capture.pcap`.

Для чего нужен Wireshark?  
?  
Для подробного GUI-анализа pcap: TCP flags, retransmissions, DNS, TLS, HTTP и другие протоколы.

Для чего нужен nmap?  
?  
Для обнаружения хостов и проверки открытых портов; использовать только там, где есть разрешение.

Что такое firewall?  
?  
Механизм фильтрации трафика по IP, портам, протоколам, состоянию соединения и другим признакам.

Что означают INPUT, OUTPUT, FORWARD?  
?  
INPUT — трафик к машине; OUTPUT — от машины; FORWARD — маршрутизируемый через машину.

Как посмотреть iptables?  
?  
`sudo iptables -L -n -v`.

Как посмотреть nftables?  
?  
`sudo nft list ruleset`.

Что такое stateful firewall?  
?  
Firewall, который отслеживает состояние соединений, например NEW, ESTABLISHED, RELATED.

Чем DROP отличается от REJECT?  
?  
DROP молча отбрасывает пакет и обычно вызывает timeout; REJECT явно сообщает об отказе, например TCP RST или ICMP error.

## 12. Docker networking

На чём основана сеть Docker?  
?  
Network namespaces, veth pairs, Linux bridge, routing, NAT и netfilter/firewall.

Что такое Docker bridge network?  
?  
Виртуальная сеть на одном Docker host, к которой подключаются контейнеры.

Что означает `-p 8080:80`?  
?  
Порт 8080 хоста публикуется на порт 80 контейнера.

Чем user-defined bridge полезнее default bridge?  
?  
Даёт удобную изоляцию и встроенное DNS-разрешение контейнеров по именам.

Что делает `--network host`?  
?  
Контейнер использует network namespace хоста.

Что делает `--network none`?  
?  
Контейнер запускается практически без внешней сети.

Что такое overlay network?  
?  
Виртуальная сеть между несколькими hosts, например в Docker Swarm.

Что такое macvlan?  
?  
Контейнер получает отдельный MAC и выглядит как отдельное устройство в L2-сети.

## 13. Kubernetes networking

Что такое CNI в Kubernetes?  
?  
Container Network Interface — стандарт и plugin-механизм настройки сетевой связности Pod.

Для чего нужен CoreDNS?  
?  
Для DNS resolution и service discovery внутри Kubernetes.

Что такое Kubernetes Service?  
?  
Стабильная точка доступа к динамическому набору Pod.

Какие основные типы Service?  
?  
ClusterIP, NodePort, LoadBalancer, ExternalName.

Чем Ingress отличается от Ingress Controller?  
?  
Ingress — декларативные L7 routing rules; Controller — компонент, который реально их реализует.

Что такое NetworkPolicy?  
?  
Правила ingress/egress для Pod; требуется CNI с поддержкой NetworkPolicy.

Для чего нужен kube-proxy?  
?  
Традиционно реализует Service forwarding/load balancing на nodes через iptables/IPVS; часть реализаций заменяет это eBPF.

Pod доступен по Pod IP, но Service не работает. Что проверять?  
?  
Service selector → EndpointSlice/endpoints → port/targetPort → readiness → kube-proxy/service dataplane → NetworkPolicy.

Pod не резолвит Service. Что проверять?  
?  
CoreDNS, DNS config Pod, Service name/namespace, доступ к DNS Service и NetworkPolicy/CNI.

## 14. Облачные сети

Что такое VPC?  
?  
Изолированная виртуальная сеть в облаке со своими CIDR, subnets, routes и security controls.

Чем public subnet отличается от private subnet?  
?  
Public subnet имеет путь для прямой интернет-связности при наличии public IP; private subnet не имеет прямого входящего доступа и часто использует NAT для egress.

Что такое Security Group?  
?  
Обычно stateful виртуальный firewall для облачного интерфейса/ресурса.

Что такое route table в VPC?  
?  
Таблица маршрутов для subnet/VPC, определяющая next hop для сетевых префиксов.

Зачем нужен NAT Gateway?  
?  
Чтобы private resources могли инициировать исходящие соединения в Интернет без прямой входящей доступности.

## 15. Сценарии траблшутинга

Сайт не открывается. С чего начинать troubleshooting?  
?  
Scope → DNS → route/reachability → TCP port → TLS → HTTP → proxy/LB → backend → application. На каждом шаге собирать evidence.

Ping работает, но HTTPS нет. Что проверять?  
?  
TCP/443 через `nc`, listening socket через `ss`, firewall/security group, TLS/SNI/certificate через `curl -v`, reverse proxy и logs.

IP работает, а домен нет. Где искать?  
?  
DNS/NSS: `getent hosts`, `dig`, `/etc/hosts`, `nsswitch.conf`, resolver config/cache.

Сервис работает на localhost, но извне нет. Что проверять?  
?  
Bind address `127.0.0.1` vs `0.0.0.0`, firewall, routing, security group, NAT/port publishing.

Что обычно означает Connection refused?  
?  
Пакет дошёл, но соединение явно отклонено: порт не слушается либо firewall делает REJECT/RST.

Что обычно означает Connection timed out?  
?  
Ответ не приходит: firewall DROP, routing issue, недоступный host, security group или проблема обратного пути.

Как на пакетах отличить refused от timeout?  
?  
Refused обычно SYN → RST; timeout — повторяющиеся SYN без ожидаемого ответа.

tcpdump не видит входящий SYN. Где проблема?  
?  
До сервера: неправильный DNS/IP, route, upstream firewall/security group, NAT, load balancer или другая сеть.

TCP handshake успешен, но HTTP не отвечает. Что это означает?  
?  
L3/L4 уже работают; искать проблему выше: TLS, HTTP, proxy, application или dependency.

Nginx отдаёт 502. Что делать?  
?  
Посмотреть error log → проверить upstream address/port → `curl` напрямую в backend → `ss` → DNS/routing/firewall → HTTP/HTTPS mismatch.

Nginx отдаёт 504. Что делать?  
?  
Проверить время ответа backend, proxy timeout, зависимости приложения, network latency/loss и logs.

Почему после деплоя могут появляться connection reset?  
?  
Backend завершается до окончания in-flight connections; проверить graceful shutdown, readiness, termination grace и connection draining.

Почему сервис работает из одной сети, но не из другой?  
?  
Сравнить routes, firewall/security groups/NACL, source allowlists, DNS, NAT, VPN и обратный маршрут.

Как диагностировать packet loss?  
?  
Сравнить loss до gateway и конечного узла через `mtr`, посмотреть TCP retransmissions и packet capture, проверить разные маршруты.

Почему loss на промежуточном hop в mtr не всегда проблема?  
?  
Router может rate-limit ICMP ответы самому себе, продолжая нормально форвардить transit traffic; важна потеря на последующих hops и destination.

Браузер ругается на сертификат, а `curl -k` работает. Что известно?  
?  
Сеть и endpoint, вероятно, доступны, но нормальная TLS verification не проходит; проверить hostname/SNI, SAN, срок и certificate chain.

Приложение не подключается к PostgreSQL `db:5432`. Какой порядок?  
?  
DNS имени → route/network namespace → `nc` на 5432 → `ss` на DB → firewall/NetworkPolicy/SG → PostgreSQL bind/pg_hba → credentials.

После VPN пропал доступ к части сетей. Что проверить?  
?  
Routes/default route, split tunneling, DNS, overlapping CIDR, MTU и firewall.

Почему overlapping CIDR опасны?  
?  
Маршруты становятся неоднозначными, особенно при VPN/VPC peering/containers; трафик может уходить не в ту сеть.

Какой универсальный порядок сетевой диагностики?  
?  
1 interface/IP → 2 route → 3 gateway/neighbor → 4 DNS → 5 TCP/UDP port → 6 TLS → 7 HTTP → 8 proxy/LB → 9 app → 10 tcpdump для подтверждения.

## Связанные заметки

- [[Computer Networks]]
- [[Сеть Linux]]
