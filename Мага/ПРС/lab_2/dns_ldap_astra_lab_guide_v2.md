# Практическая работа: DNS + LDAP на двух Debian VM

> Переработанная версия гайда под **реальную конфигурацию** двух VM:
>
> - `astra` — `192.168.56.103`
> - `astra2` — `192.168.56.104`
> - обе VM работают под **Debian**
> - гипервизор — VirtualBox
> - `eth0` — внутренняя сеть между VM
> - `eth1` — выход в интернет через VirtualBox NAT
> - сеть управляется стандартным Debian `ifupdown` (`networking.service`)
> - NetworkManager и `systemd-networkd` не используются

---

# 0. Как пользоваться этим гайдом

Это не просто список команд.

У каждого крупного этапа есть четыре части:

1. **Что делаем** — какое изменение вносим.
2. **Зачем** — какую часть задания это реализует.
3. **Как это работает** — что происходит внутри Linux/BIND/LDAP.
4. **Как проверить** — команда, по которой можно доказать, что этап выполнен.

Лучше выполнять лабораторную строго сверху вниз.

Не переходи к BIND, пока полностью не работает сеть.  
Не переходи к LDAP, пока полностью не работает DNS.

---

# 1. Что требуется построить

Есть два вычислительных узла.

## Узел 1

```text
hostname: astra

LAN IPv4:
192.168.56.103/24

LAN IPv6:
fd00:56::103/64

роль:
DNS1
LDAP server
```

## Узел 2

```text
hostname: astra2

LAN IPv4:
192.168.56.104/24

LAN IPv6:
fd00:56::104/64

роль:
DNS2
LDAP client
```

Внутренняя сеть:

```text
IPv4: 192.168.56.0/24
IPv6: fd00:56::/64
```

---

# 2. Итоговая схема

```text
                            INTERNET
                               │
                      VirtualBox NAT
                               │
              ┌────────────────┴────────────────┐
              │                                 │
            eth1                              eth1
              │                                 │
       ┌──────────────┐                  ┌──────────────┐
       │    astra     │                  │    astra2    │
       │    DNS1      │                  │    DNS2      │
       │ LDAP SERVER  │                  │ LDAP CLIENT  │
       └──────┬───────┘                  └──────┬───────┘
              │ eth0                            │ eth0
              │                                 │
     192.168.56.103                     192.168.56.104
       fd00:56::103                       fd00:56::104
              │                                 │
              └────────── LAB LAN ──────────────┘
```

## Почему нужны два интерфейса

`eth0` нужен только для лабораторной сети:

```text
astra <-----> astra2
```

По нему будут идти:

- DNS-запросы;
- zone transfer;
- `rsync`;
- LDAP;
- SSH между узлами.

`eth1` нужен только для внешней сети:

```text
VM -> VirtualBox NAT -> Интернет
```

Через него нужны:

- `apt update`;
- установка пакетов;
- запросы DNS1 к forward-серверам.

Так мы не смешиваем лабораторную сеть и Интернет.

---

# 3. Реальное состояние сетевой подсистемы

На `astra` уже было проверено:

```bash
systemctl is-active NetworkManager
```

Результат:

```text
inactive
```

```bash
systemctl is-active networking
```

Результат:

```text
active
```

```bash
systemctl is-active systemd-networkd
```

Результат:

```text
inactive
```

Следовательно, сетью управляет:

```text
ifupdown
```

Основной файл:

```text
/etc/network/interfaces
```

содержит:

```text
source /etc/network/interfaces.d/*
```

Поэтому удобнее хранить конфигурацию каждого интерфейса отдельно:

```text
/etc/network/interfaces.d/eth0
/etc/network/interfaces.d/eth1
```

---

# 4. Важное замечание для Ghostty

При SSH из Ghostty Debian может получить:

```text
TERM=xterm-ghostty
```

Если на сервере нет соответствующей записи `terminfo`, `nano` выдаёт:

```text
Error opening terminal: xterm-ghostty
```

Перед использованием `nano` выполни:

```bash
export TERM=xterm-256color
```

Проверка:

```bash
echo $TERM
```

Должно быть:

```text
xterm-256color
```

Это не проблема `nano` и не проблема SSH.  
Удалённая система просто не знает тип терминала `xterm-ghostty`.

---

# ЧАСТЬ I. СЕТЬ

# 5. Настраиваем `astra`

> Ниже команды предполагают, что ты уже работаешь от `root`.
> Если работаешь обычным пользователем, добавляй `sudo`.

---

## 5.1. Внутренний интерфейс `eth0`

Открыть:

```bash
nano /etc/network/interfaces.d/eth0
```

Полностью записать:

```text
auto eth0

iface eth0 inet static
    address 192.168.56.103
    netmask 255.255.255.0

iface eth0 inet6 static
    address fd00:56::103
    netmask 64
```

## Что мы сделали

IPv4:

```text
192.168.56.103/24
```

теперь закреплён за `astra`.

IPv6:

```text
fd00:56::103/64
```

тоже закреплён за `astra`.

## Почему `eth0` должен быть static

На этой машине будет DNS-сервер и LDAP-сервер.

Другой узел должен всегда знать, что DNS1/LDAP находится именно здесь:

```text
192.168.56.103
```

Если оставить DHCP, адрес теоретически может измениться, после чего сломаются:

- slave-зоны;
- LDAP client;
- `rsync`;
- конфигурация DNS.

## Почему здесь нет `gateway`

Потому что `eth0` — локальная сеть.

Через него не должен идти Интернет.

Default gateway появится на `eth1`.

---

## 5.2. Внешний интерфейс `eth1`

Создать:

```bash
nano /etc/network/interfaces.d/eth1
```

Записать:

```text
auto eth1

iface eth1 inet dhcp
```

## Зачем DHCP

VirtualBox NAT содержит свой DHCP-сервер.

Он должен автоматически выдать VM примерно:

```text
IPv4:    10.0.3.15/24
gateway: 10.0.3.2
```

Раньше `eth1` был физически поднят, но IPv4 не получал, потому что у Debian не было конфигурации:

```text
iface eth1 inet dhcp
```

Именно поэтому приходилось временно выполнять:

```bash
ip addr add 10.0.3.15/24 dev eth1
ip route add default via 10.0.3.2 dev eth1
```

После настройки `ifupdown` вручную делать это больше не нужно.

---

# 6. Настраиваем `astra2`

## 6.1. `eth0`

Открыть:

```bash
nano /etc/network/interfaces.d/eth0
```

Записать:

```text
auto eth0

iface eth0 inet static
    address 192.168.56.104
    netmask 255.255.255.0

iface eth0 inet6 static
    address fd00:56::104
    netmask 64
```

## Почему здесь `.104`

Узлы должны иметь разные адреса в одной сети:

```text
astra  = 192.168.56.103
astra2 = 192.168.56.104
```

---

## 6.2. `eth1`

Создать:

```bash
nano /etc/network/interfaces.d/eth1
```

Записать:

```text
auto eth1

iface eth1 inet dhcp
```

---

# 7. Применяем сетевую конфигурацию

Поскольку SSH идёт через `eth0`, безопаснее не делать:

```bash
systemctl restart networking
```

из текущей SSH-сессии.

После сохранения файлов перезагрузи каждую VM:

```bash
reboot
```

После загрузки снова подключись.

Для первой:

```bash
ssh root@192.168.56.103
```

Для второй:

```bash
ssh root@192.168.56.104
```

Если используешь Ghostty и снова понадобится `nano`:

```bash
export TERM=xterm-256color
```

---

# 8. Проверяем сеть после перезагрузки

## На `astra`

```bash
ip -br a
```

Ожидаем примерно:

```text
lo     UNKNOWN  127.0.0.1/8 ::1/128
eth0   UP       192.168.56.103/24 fd00:56::103/64 fe80::...
eth1   UP       10.0.3.15/24 ...
```

Проверить маршруты:

```bash
ip route
```

Ключевые строки:

```text
default via 10.0.3.2 dev eth1
192.168.56.0/24 dev eth0
```

