# Сеть Linux

[[OS/Linux/Linux|Карта Linux]]

## Сетевой мониторинг

Roadmap рекомендует:

```text
ss
netstat
iftop
nload
```

### `ss`

Показать сетевые соединения и сокеты.

```bash
ss -tulpn
```

Очень полезная команда при поиске:

```text
какой сервис слушает порт?
```

## Linux Networking

Эта тема формально пересекается со следующим блоком roadmap — Networks, но базу надо начать здесь.

### Нужно понимать

- IP address;
- subnet;
- default gateway;
- DNS;
- TCP / UDP;
- port;
- listening socket;
- localhost / loopback;
- routing table.

### Команды

```bash
ip addr
ip link
ip route
ss -tulpn
ping
curl
wget
traceroute
mtr
nc
nmap
resolvectl
nslookup
dig
tcpdump
```

### Практика

- [ ] Найти свой IP.
- [ ] Найти default gateway.
- [ ] Посмотреть routing table.
- [ ] Найти все listening TCP ports.
- [ ] Запустить локальный HTTP-сервер на 8080.
- [ ] Проверить его через `curl`.
- [ ] Найти процесс, который слушает порт 8080.
- [ ] Проверить DNS через `dig`.
- [ ] Сделать простой `tcpdump` HTTP/DNS-трафика.

## Связанные темы

- [[OS/Linux/Administration/Firewall]]
- [[OS/Linux/Administration/SSH и передача файлов]]
- [[OS/Linux/Administration/Диагностика проблем]]