### Что означает `default`

```text
default via 10.0.3.2 dev eth1
```

означает:

> если адрес назначения не относится ни к одной известной локальной сети, отправить пакет через шлюз VirtualBox NAT на `eth1`.

### Что означает локальный маршрут

```text
192.168.56.0/24 dev eth0
```

означает:

> любые адреса `192.168.56.x` доступны напрямую через `eth0`.

---

## На `astra2`

```bash
ip -br a
ip route
```

Ожидаем:

```text
eth0 -> 192.168.56.104/24
        fd00:56::104/64

eth1 -> IPv4 от VirtualBox NAT
```

и:

```text
default -> eth1
192.168.56.0/24 -> eth0
```

---

# 9. Проверяем связность между VM

## IPv4

С `astra`:

```bash
ping -c 3 192.168.56.104
```

С `astra2`:

```bash
ping -c 3 192.168.56.103
```

Если ответы есть, внутренняя IPv4-сеть работает.

---

## IPv6

С `astra`:

```bash
ping -6 -c 3 fd00:56::104
```

С `astra2`:

```bash
ping -6 -c 3 fd00:56::103
```

## Зачем мы добавляли отдельные `fd00:56::...`

Адреса вида:

```text
fe80::...
```

у тебя уже были автоматически.

Это `link-local` IPv6. Они предназначены только для локального сегмента и обычно требуют указания интерфейса.

Для лабораторной удобнее создать свою стабильную IPv6-сеть:

```text
fd00:56::/64
```

`fd00::/8` относится к ULA — локальным IPv6-адресам.

---

# 10. Проверяем Интернет

С обеих VM:

```bash
ping -c 3 8.8.8.8
```

Это проверка Интернета **без DNS**.

Далее:

```bash
ping -c 3 debian.org
```

Это уже проверяет:

1. маршрутизацию;
2. доступ во внешнюю сеть;
3. работу текущего DNS resolver.

Если `8.8.8.8` работает, а `debian.org` нет:

```bash
cat /etc/resolv.conf
```

Проблема тогда именно в DNS, а не в маршрутизации.

---

# 11. Добавляем временные локальные имена в `/etc/hosts`

Пока наш BIND ещё не настроен, Linux сам не знает имён `dns1` и `dns2`.

На обеих VM:

```bash
nano /etc/hosts
```

Добавить:

```text
192.168.56.103  astra dns1
192.168.56.104  astra2 dns2

fd00:56::103    astra dns1
fd00:56::104    astra2 dns2
```

Проверка:

```bash
getent hosts astra
getent hosts astra2
```

`/etc/hosts` здесь нужен только как локальная таблица соответствия имён и IP.

Это ещё **не DNS**.

---

# ЧАСТЬ II. DNS / BIND9

# 12. Что такое DNS в этой лабораторной

DNS сопоставляет имена и данные.

Например:

```text
astra.zone1.local
```

может соответствовать:

```text
192.168.56.103
```

Это обычная `A`-запись.

Для IPv6 используется `AAAA`:

```text
astra.zone1.local -> fd00:56::103
```

Обратное преобразование:

```text
192.168.56.103 -> astra.zone1.local
```

делается через reverse DNS и `PTR`.

---

# 13. Что такое DNS-зона

Зона — часть пространства DNS, за которую отвечает конкретный DNS-сервер.

Например:

```text
zone1.local
```

В ней могут находиться:

```text
ns1.zone1.local
ns2.zone1.local
astra.zone1.local
astra2.zone1.local
```

В BIND зона описывается:

1. блоком в конфигурации;
2. файлом с DNS-записями.

---

# 14. Master и Slave

В старой терминологии BIND:

```text
Master
Slave
```

В новой часто встречаются:

```text
Primary
Secondary
```

Смысл одинаковый.

## Master

Хранит исходный файл зоны.

Изменения делаем именно здесь.

## Slave

Сам получает копию зоны с master.

Файл slave вручную редактировать не нужно.

Передача зоны называется:

```text
zone transfer
```

Полная передача:

```text
AXFR
```

Инкрементальная:

```text
IXFR
```

---

# 15. Распределение зон по заданию

## DNS1 — `astra`

```text
zone1.local                    Master
zone2.local                    Slave
zone3.local                    Master
магистратура.сибгути.local     Slave
IPv4 reverse                   Master
IPv6 reverse                   Slave
```

## DNS2 — `astra2`

```text
zone1.local                    Slave
zone2.local                    Master
zone3.local                    Master
магистратура.сибгути.local     Master
IPv4 reverse                   Slave
IPv6 reverse                   Master
```

Особенность:

```text
zone3.local
```

не должна реплицироваться штатным zone transfer.

Для неё используем:

```text
rsync
```

---

# 16. Устанавливаем BIND

На обеих VM:

```bash
apt update
apt install -y bind9 bind9-utils dnsutils rsync
```

Что устанавливается:

```text
bind9
```

сам DNS-сервер.

```text
bind9-utils
```

служебные программы BIND:

- `named-checkconf`;
- `named-checkzone`;
- `rndc`.

```text
dnsutils
```

даёт `dig`.

```text
rsync
```

понадобится для `zone3.local`.

Проверить:

```bash
named -v
dig -v
```

---

# 17. Запускаем BIND

Попробовать:

```bash
systemctl enable --now bind9
systemctl status bind9 --no-pager
```

Если в конкретной версии Debian unit называется `named`:

```bash
systemctl enable --now named
systemctl status named --no-pager
```

---

# 18. Как устроены файлы BIND

Основные файлы:

```text
/etc/bind/named.conf
/etc/bind/named.conf.options
/etc/bind/named.conf.local
```

## `named.conf`

Главный файл.

Обычно он просто подключает остальные.

## `named.conf.options`

Общие настройки сервера:

- на каких IP слушать;
- кому разрешать запросы;
- разрешена ли рекурсия;
- какие использовать forwarders.

## `named.conf.local`

Именно здесь объявим наши лабораторные зоны.

---

# 19. Каталоги зон

На обеих VM:

```bash
mkdir -p /etc/bind/zones
mkdir -p /var/cache/bind/slaves
chown bind:bind /var/cache/bind/slaves
```

## Почему два разных каталога

### Master

```text
/etc/bind/zones/
```

Файлы создаём и редактируем сами.

### Slave

```text
/var/cache/bind/slaves/
```

Файлы создаёт сам процесс BIND при zone transfer.

Поэтому BIND должен иметь право записи туда.

---

# 20. Логирование DNS

По заданию требуется подробное логирование.

На обеих VM:

```bash
mkdir -p /var/log/named
chown bind:bind /var/log/named
```

Создать:

```bash
nano /etc/bind/named.conf.logging
```

Содержимое:

```conf
logging {
    channel main_log {
        file "/var/log/named/named.log" versions 5 size 10m;
        severity debug 3;
        print-time yes;
        print-severity yes;
        print-category yes;
    };

    channel query_log {
        file "/var/log/named/query.log" versions 5 size 10m;
        severity info;
        print-time yes;
        print-severity yes;
        print-category yes;
    };

    category default   { main_log; };
    category general   { main_log; };
    category security  { main_log; };
    category config    { main_log; };
    category resolver  { main_log; };
    category notify    { main_log; };
    category xfer-in   { main_log; };
    category xfer-out  { main_log; };
    category update    { main_log; };
    category queries   { query_log; };
};
```

Открыть:

```bash
nano /etc/bind/named.conf
```

Убедиться, что подключены:

```conf
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";
include "/etc/bind/named.conf.logging";
```

Проверка синтаксиса:

```bash
named-checkconf
```

## Почему команда ничего не пишет при успехе

Для многих Unix-утилит нормальный результат:

```text
нет вывода
exit code 0
```

означает:

```text
ошибок нет
```

---

# 21. Общие настройки DNS1

На `astra`:

```bash
nano /etc/bind/named.conf.options
```

Содержимое:

```conf
acl lab-net {
    127.0.0.0/8;
    ::1;
    192.168.56.0/24;
    fd00:56::/64;
};

options {
    directory "/var/cache/bind";

    listen-on {
        127.0.0.1;
        192.168.56.103;
    };

    listen-on-v6 {
        ::1;
        fd00:56::103;
    };

    allow-query {
        lab-net;
    };

    recursion yes;

    allow-recursion {
        lab-net;
    };

    forwarders {
        1.1.1.1;
        8.8.8.8;
    };

    forward first;

    dnssec-validation auto;
};
```

---

## Зачем нужен `acl lab-net`

Это список сетей, которым мы доверяем:

```text
127.0.0.0/8      localhost IPv4
::1              localhost IPv6
192.168.56.0/24  наша LAN IPv4
fd00:56::/64     наша LAN IPv6
```

Потом этот ACL используется в:

```conf
allow-query
allow-recursion
```

---

## Зачем `listen-on`

```conf
listen-on {
    127.0.0.1;
    192.168.56.103;
};
```

BIND будет принимать IPv4 DNS-запросы именно на этих адресах.

---

## Зачем `recursion yes`

DNS1 должен уметь отвечать не только за наши локальные зоны.

Например клиент спросит:

```text
debian.org
```

Наш DNS не является authoritative для `debian.org`.

Поэтому ему нужно обратиться дальше.

---

## Что такое `forwarders`

```conf
forwarders {
    1.1.1.1;
    8.8.8.8;
};
```

Если DNS1 сам не знает внешний домен, он сначала спрашивает эти DNS.

Схема:

```text
client
  |
  v
DNS1
  |
  +---- знает локальную зону? ----> отвечает сам
  |
  +---- не знает? ----> 1.1.1.1 / 8.8.8.8
```

---

## Почему ограничиваем `allow-recursion`

Нельзя без необходимости делать открытый recursive resolver для всех.

Поэтому:

```conf
allow-recursion {
    lab-net;
};
```

разрешает рекурсию только нашим узлам.

---

# 22. Общие настройки DNS2

На `astra2`:

```bash
nano /etc/bind/named.conf.options
```

Содержимое:

```conf
acl lab-net {
    127.0.0.0/8;
    ::1;
    192.168.56.0/24;
    fd00:56::/64;
};

options {
    directory "/var/cache/bind";

    listen-on {
        127.0.0.1;
        192.168.56.104;
    };

    listen-on-v6 {
        ::1;
        fd00:56::104;
    };

    allow-query {
        lab-net;
    };

    recursion no;

    dnssec-validation auto;
};
```

DNS2 в нашей схеме используется как authoritative DNS для лабораторных зон.

Рекурсивным resolver по условию должен быть DNS1, поэтому на DNS2:

```conf
recursion no;
```

---

# 23. Зона `zone1.local`

По заданию:

```text
astra  / DNS1 -> Master
astra2 / DNS2 -> Slave
```

Схема:

```text
/etc/bind/zones/db.zone1.local
              |
              | AXFR / IXFR
              v
           DNS2
```

---

## 23.1. DNS1: объявляем master

На `astra`:

```bash
nano /etc/bind/named.conf.local
```

Добавить:

```conf
zone "zone1.local" {
    type master;
    file "/etc/bind/zones/db.zone1.local";

    allow-transfer {
        192.168.56.104;
        fd00:56::104;
    };

    also-notify {
        192.168.56.104;
        fd00:56::104;
    };
};
```

### `allow-transfer`

Разрешает DNS2 получить копию зоны.

Без этого slave не сможет синхронизироваться.

### `also-notify`

После изменения master сообщает DNS2:

> зона изменилась, проверь новый serial.

Это ускоряет обновление.

---

## 23.2. DNS1: файл зоны

Создать:

```bash
nano /etc/bind/zones/db.zone1.local
```

Записать:

```dns
$TTL 3600

@   IN  SOA ns1.zone1.local. admin.zone1.local. (
        2026091801
        3600
        900
        604800
        3600
)

    IN  NS  ns1.zone1.local.
    IN  NS  ns2.zone1.local.

ns1     IN  A       192.168.56.103
ns1     IN  AAAA    fd00:56::103

ns2     IN  A       192.168.56.104
ns2     IN  AAAA    fd00:56::104

astra   IN  A       192.168.56.103
astra   IN  AAAA    fd00:56::103

astra2  IN  A       192.168.56.104
astra2  IN  AAAA    fd00:56::104

dns2        IN  NS      ns.dns2.zone1.local.
ns.dns2     IN  A       192.168.56.104
ns.dns2     IN  AAAA    fd00:56::104
```

---

# 24. Что означают записи в zone-файле

## `$TTL`

```dns
$TTL 3600
```

Ответ можно кэшировать 3600 секунд.

То есть:

```text
1 час
```

---

## `SOA`

```dns
@ IN SOA ns1.zone1.local. admin.zone1.local. (
    2026091801
    ...
)
```

SOA означает:

```text
Start Of Authority
```

Это служебная информация о зоне.

### Первый параметр

```text
ns1.zone1.local.
```

основной сервер зоны.

### Второй параметр

```text
admin.zone1.local.
```

условный e-mail администратора.

В SOA символ `@` e-mail заменяется точкой.

То есть это примерно:

```text
admin@zone1.local
```

---

# 25. Что такое serial

```text
2026091801
```

Это версия zone-файла.

Slave сравнивает serial master со своим.

Если master больше:

```text
Master: 2026091802
Slave:  2026091801
```

slave понимает:

> моя копия устарела.

И забирает новую.

Удобный формат:

```text
YYYYMMDDNN
```

Например:

```text
2026091801
2026091802
2026091803
```

После каждого изменения master-зоны serial необходимо увеличивать.

---

# 26. `NS`, `A` и `AAAA`

```dns
IN NS ns1.zone1.local.
```

`NS` показывает DNS-сервер зоны.

```dns
ns1 IN A 192.168.56.103
```

`A` — IPv4.

```dns
ns1 IN AAAA fd00:56::103
```

`AAAA` — IPv6.

---

# 27. Проверяем master-файл `zone1.local`

На `astra`:

```bash
named-checkzone zone1.local /etc/bind/zones/db.zone1.local
```

Нормальный результат:

```text
zone zone1.local/IN: loaded serial 2026091801
OK
```

До успешного `OK` BIND перезапускать не надо.

---

# 28. DNS2: slave `zone1.local`

На `astra2`:

```bash
nano /etc/bind/named.conf.local
```

Добавить:

```conf
zone "zone1.local" {
    type slave;

    masters {
        192.168.56.103;
        fd00:56::103;
    };

    file "/var/cache/bind/slaves/db.zone1.local";
};
```

## Почему файла зоны вручную нет

Потому что DNS2 — slave.

Он должен получить файл автоматически с:

```text
192.168.56.103
```

---

# 29. Проверяем `zone1.local`

Сначала:

```bash
named-checkconf
```

На обеих VM.

Затем перезапустить BIND.

После этого:

```bash
dig @192.168.56.103 zone1.local SOA
```

Это запрос напрямую к master.

И:

```bash
dig @192.168.56.104 zone1.local SOA
```

Это запрос к slave.

Serial должен совпадать.

Проверить конкретную запись:

```bash
dig @192.168.56.103 astra.zone1.local A
dig @192.168.56.104 astra.zone1.local A
```

Оба должны вернуть:

```text
192.168.56.103
```

---

# 30. Делегирование `dns2.zone1.local`

По заданию DNS1 должен делегировать подзону:

```text
dns2.zone1.local
```

на DNS2.

## Что значит делегирование

DNS1 отвечает за:

```text
zone1.local
```

Но мы говорим ему:

> всё, что находится внутри `dns2.zone1.local`, обслуживает другой DNS — DNS2.

Схема:

```text
zone1.local                     DNS1
    |
    +-- astra.zone1.local       DNS1
    |
    +-- dns2.zone1.local ------ DNS2
            |
            +-- www
            +-- test
```

---

# 31. Glue record

В родительской зоне мы записали:

```dns
dns2 IN NS ns.dns2.zone1.local.
```

Но возникает проблема:

> чтобы найти `dns2.zone1.local`, нужно узнать IP `ns.dns2.zone1.local`;  
> а чтобы узнать IP `ns.dns2.zone1.local`, нужно обратиться в `dns2.zone1.local`.

Получается цикл.

Поэтому в родительской зоне добавляются glue records:

```dns
ns.dns2 IN A    192.168.56.104
ns.dns2 IN AAAA fd00:56::104
```

Они сразу дают IP DNS2.

---

# 32. DNS2: создаём делегированную зону

На `astra2` в:

```text
/etc/bind/named.conf.local
```

добавить:

```conf
zone "dns2.zone1.local" {
    type master;
    file "/etc/bind/zones/db.dns2.zone1.local";
};
```

Создать:

```bash
nano /etc/bind/zones/db.dns2.zone1.local
```

Содержимое:

```dns
$TTL 3600

@   IN  SOA ns.dns2.zone1.local. admin.dns2.zone1.local. (
        2026091801
        3600
        900
        604800
        3600
)

    IN  NS  ns.dns2.zone1.local.

ns      IN  A       192.168.56.104
ns      IN  AAAA    fd00:56::104

www     IN  A       192.168.56.104
test    IN  A       192.168.56.104
```

Проверка:

```bash
named-checkzone dns2.zone1.local /etc/bind/zones/db.dns2.zone1.local
```

После reload:

```bash
dig @192.168.56.103 dns2.zone1.local NS
```

и:

```bash
dig @192.168.56.103 www.dns2.zone1.local A
```

---

# 33. Зона `zone2.local`

По заданию здесь роли меняются:

```text
DNS2 -> Master
DNS1 -> Slave
```

Это нужно, чтобы показать, что оба DNS умеют работать как master и как slave.

---

## DNS2

В `/etc/bind/named.conf.local`:

```conf
zone "zone2.local" {
    type master;
    file "/etc/bind/zones/db.zone2.local";

    allow-transfer {
        192.168.56.103;
        fd00:56::103;
    };

    also-notify {
        192.168.56.103;
        fd00:56::103;
    };
};
```

Создать:

```bash
nano /etc/bind/zones/db.zone2.local
```

```dns
$TTL 3600

@   IN  SOA ns2.zone2.local. admin.zone2.local. (
        2026091801
        3600
        900
        604800
        3600
)

    IN  NS  ns1.zone2.local.
    IN  NS  ns2.zone2.local.

ns1     IN  A       192.168.56.103
ns1     IN  AAAA    fd00:56::103

ns2     IN  A       192.168.56.104
ns2     IN  AAAA    fd00:56::104

astra   IN  A       192.168.56.103
astra   IN  AAAA    fd00:56::103

astra2  IN  A       192.168.56.104
astra2  IN  AAAA    fd00:56::104

www     IN  A       192.168.56.104
```

Проверка:

```bash
	named-checkzone zone2.local /etc/bind/zones/db.zone2.local
```

---

## DNS1

Добавить:

```conf
zone "zone2.local" {
    type slave;

    masters {
        192.168.56.104;
        fd00:56::104;
    };

    file "/var/cache/bind/slaves/db.zone2.local";
};
```

Проверка:

```bash
dig @192.168.56.104 zone2.local SOA
dig @192.168.56.103 zone2.local SOA
```

---

# 34. Зона `zone3.local`: почему здесь нужен `rsync`

По заданию:

```text
DNS1 -> Master
DNS2 -> Master
```

Обычный master/slave здесь использовать нельзя.

Нужно дополнительное средство репликации.

Используем:

```text
rsync + SSH + rndc reload
```

Схема:

```text
DNS1
/etc/bind/zones/db.zone3.local
        |
        | rsync через SSH
        v
DNS2
/var/cache/bind/zone3/db.zone3.local
        |
        | rndc reload zone3.local
        v
BIND перечитывает файл
```

Оба BIND считают себя `master`.

Но фактически редактировать файл будем на DNS1.

---

# 35. DNS1: `zone3.local`

На `astra`:

```conf
zone "zone3.local" {
    type master;
    file "/etc/bind/zones/db.zone3.local";
};
```

Создать:

```bash
nano /etc/bind/zones/db.zone3.local
```

```dns
$TTL 3600

@   IN  SOA ns1.zone3.local. admin.zone3.local. (
        2026091801
        3600
        900
        604800
        3600
)

    IN  NS  ns1.zone3.local.
    IN  NS  ns2.zone3.local.

ns1     IN  A       192.168.56.103
ns1     IN  AAAA    fd00:56::103

ns2     IN  A       192.168.56.104
ns2     IN  AAAA    fd00:56::104

www     IN  A       192.168.56.103
test    IN  A       192.168.56.104
```

Проверить:

```bash
named-checkzone zone3.local /etc/bind/zones/db.zone3.local
```

---

# 36. DNS2: пользователь `zonesync`

На `astra2`:

```bash
adduser --disabled-password --gecos "" zonesync
usermod -aG bind zonesync
```

Создать каталог:

```bash
mkdir -p /var/cache/bind/zone3
chown zonesync:bind /var/cache/bind/zone3
chmod 2750 /var/cache/bind/zone3
```

## Зачем отдельный пользователь

Не нужно давать DNS1 полноценный root-доступ к DNS2.

Пользователь:

```text
zonesync
```

существует только для копирования зоны.

Это принцип минимальных привилегий.

## Зачем `2750`

Бит `2` означает `setgid` на каталоге.

Новые файлы внутри будут наследовать группу:

```text
bind
```

Так BIND сможет их читать.

---

# 37. DNS2: объявляем `zone3.local`

На `astra2`:

```conf
zone "zone3.local" {
    type master;
    file "/var/cache/bind/zone3/db.zone3.local";
};
```

Пока файл ещё не скопирован, зона не сможет нормально загрузиться.

Следующий этап создаст файл.

---

# 38. SSH-ключ для `rsync`

На DNS1:

```bash
mkdir -p /root/.ssh

ssh-keygen \
    -t ed25519 \
    -f /root/.ssh/id_ed25519_zone3 \
    -N ""
```

## Почему без пароля

Синхронизация должна выполняться автоматически.

Если ключ требует интерактивного пароля, systemd не сможет выполнить синхронизацию самостоятельно.

Публичный ключ:

```bash
cat /root/.ssh/id_ed25519_zone3.pub
```

Скопировать строку.

На DNS2:

```bash
mkdir -p /home/zonesync/.ssh
nano /home/zonesync/.ssh/authorized_keys
```

Вставить ключ.

Затем:

```bash
chown -R zonesync:zonesync /home/zonesync/.ssh
chmod 700 /home/zonesync/.ssh
chmod 600 /home/zonesync/.ssh/authorized_keys
```

Проверить с DNS1:

```bash
ssh \
    -i /root/.ssh/id_ed25519_zone3 \
    zonesync@192.168.56.104
```

Первый раз подтвердить fingerprint:

```text
yes
```

После входа:

```bash
exit
```

---

# 39. Разрешаем только reload одной зоны

После копирования файла DNS2 должен сказать BIND:

> перечитай `zone3.local`.

Для этого используется:

```bash
rndc reload zone3.local
```

Проверить путь:

```bash
command -v rndc
```

Обычно:

```text
/usr/sbin/rndc
```

На DNS2:

```bash
visudo -f /etc/sudoers.d/zonesync-rndc
```

Добавить:

```text
zonesync ALL=(root) NOPASSWD: /usr/sbin/rndc reload zone3.local
```

Проверить:

```bash
visudo -cf /etc/sudoers.d/zonesync-rndc
```

Мы не выдаём `zonesync` полный `sudo`.

Разрешена только одна конкретная команда.

---

# 40. Скрипт синхронизации

На DNS1:

```bash
nano /usr/local/sbin/sync-zone3.sh
```

Содержимое:

```bash
#!/bin/bash

set -euo pipefail

ZONE="zone3.local"
ZONE_FILE="/etc/bind/zones/db.zone3.local"

REMOTE_HOST="192.168.56.104"
REMOTE_USER="zonesync"
REMOTE_FILE="/var/cache/bind/zone3/db.zone3.local"

SSH_KEY="/root/.ssh/id_ed25519_zone3"

echo "[1/3] Проверяем zone-файл"
named-checkzone "$ZONE" "$ZONE_FILE"

echo "[2/3] Копируем файл на DNS2"
rsync -av \
    --chmod=F640 \
    -e "ssh -i $SSH_KEY" \
    "$ZONE_FILE" \
    "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_FILE}"

echo "[3/3] Просим BIND DNS2 перечитать зону"
ssh \
    -i "$SSH_KEY" \
    "${REMOTE_USER}@${REMOTE_HOST}" \
    "sudo /usr/sbin/rndc reload zone3.local"

echo "zone3.local синхронизирована"
```

Сделать исполняемым:

```bash
chmod +x /usr/local/sbin/sync-zone3.sh
```

Запустить:

```bash
/usr/local/sbin/sync-zone3.sh
```

На DNS2:

```bash
ls -l /var/cache/bind/zone3/
```

Должен появиться:

```text
db.zone3.local
```

---

# 41. Автоматизация `zone3.local`

Хотим, чтобы после изменения master-файла синхронизация выполнялась автоматически.

Создать на DNS1:

```bash
nano /etc/systemd/system/zone3-sync.service
```

```ini
[Unit]
Description=Synchronize zone3.local to DNS2

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/sync-zone3.sh
```

Создать:

```bash
nano /etc/systemd/system/zone3-sync.path
```

```ini
[Unit]
Description=Watch zone3.local for changes

[Path]
PathChanged=/etc/bind/zones/db.zone3.local
Unit=zone3-sync.service

[Install]
WantedBy=multi-user.target
```

Загрузить:

```bash
systemctl daemon-reload
systemctl enable --now zone3-sync.path
```

Проверка:

```bash
systemctl status zone3-sync.path --no-pager
```

---

# 42. Проверяем автоматический `rsync`

На DNS1 открыть:

```bash
nano /etc/bind/zones/db.zone3.local
```

Увеличить serial:

```text
2026091801
```

на:

```text
2026091802
```

Добавить:

```dns
demo IN A 192.168.56.103
```

Сохранить.

Проверить:

```bash
journalctl -u zone3-sync.service -n 30 --no-pager
```

На DNS2:

```bash
dig @192.168.56.104 demo.zone3.local A
```

Ожидаем:

```text
192.168.56.103
```

Таким образом мы доказываем именно требование:

```text
репликация zone3.local дополнительными средствами
+
автоматическое перечитывание DNS-сервером
```

---

# 43. IDN-зона `магистратура.сибгути.local`

DNS исторически использует ASCII.

Поэтому кириллица кодируется через IDNA/Punycode.

```text
магистратура
```

становится:

```text
xn--80aaakzv5abgkcm
```

А:

```text
сибгути
```

становится:

```text
xn--90aenc5bjg
```

Итог:

```text
магистратура.сибгути.local
```

представляется в BIND как:

```text
xn--80aaakzv5abgkcm.xn--90aenc5bjg.local
```

По заданию:

```text
DNS2 -> Master
DNS1 -> Slave
```

---

# 44. DNS2: master IDN-зоны

На DNS2:

```conf
zone "xn--80aaakzv5abgkcm.xn--90aenc5bjg.local" {
    type master;
    file "/etc/bind/zones/db.magistratura";

    allow-transfer {
        192.168.56.103;
        fd00:56::103;
    };

    also-notify {
        192.168.56.103;
        fd00:56::103;
    };
};
```

Создать:

```bash
nano /etc/bind/zones/db.magistratura
```

```dns
$TTL 3600

@   IN  SOA ns2.xn--80aaakzv5abgkcm.xn--90aenc5bjg.local. admin.xn--80aaakzv5abgkcm.xn--90aenc5bjg.local. (
        2026091801
        3600
        900
        604800
        3600
)

    IN  NS  ns1.xn--80aaakzv5abgkcm.xn--90aenc5bjg.local.
    IN  NS  ns2.xn--80aaakzv5abgkcm.xn--90aenc5bjg.local.

ns1     IN  A       192.168.56.103
ns1     IN  AAAA    fd00:56::103

ns2     IN  A       192.168.56.104
ns2     IN  AAAA    fd00:56::104

www     IN  A       192.168.56.104
```

Проверить:

```bash
named-checkzone \
    xn--80aaakzv5abgkcm.xn--90aenc5bjg.local \
    /etc/bind/zones/db.magistratura
```

---

# 45. DNS1: slave IDN-зоны

На DNS1:

```conf
zone "xn--80aaakzv5abgkcm.xn--90aenc5bjg.local" {
    type slave;

    masters {
        192.168.56.104;
        fd00:56::104;
    };

    file "/var/cache/bind/slaves/db.magistratura";
};
```

Проверить:

```bash
dig \
    @192.168.56.104 \
    xn--80aaakzv5abgkcm.xn--90aenc5bjg.local \
    SOA
```

и:

```bash
dig \
    @192.168.56.103 \
    xn--80aaakzv5abgkcm.xn--90aenc5bjg.local \
    SOA
```

---

# 46. Reverse DNS IPv4

Обычно DNS делает:

```text
имя -> IP
```

Например:

```text
astra.zone1.local -> 192.168.56.103
```

Reverse DNS делает наоборот:

```text
IP -> имя
```

Для IPv4 используется:

```text
in-addr.arpa
```

Наша сеть:

```text
192.168.56.0/24
```

обратная зона:

```text
56.168.192.in-addr.arpa
```

Октеты записаны в обратном порядке.

По заданию:

```text
DNS1 -> Master
DNS2 -> Slave
```

---

# 47. DNS1: IPv4 reverse master

На DNS1:

```conf
zone "56.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.56";

    allow-transfer {
        192.168.56.104;
        fd00:56::104;
    };

    also-notify {
        192.168.56.104;
        fd00:56::104;
    };
};
```

Создать:

```bash
nano /etc/bind/zones/db.192.168.56
```

```dns
$TTL 3600

@   IN  SOA ns1.zone1.local. admin.zone1.local. (
        2026091801
        3600
        900
        604800
        3600
)

    IN  NS  ns1.zone1.local.
    IN  NS  ns2.zone1.local.

103 IN PTR astra.zone1.local.
104 IN PTR astra2.zone1.local.
```

## Почему только `103` и `104`

Сама зона уже означает:

```text
192.168.56
```

Поэтому:

```text
103
```

становится:

```text
192.168.56.103
```

---

# 48. DNS2: IPv4 reverse slave

На DNS2:

```conf
zone "56.168.192.in-addr.arpa" {
    type slave;

    masters {
        192.168.56.103;
        fd00:56::103;
    };

    file "/var/cache/bind/slaves/db.192.168.56";
};
```

Проверить:

```bash
dig @192.168.56.103 -x 192.168.56.103
```

Ожидаем:

```text
astra.zone1.local.
```

---

# 49. Reverse DNS IPv6

IPv6 reverse использует:

```text
ip6.arpa
```

Но там разворачиваются не байты и не группы по `:`, а отдельные hexadecimal digits — nibble.

Сеть:

```text
fd00:56::/64
```

в полном виде:

```text
fd00:0056:0000:0000::/64
```

Reverse zone:

```text
0.0.0.0.0.0.0.0.6.5.0.0.0.0.d.f.ip6.arpa
```

По заданию:

```text
DNS2 -> Master
DNS1 -> Slave
```

---

# 50. DNS2: IPv6 reverse master

На DNS2:

```conf
zone "0.0.0.0.0.0.0.0.6.5.0.0.0.0.d.f.ip6.arpa" {
    type master;
    file "/etc/bind/zones/db.fd00-56";

    allow-transfer {
        192.168.56.103;
        fd00:56::103;
    };

    also-notify {
        192.168.56.103;
        fd00:56::103;
    };
};
```

Создать:

```bash
nano /etc/bind/zones/db.fd00-56
```

```dns
$TTL 3600

@   IN  SOA ns2.zone2.local. admin.zone2.local. (
        2026091801
        3600
        900
        604800
        3600
)

    IN  NS  ns1.zone2.local.
    IN  NS  ns2.zone2.local.

3.0.1.0.0.0.0.0.0.0.0.0.0.0.0.0 IN PTR astra.zone1.local.
4.0.1.0.0.0.0.0.0.0.0.0.0.0.0.0 IN PTR astra2.zone2.local.
```

Проверить:

```bash
named-checkzone \
    0.0.0.0.0.0.0.0.6.5.0.0.0.0.d.f.ip6.arpa \
    /etc/bind/zones/db.fd00-56
```

---

# 51. DNS1: IPv6 reverse slave

На DNS1:

```conf
zone "0.0.0.0.0.0.0.0.6.5.0.0.0.0.d.f.ip6.arpa" {
    type slave;

    masters {
        192.168.56.104;
        fd00:56::104;
    };

    file "/var/cache/bind/slaves/db.fd00-56";
};
```

Проверить:

```bash
dig @192.168.56.104 -x fd00:56::103
dig @192.168.56.103 -x fd00:56::104
```

---

# 52. Финальная проверка конфигурации BIND

На обеих VM:

```bash
named-checkconf
```

Полезно также:

```bash
named-checkconf -z
```

Опция `-z` заставляет BIND проверить загрузку master-зон.

После успешной проверки:

```bash
systemctl restart bind9
```

или, если service называется `named`:

```bash
systemctl restart named
```

Проверить:

```bash
systemctl --no-pager --full status bind9
```

---

# 53. Проверяем порт DNS

На обеих VM:

```bash
ss -lntup | grep ':53'
```

DNS использует два транспорта:

```text
UDP 53
TCP 53
```

UDP обычно используется для обычных запросов.

TCP нужен в том числе для zone transfer.

---

# 54. Проверяем slave-файлы

На DNS1:

```bash
ls -lah /var/cache/bind/slaves/
```

Ожидаем копии зон, где DNS1 является slave:

```text
db.zone2.local
db.magistratura
db.fd00-56
```

На DNS2:

```bash
ls -lah /var/cache/bind/slaves/
```

Ожидаем:

```text
db.zone1.local
db.192.168.56
```

Не редактируй их вручную.

---

# 55. Проверяем AXFR

С DNS2 запросить master-зону DNS1:

```bash
dig @192.168.56.103 zone1.local AXFR
```

Должны прийти записи зоны.

С DNS1 проверить `zone2.local`:

```bash
dig @192.168.56.104 zone2.local AXFR
```

Для:

```text
zone3.local
```

AXFR между узлами специально не используем, потому что требование реализовано через `rsync`.

---

# 56. Проверяем рекурсию DNS1

```bash
dig @192.168.56.103 debian.org A
```

Нормально:

```text
status: NOERROR
```

и есть:

```text
ANSWER SECTION
```

Таким образом путь выглядит:

```text
клиент
  |
  v
DNS1
  |
  v
forwarder
  |
  v
ответ
```

---

# 57. Проверяем DNS2 без рекурсии

```bash
dig @192.168.56.104 debian.org A
```

DNS2 не обязан рекурсивно разрешать внешний домен, потому что:

```conf
recursion no;
```

Но он обязан отвечать за свои authoritative-зоны.

Например:

```bash
dig @192.168.56.104 zone2.local SOA
```

---

# 58. Проверяем логирование

На DNS1:

```bash
tail -f /var/log/named/query.log
```

В другом SSH-окне:

```bash
dig @192.168.56.103 debian.org
```

В логе должен появиться запрос.

Общий debug-log:

```bash
tail -f /var/log/named/named.log
```

В нём полезно искать:

- загрузку зон;
- NOTIFY;
- transfer;
- ошибки конфигурации.

---

# 59. Почему `.local` иногда ведёт себя необычно

`.local` исторически используется mDNS/Avahi.

Но в задании прямо заданы:

```text
zone1.local
zone2.local
zone3.local
```

Поэтому их не меняем.

При проверках лучше явно указывать DNS-сервер:

```bash
dig @192.168.56.103 astra.zone1.local
```

а не просто:

```bash
host astra.zone1.local
```

Так мы точно проверяем BIND, а не mDNS или системный resolver.

---

# ЧАСТЬ III. LDAP

# 60. Что мы строим

Используем следующую трактовку задания:

```text
astra  = LDAP server
astra2 = LDAP client
```

То есть:

```text
информация о пользователе хранится на astra
```

а:

```text
войти этим пользователем можно на astra2
```

Схема:

```text
          LDAP
astra ------------- astra2
server               client
                       |
                       +-- NSS ищет пользователя
                       |
                       +-- PAM проверяет пароль
                       |
                       +-- SSH создаёт login session
```

---

# 61. Важное противоречие исходного задания

В первой строке LDAP-части сервер требуется настроить на узле 1.

В следующей строке написано, что узел 2 должен авторизовывать пользователей LDAP-сервера, работающего на узле 2.

Это противоречит предыдущему пункту.

Логичная архитектура:

```text
узел 1 = LDAP server
узел 2 = LDAP client
```

именно её используем ниже.

---

# 62. Что такое LDAP

LDAP — протокол доступа к каталогу.

Каталог похож на древовидную базу данных.

Мы создадим дерево:

```text
dc=lab,dc=local
│
├── ou=People
│   │
│   └── uid=student
│
└── ou=Groups
    │
    └── cn=ldapusers
```

---

# 63. Что такое DN

DN:

```text
Distinguished Name
```

Это полный путь объекта в LDAP.

Например:

```text
uid=student,ou=People,dc=lab,dc=local
```

Разбирается справа налево:

```text
dc=local
└── dc=lab
    └── ou=People
        └── uid=student
```

---

# 64. Устанавливаем LDAP server на `astra`

```bash
apt update
apt install -y slapd ldap-utils
```

Затем:

```bash
dpkg-reconfigure slapd
```

Использовать:

```text
Omit OpenLDAP server configuration? -> No

DNS domain name:
lab.local

Organization:
Lab

Administrator password:
выбрать пароль

Database backend:
MDB

Remove database when slapd is purged?
No

Move old database?
Yes
```

После этого base DN:

```text
dc=lab,dc=local
```

Administrator:

```text
cn=admin,dc=lab,dc=local
```

---

# 65. Почему `lab.local`

LDAP DN строится из доменного имени.

```text
lab.local
```

превращается в:

```text
dc=lab,dc=local
```

Это отдельная логическая LDAP-структура.

Она не обязана совпадать с `zone1.local`.

---

# 66. Проверяем `slapd`

```bash
systemctl status slapd --no-pager
```

Порт LDAP:

```bash
ss -lntp | grep ':389'
```

LDAP без TLS использует:

```text
TCP 389
```

Для изолированной учебной LAN этого достаточно.

В реальной инфраструктуре пароли LDAP не следует передавать в открытом виде — нужен TLS/LDAPS или StartTLS.

---

# 67. Проверяем каталог

На `astra`:

```bash
ldapsearch \
    -x \
    -LLL \
    -H ldap://127.0.0.1 \
    -b dc=lab,dc=local
```

Параметры:

```text
-x
```

simple authentication вместо SASL.

```text
-LLL
```

короткий чистый вывод.

```text
-H
```

URI LDAP server.

```text
-b
```

точка начала поиска.

---

# 68. Создаём организационные подразделения

Создать:

```bash
nano ~/base.ldif
```

Содержимое:

```ldif
dn: ou=People,dc=lab,dc=local
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=lab,dc=local
objectClass: organizationalUnit
ou: Groups
```

Добавить:

```bash
ldapadd \
    -x \
    -D "cn=admin,dc=lab,dc=local" \
    -W \
    -H ldap://127.0.0.1 \
    -f ~/base.ldif
```

## Что такое LDIF

LDIF — текстовый формат представления LDAP-записей.

Мы не редактируем LDAP как обычный файл базы.

Мы описываем объект в `.ldif` и передаём его LDAP-серверу.

---

# 69. Выбираем UID/GID

Перед созданием пользователя убедиться, что `10000` свободен.

На `astra2`:

```bash
getent passwd 10000
getent group 10000
```

Если вывода нет — значения свободны.

Будем использовать:

```text
uidNumber = 10000
gidNumber = 10000
```

---

# 70. Создаём LDAP-группу

На `astra`:

```bash
nano ~/group.ldif
```

```ldif
dn: cn=ldapusers,ou=Groups,dc=lab,dc=local
objectClass: top
objectClass: posixGroup
cn: ldapusers
gidNumber: 10000
memberUid: student
```

Добавить:

```bash
ldapadd \
    -x \
    -D "cn=admin,dc=lab,dc=local" \
    -W \
    -H ldap://127.0.0.1 \
    -f ~/group.ldif
```

---

# 71. Генерируем пароль пользователя

```bash
slappasswd
```

Ввести пароль.

Получится:

```text
{SSHA}...
```

Это salted hash пароля.

Пароль в открытом виде в LDIF не сохраняем.

---

# 72. Создаём LDAP-пользователя

```bash
nano ~/student.ldif
```

Содержимое:

```ldif
dn: uid=student,ou=People,dc=lab,dc=local
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: student
cn: Student
sn: User
uidNumber: 10000
gidNumber: 10000
homeDirectory: /home/student
loginShell: /bin/bash
userPassword: {SSHA}ВСТАВИТЬ_ХЭШ_ИЗ_SLAPPASSWD
```

Добавить:

```bash
ldapadd \
    -x \
    -D "cn=admin,dc=lab,dc=local" \
    -W \
    -H ldap://127.0.0.1 \
    -f ~/student.ldif
```

---

# 73. Почему LDAP-пользователю нужны Linux-атрибуты

Чтобы Linux мог воспринимать LDAP-запись как пользователя, нужны:

```text
uidNumber
gidNumber
homeDirectory
loginShell
```

Это аналоги полей из:

```text
/etc/passwd
```

Например локальный пользователь выглядит примерно:

```text
student:x:10000:10000:Student:/home/student:/bin/bash
```

Мы храним аналогичные сведения не локально, а в LDAP.

---

# 74. Проверяем пользователя на сервере

```bash
ldapsearch \
    -x \
    -LLL \
    -H ldap://192.168.56.103 \
    -b "dc=lab,dc=local" \
    "(uid=student)"
```

Должны появиться:

```text
uid: student
uidNumber: 10000
gidNumber: 10000
homeDirectory: /home/student
loginShell: /bin/bash
```

---

# 75. Что нужно клиенту Linux для LDAP login

Одного `ldapsearch` недостаточно.

Нужно подключить LDAP к двум подсистемам Linux.

## NSS

```text
Name Service Switch
```

Отвечает на вопрос:

> существует ли пользователь `student` и какие у него UID, GID, home, shell?

Команда:

```bash
getent passwd student
```

использует NSS.

## PAM

```text
Pluggable Authentication Modules
```

Отвечает на вопрос:

> правильный ли пароль ввёл пользователь?

SSH, `su`, `login` используют PAM.

То есть:

```text
NSS -> кто такой student?
PAM -> верный ли пароль student?
```

Нужны оба.

---

# 76. Устанавливаем LDAP client на `astra2`

```bash
apt update

apt install -y \
    libnss-ldapd \
    libpam-ldapd \
    nslcd \
    ldap-utils
```

Если установщик спрашивает LDAP URI:

```text
ldap://192.168.56.103/
```

Search base:

```text
dc=lab,dc=local
```

Для NSS выбрать:

```text
passwd
group
shadow
```

---

# 77. Что такое `nslcd`

`nslcd` — локальный демон на `astra2`.

Схема:

```text
Linux NSS/PAM
    |
    v
  nslcd
    |
    v
LDAP server 192.168.56.103
```

Он выполняет LDAP-запросы от имени NSS/PAM.

---

# 78. Настраиваем `nslcd`

На `astra2`:

```bash
nano /etc/nslcd.conf
```

Ключевые параметры:

```conf
uid nslcd
gid nslcd

uri ldap://192.168.56.103/

base dc=lab,dc=local
```

После изменения:

```bash
systemctl restart nslcd
systemctl status nslcd --no-pager
```

---

# 79. Проверяем LDAP с клиента до настройки NSS

На `astra2`:

```bash
ldapsearch \
    -x \
    -LLL \
    -H ldap://192.168.56.103 \
    -b dc=lab,dc=local \
    "(uid=student)"
```

Если это работает, значит уже исправны:

```text
astra2
  |
  | TCP 389
  v
astra LDAP
```

Если не работает, бессмысленно пока диагностировать PAM.

---

# 80. Настраиваем NSS

Открыть:

```bash
nano /etc/nsswitch.conf
```

Найти:

```text
passwd:
group:
shadow:
```

Если используется `compat`, сделать:

```text
passwd:         compat ldap
group:          compat ldap
shadow:         compat ldap
```

Если там `files` / `systemd`, можно получить, например:

```text
passwd:         files systemd ldap
group:          files systemd ldap
shadow:         files ldap
```

Главное — наличие:

```text
ldap
```

---

# 81. Проверяем NSS

```bash
getent passwd student
```

Ожидаем примерно:

```text
student:x:10000:10000:Student:/home/student:/bin/bash
```

Группа:

```bash
getent group ldapusers
```

Если `ldapsearch` работает, а `getent` нет:

```bash
systemctl status nslcd
journalctl -u nslcd -n 100 --no-pager
```

---

# 82. Настраиваем PAM

На `astra2`:

```bash
pam-auth-update
```

Включить LDAP authentication.

Также желательно включить:

```text
Create home directory on login
```

---

# 83. Зачем нужен `pam_mkhomedir`

LDAP сообщает:

```text
homeDirectory: /home/student
```

Но LDAP не создаёт этот каталог физически.

Если пользователь впервые входит на `astra2`, Linux должен создать:

```text
/home/student
```

Для этого используется:

```text
pam_mkhomedir.so
```

Проверить:

```bash
grep -R "pam_mkhomedir" /etc/pam.d/
```

Если не настроено, в:

```bash
nano /etc/pam.d/common-session
```

добавить:

```text
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

---

# 84. Проверяем LDAP login локально

На `astra2`:

```bash
su - student
```

Ввести LDAP-пароль.

После входа:

```bash
whoami
```

Ожидаем:

```text
student
```

Проверить:

```bash
id
pwd
```

Ожидаем home:

```text
/home/student
```

---

# 85. Проверяем LDAP login через SSH

На `astra2`:

```bash
sshd -T | grep passwordauthentication
```

Нужно:

```text
passwordauthentication yes
```

Затем с первой VM или основного компьютера:

```bash
ssh student@192.168.56.104
```

Пароль вводится LDAP-пользователя.

После входа:

```bash
whoami
```

Ожидаем:

```text
student
```

Это итоговое доказательство работы LDAP-аутентификации.

---

# ЧАСТЬ IV. ДИАГНОСТИКА

# 86. BIND не стартует

Сначала:

```bash
named-checkconf
```

Затем:

```bash
named-checkconf -z
```

И журнал:

```bash
journalctl -u bind9 -n 100 --no-pager
```

или:

```bash
journalctl -u named -n 100 --no-pager
```

---

# 87. Ошибка в master-зоне

Проверять конкретный файл:

```bash
named-checkzone zone1.local /etc/bind/zones/db.zone1.local
```

Не пытайся угадывать ошибку глазами, пока `named-checkzone` может показать её сам.

---

# 88. Slave не получает зону

Проверить:

```bash
ss -lntup | grep ':53'
```

Затем zone transfer вручную:

```bash
dig @192.168.56.103 zone1.local AXFR
```

Проверить firewall:

```bash
nft list ruleset
```

AXFR требует TCP/53.

---

# 89. Изменение master не приходит на slave

Чаще всего забыли увеличить:

```text
SOA serial
```

Например было:

```text
2026091801
```

стало:

```text
2026091802
```

Затем:

```bash
rndc reload zone1.local
```

---

# 90. `rsync zone3` не работает

Проверить SSH:

```bash
ssh \
    -i /root/.ssh/id_ed25519_zone3 \
    zonesync@192.168.56.104
```

Проверить скрипт вручную:

```bash
/usr/local/sbin/sync-zone3.sh
```

Проверить systemd:

```bash
systemctl status zone3-sync.path
journalctl -u zone3-sync.service -n 50 --no-pager
```

---

# 91. LDAP server недоступен

С `astra2`:

```bash
ping -c 3 192.168.56.103
```

Потом:

```bash
nc -vz 192.168.56.103 389
```

Если `nc` нет:

```bash
apt install -y netcat-openbsd
```

---

# 92. LDAP находится через `ldapsearch`, но не через `getent`

Это почти наверняка слой:

```text
NSS / nslcd
```

Проверить:

```bash
systemctl status nslcd
grep -E '^(passwd|group|shadow):' /etc/nsswitch.conf
journalctl -u nslcd -n 100 --no-pager
```

---

# 93. `getent` работает, но пароль не принимается

Тогда NSS уже исправен.

Проблема обычно в:

```text
PAM
```

Проверить:

```bash
pam-auth-update
```

и:

```bash
grep -R ldap /etc/pam.d/
```

---

# 94. Пользователь входит, но `/home/student` отсутствует

Проблема:

```text
pam_mkhomedir
```

Проверить:

```bash
grep -R pam_mkhomedir /etc/pam.d/
```

---

# ЧАСТЬ V. ЧТО ПОКАЗАТЬ НА СДАЧЕ

# 95. Сеть

На `astra`:

```bash
hostname
ip -br a
ip route
```

Должно быть видно:

```text
eth0 -> 192.168.56.103
        fd00:56::103

eth1 -> NAT / Internet
```

На `astra2`:

```text
eth0 -> 192.168.56.104
        fd00:56::104

eth1 -> NAT / Internet
```

---

# 96. IPv4 и IPv6 между VM

```bash
ping -c 3 192.168.56.104
ping -6 -c 3 fd00:56::104
```

На второй в обратную сторону:

```bash
ping -c 3 192.168.56.103
ping -6 -c 3 fd00:56::103
```

---

# 97. DNS1 и DNS2 запущены

```bash
systemctl status bind9 --no-pager
```

или:

```bash
systemctl status named --no-pager
```

---

# 98. Master/Slave `zone1`

```bash
dig @192.168.56.103 zone1.local SOA
dig @192.168.56.104 zone1.local SOA
```

Объяснение преподавателю:

```text
На DNS1 zone1.local является master.
DNS2 является slave.
Одинаковый SOA serial показывает, что репликация прошла.
```

---

# 99. Master/Slave `zone2`

```bash
dig @192.168.56.104 zone2.local SOA
dig @192.168.56.103 zone2.local SOA
```

Здесь master уже DNS2.

---

# 100. `zone3.local` через `rsync`

```bash
systemctl status zone3-sync.path
journalctl -u zone3-sync.service -n 30 --no-pager
```

И:

```bash
dig @192.168.56.104 demo.zone3.local A
```

Объяснение:

```text
Здесь оба DNS имеют тип master.
Штатный AXFR/IXFR не используется.
Zone-файл копируется rsync по SSH.
После копирования выполняется rndc reload.
```

---

# 101. Делегирование

```bash
dig @192.168.56.103 dns2.zone1.local NS
dig @192.168.56.103 www.dns2.zone1.local A
```

Объяснение:

```text
DNS1 authoritative для zone1.local,
но поддомен dns2.zone1.local делегирован DNS2.
Glue record сообщает адрес DNS2.
```

---

# 102. IPv4 reverse

```bash
dig @192.168.56.103 -x 192.168.56.103
```

Ожидаем:

```text
astra.zone1.local.
```

---

# 103. IPv6 reverse

```bash
dig @192.168.56.104 -x fd00:56::104
```

---

# 104. Forwarders / recursion

```bash
dig @192.168.56.103 debian.org A
```

Объяснение:

```text
DNS1 не authoritative для debian.org.
Он выполняет recursive lookup с использованием forwarders.
```

---

# 105. DNS logging

```bash
tail -n 30 /var/log/named/query.log
```

---

# 106. LDAP server

На `astra`:

```bash
systemctl status slapd --no-pager
```

Пользователь:

```bash
ldapsearch \
    -x \
    -LLL \
    -H ldap://192.168.56.103 \
    -b dc=lab,dc=local \
    "(uid=student)"
```

---

# 107. LDAP client

На `astra2`:

```bash
getent passwd student
```

Это доказывает работу NSS.

Затем:

```bash
su - student
```

или:

```bash
ssh student@192.168.56.104
```

Это доказывает работу PAM authentication.

---

# 108. Минимальный чек-лист готовности

## Сеть

- [ ] `astra eth0 = 192.168.56.103/24`
- [ ] `astra2 eth0 = 192.168.56.104/24`
- [ ] `astra eth0 = fd00:56::103/64`
- [ ] `astra2 eth0 = fd00:56::104/64`
- [ ] на обеих `eth1` работает через VirtualBox NAT
- [ ] на обеих default route идёт через `eth1`
- [ ] IPv4 ping между VM работает
- [ ] IPv6 ping между VM работает
- [ ] Интернет работает

## DNS

- [ ] BIND работает на обеих VM
- [ ] подробное логирование включено
- [ ] `zone1.local`: DNS1 master, DNS2 slave
- [ ] `zone2.local`: DNS2 master, DNS1 slave
- [ ] `zone3.local`: master/master + rsync
- [ ] IDN-зона: DNS2 master, DNS1 slave
- [ ] IPv4 reverse: DNS1 master, DNS2 slave
- [ ] IPv6 reverse: DNS2 master, DNS1 slave
- [ ] `dns2.zone1.local` делегирован DNS2
- [ ] DNS1 выполняет recursion через forwarders
- [ ] AXFR работает только там, где нужен
- [ ] `zone3.local` обновляется без AXFR

## LDAP

- [ ] `slapd` работает на `astra`
- [ ] создан `dc=lab,dc=local`
- [ ] создан `ou=People`
- [ ] создан `ou=Groups`
- [ ] создан `student`
- [ ] создана группа `ldapusers`
- [ ] `ldapsearch` работает с `astra2`
- [ ] `getent passwd student` работает
- [ ] PAM принимает LDAP-пароль
- [ ] `/home/student` создаётся
- [ ] `ssh student@192.168.56.104` работает

---

# 109. Короткая логика всей работы

Если на защите попросят объяснить систему целиком:

```text
1. Есть две Debian VM в одной IPv4/IPv6 LAN.

2. У каждой два интерфейса:
   eth0 — лабораторная LAN;
   eth1 — VirtualBox NAT для Интернета.

3. На обеих работает BIND9.

4. Часть DNS-зон хранится на DNS1 как master
   и передаётся DNS2 стандартным zone transfer.

5. Для других зон master — DNS2,
   а DNS1 получает slave-копию.

6. zone3.local специально синхронизируется rsync,
   после чего BIND DNS2 автоматически перечитывает файл.

7. DNS1 делегирует подзону dns2.zone1.local DNS2.

8. DNS1 работает как recursive resolver через forwarders.

9. Настроены прямые A/AAAA и обратные PTR-зоны IPv4/IPv6.

10. На astra работает LDAP directory.

11. Пользователь student хранится не локально на astra2,
    а в LDAP на astra.

12. NSS позволяет astra2 увидеть LDAP-пользователя.

13. PAM позволяет проверить его LDAP-пароль.

14. Поэтому student может авторизоваться на astra2,
    в том числе по SSH.
```

Это и есть смысл всей лабораторной: построить маленькую распределённую инфраструктуру, где сетевые сервисы на двух узлах взаимодействуют друг с другом, реплицируют данные и предоставляют централизованную аутентификацию.
