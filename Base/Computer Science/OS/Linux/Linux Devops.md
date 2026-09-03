# 1. Терминал, файловая система и базовые команды

## Нужно понимать

- абсолютный и относительный путь;
- домашняя директория `~`;
- текущая директория `.`;
- родительская директория `..`;
- скрытые файлы;
- inode в общих чертах;
- hard link и symbolic link;
- стандартную структуру Linux:
    - `/etc`
    - `/var`
    - `/home`
    - `/tmp`
    - `/usr`
    - `/opt`
    - `/proc`
    - `/sys`
    - `/dev`

## Команды — обязательный минимум

```bash
pwd
ls
ls -la
cd
mkdir
rmdir
touch
cp
mv
rm
rm -r
cat
less
head
tail
file
stat
find
which
whereis
ln
ln -s
tree
```

## Практика

- [ ] Создать дерево директорий из 3–4 уровней.
- [ ] Создать несколько файлов.
- [ ] Скопировать и переместить их.
- [ ] Найти файл через `find`.
- [ ] Создать symbolic link.
- [ ] Создать hard link.
- [ ] Через `stat` сравнить inode исходного файла, hard link и symlink.
- [ ] Найти расположение бинарника `bash`, `ssh`, `curl`.

### Мини-лаба

Создай:

```text
linux-lab/
├── configs/
├── logs/
└── data/
```

Положи в `logs/` 10 файлов разных размеров и дат изменения.

Найди:

- все `.log` файлы;
- файлы больше заданного размера;
- файлы, изменённые за последние сутки.

---

# 2. stdin, stdout, stderr, pipes

Это одна из самых полезных тем для реальной работы.

## Нужно понимать

У процесса есть стандартные потоки:

```text
0 — stdin
1 — stdout
2 — stderr
```

## Освоить

```bash
>
>>
2>
2>&1
|
tee
xargs
```

## Примеры

```bash
echo "hello" > file.txt

echo "second line" >> file.txt

ls /root 2> errors.log

command > output.log 2>&1

cat /etc/passwd | grep bash

find . -name '*.log' | xargs wc -l
```

## Практика

- [ ] Перенаправить stdout команды в файл.
- [ ] Перенаправить stderr отдельно.
- [ ] Сохранить stdout и stderr в один файл.
- [ ] Построить pipeline минимум из трёх команд.
- [ ] Использовать `tee`, чтобы одновременно видеть вывод и сохранять его в файл.

---

# 3. Работа с текстом

DevOps постоянно работает с конфигами и логами.

## Обязательные инструменты

```bash
grep
awk
sed
cut
sort
uniq
wc
tr
xargs
```

## Что уметь

- найти строку по шаблону;
- исключить строки;
- считать количество совпадений;
- выбрать колонку;
- сортировать;
- удалять дубликаты;
- заменять текст;
- обрабатывать вывод другой команды.

## Практика

Скопируй `/etc/passwd` в свою директорию и ответь командами:

- [ ] сколько строк в файле;
- [ ] сколько уникальных shell используется;
- [ ] какие пользователи используют `/bin/bash`;
- [ ] вывести только имена пользователей;
- [ ] отсортировать имена;
- [ ] заменить один текстовый фрагмент через `sed` в копии файла.

### Лаба по логам

Создай `app.log` примерно такого вида:

```text
2026-09-03T10:00:01 INFO user=42 request=/api/users status=200 duration=31ms
2026-09-03T10:00:03 ERROR user=17 request=/api/orders status=500 duration=814ms
2026-09-03T10:00:04 INFO user=11 request=/health status=200 duration=2ms
```

Научись получать:

- только `ERROR`;
- все ответы `500`;
- количество запросов по каждому status code;
- самые долгие запросы.

---

# 4. Пользователи, группы и права

## Нужно понимать

- user;
- group;
- UID / GID;
- root;
- sudo;
- owner / group / others;
- `r`, `w`, `x`;
- octal permissions;
- почему права директории отличаются по смыслу от прав файла.

## Команды

```bash
id
whoami
who
w
sudo
su
useradd
usermod
userdel
groupadd
passwd
chmod
chown
chgrp
umask
```

## Обязательно уметь объяснить

```text
644
600
700
755
775
```

Например:

```bash
chmod 755 script.sh
chmod u+x script.sh
chown user:group file
```

## Практика

- [ ] Создать тестового пользователя.
- [ ] Создать группу.
- [ ] Добавить пользователя в группу.
- [ ] Создать файл, доступный только владельцу.
- [ ] Создать директорию, доступную владельцу и группе.
- [ ] Объяснить, что означает `755` для директории.
- [ ] Проверить текущий `umask`.

---

# 5. Процессы и сигналы

Один из важнейших разделов Linux для DevOps.

## Нужно понимать

- процесс;
- PID;
- PPID;
- parent / child process;
- foreground / background process;
- zombie process;
- жизненный цикл процесса;
- `fork()` → `exec()` → `wait()` на концептуальном уровне.

## Команды

```bash
ps aux
ps -ef
pgrep
pkill
kill
killall
top
htop
jobs
bg
fg
nohup
nice
renice
```

## Основные сигналы

```text
SIGTERM  15 — корректное завершение
SIGKILL   9 — принудительное завершение
SIGHUP    1 — hangup / часто используется для reload
SIGINT    2 — Ctrl+C
SIGSTOP  19 — остановить процесс
SIGCONT  18 — продолжить процесс
```

Важно:

```bash
kill PID
```

обычно отправляет `SIGTERM`, а не «убивает процесс мгновенно».

## Состояния процессов

```text
R — Running
S — Sleeping
D — Uninterruptible sleep
Z — Zombie
T — Stopped
```

## Практика

- [ ] Запустить `sleep 1000`.
- [ ] Найти его PID.
- [ ] Завершить через SIGTERM.
- [ ] Запустить ещё раз и завершить SIGKILL.
- [ ] Запустить процесс в фоне через `&`.
- [ ] Использовать `jobs`, `fg`, `bg`.
- [ ] Запустить процесс через `nohup`.
- [ ] Посмотреть дерево процессов.

### Мини-эксперимент

```bash
sleep 1000 &
PID=$!
cat /proc/$PID/status
cat /proc/$PID/cmdline
ls -l /proc/$PID/fd
```

Нужно понять, что `/proc` — это псевдофайловая система с информацией о ядре и процессах.

---

# 6. systemd и управление сервисами

## Нужно понимать

- что такое init system;
- PID 1;
- service/unit;
- target;
- dependency;
- restart policy;
- journal.

## Команды

```bash
systemctl status SERVICE
systemctl start SERVICE
systemctl stop SERVICE
systemctl restart SERVICE
systemctl reload SERVICE
systemctl enable SERVICE
systemctl disable SERVICE
systemctl daemon-reload
systemctl list-units
journalctl
journalctl -u SERVICE
journalctl -f
```

## Практика — свой systemd unit

Создай простейший HTTP-сервер.

Например:

```bash
python -m http.server 8080
```

Оформи его как systemd service.

Пример структуры unit-файла:

```ini
[Unit]
Description=DevOps Linux Lab HTTP Server
After=network.target

[Service]
Type=simple
User=YOUR_USER
WorkingDirectory=/home/YOUR_USER/linux-lab
ExecStart=/usr/bin/python -m http.server 8080
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

## Проверка

- [ ] Сервис запускается через `systemctl start`.
- [ ] Логи видны через `journalctl -u`.
- [ ] Сервис включён в автозагрузку.
- [ ] После `kill -9 <PID>` systemd поднимает процесс заново.
- [ ] После reboot сервис стартует автоматически.

### Дополнительно

Разобраться:

- `Restart=on-failure` vs `Restart=always`;
- `After=`;
- `Requires=`;
- `Wants=`;
- systemd timers.

---

# 7. Bash scripting

Не нужно становиться Bash-разработчиком. Нужно уметь автоматизировать рутину.

## Освоить

- shebang;
- переменные;
- аргументы `$1`, `$2`, `$@`;
- `$?`;
- exit code;
- `if`;
- `case`;
- `for`;
- `while`;
- functions;
- command substitution `$(...)`;
- `set -euo pipefail`.

## Минимальный шаблон

```bash
#!/usr/bin/env bash
set -euo pipefail

main() {
    echo "hello"
}

main "$@"
```

## Практические скрипты

### Скрипт 1 — health check

Проверяет URL:

```bash
./check_url.sh http://localhost:8080
```

Возвращает:

```text
OK 200
```

или ненулевой exit code при ошибке.

### Скрипт 2 — disk warning

Проверяет заполнение `/` и выводит предупреждение при превышении, например, 80%.

### Скрипт 3 — backup

Создаёт timestamped архив директории:

```text
backup-2026-09-03-2100.tar.gz
```

## Чеклист

- [ ] Умею читать аргументы.
- [ ] Проверяю exit code команды.
- [ ] Использую функции.
- [ ] Обрабатываю ошибочные входные данные.
- [ ] Скрипты исполняемые через `chmod +x`.

---

# 8. Переменные окружения

## Нужно понимать

```bash
env
printenv
export
unset
echo "$PATH"
```

Разобраться:

- shell variable vs environment variable;
- `PATH`;
- `.bashrc`;
- `.bash_profile`;
- `.profile`;
- почему сервис, запущенный через systemd, может не видеть переменные твоего интерактивного shell.

## Практика

- [ ] Создать `APP_ENV=dev`.
- [ ] Передать переменную дочернему процессу.
- [ ] Добавить собственную директорию `~/bin` в `PATH`.
- [ ] Положить туда executable script и запустить его просто по имени.

---

# 9. Диски и файловые системы

## Нужно понимать

- block device;
- partition;
- filesystem;
- mount point;
- inode;
- filesystem space vs inode exhaustion;
- `/etc/fstab`;
- swap;
- LVM — концептуально;
- RAID — концептуально.

## Команды

```bash
lsblk
blkid
df -h
df -i
du -sh
du -h --max-depth=1
mount
umount
findmnt
fdisk -l
free -h
swapon --show
```

## Обязательно уметь ответить

> `No space left on device`, но `df -h` показывает свободное место. Почему?

Одна из основных причин — закончились **inode**. Проверка:

```bash
df -i
```

## Практика

- [ ] Найти самый большой каталог в домашней директории.
- [ ] Сравнить `df` и `du`.
- [ ] Посмотреть количество inode.
- [ ] Посмотреть все mount points.
- [ ] Найти swap.
- [ ] Изучить `/etc/fstab`, не изменяя его без необходимости.

---

# 10. CPU, RAM, Load Average и диагностика ресурсов

## Главные команды

```bash
top
htop
free -h
uptime
vmstat
iostat
pidstat
```

Не все утилиты могут быть установлены по умолчанию.

## Нужно понимать

### Load Average

Обычно показывается за:

```text
1 минута / 5 минут / 15 минут
```

Нельзя оценивать load average без количества CPU cores.

Проверь:

```bash
nproc
uptime
```

### CPU

В `top` обратить внимание на:

```text
us — user CPU
sy — system CPU
wa — I/O wait
id — idle
si — software interrupts
```

### RAM

Понимать:

- free;
- used;
- available;
- page cache;
- swap;
- RSS;
- VSZ.

Важно: маленькое значение `free` само по себе не означает нехватку памяти.

## OOM Killer

Разобраться:

- что происходит при нехватке RAM;
- почему Linux может убить процесс;
- где искать информацию об OOM;
- что такое `oom_score` и `oom_score_adj`.

Пример поиска:

```bash
journalctl -k | grep -i oom
```

## Практика

- [ ] Посмотреть load average.
- [ ] Сопоставить его с количеством CPU cores.
- [ ] Найти процесс с максимальным CPU.
- [ ] Найти процесс с максимальным RAM.
- [ ] Посмотреть `free -h` и объяснить `available`.
- [ ] Запустить `vmstat 1` и посмотреть систему в динамике.

---

# 11. Логи

## Нужно знать

```text
/var/log/
journalctl
syslog
logrotate
```

В современных systemd-системах значительная часть логов доступна через journal.

## Команды

```bash
journalctl
journalctl -b
journalctl -p err
journalctl -u sshd
journalctl --since "10 minutes ago"
journalctl -f
```

## Практика

- [ ] Найти логи текущей загрузки системы.
- [ ] Найти сообщения уровня error.
- [ ] Посмотреть live logs своего systemd-сервиса.
- [ ] Намеренно сломать `ExecStart`, увидеть ошибку и диагностировать её по журналу.

---

# 12. SSH

SSH — обязательный навык DevOps.

## Нужно понимать

- client/server;
- SSH key pair;
- private key;
- public key;
- `authorized_keys`;
- host key;
- SSH agent;
- port forwarding.

## Команды

```bash
ssh
ssh-keygen
ssh-copy-id
scp
rsync
```

Конфиг:

```text
~/.ssh/config
```

Пример:

```sshconfig
Host dev-server
    HostName 192.168.1.50
    User devops
    IdentityFile ~/.ssh/id_ed25519
```

Теперь:

```bash
ssh dev-server
```

## Практика

Лучше использовать отдельную VM.

- [ ] Создать SSH key pair.
- [ ] Настроить вход по ключу.
- [ ] Отключить password authentication на тестовом сервере.
- [ ] Отключить root login.
- [ ] Передать файл через `scp`.
- [ ] Синхронизировать директорию через `rsync`.
- [ ] Настроить `~/.ssh/config`.

### Port forwarding

Разобраться минимум с local forwarding:

```bash
ssh -L 8080:localhost:80 user@server
```

---

# 13. Linux Networking

Эта тема формально пересекается со следующим блоком roadmap — Networks, но базу надо начать здесь.

## Нужно понимать

- IP address;
- subnet;
- default gateway;
- DNS;
- TCP / UDP;
- port;
- listening socket;
- localhost / loopback;
- routing table.

## Команды

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

## Практика

- [ ] Найти свой IP.
- [ ] Найти default gateway.
- [ ] Посмотреть routing table.
- [ ] Найти все listening TCP ports.
- [ ] Запустить локальный HTTP-сервер на 8080.
- [ ] Проверить его через `curl`.
- [ ] Найти процесс, который слушает порт 8080.
- [ ] Проверить DNS через `dig`.
- [ ] Сделать простой `tcpdump` HTTP/DNS-трафика.

---

# 14. Firewall

Для старта достаточно понимать концепцию.

На разных системах можно встретить:

- nftables;
- iptables;
- ufw;
- firewalld.

## Нужно уметь

- разрешить SSH;
- разрешить HTTP/HTTPS;
- закрыть ненужный порт;
- проверить результат с другой машины.

**Важно:** на удалённом сервере сначала разрешай SSH и только потом ужесточай firewall, иначе можно потерять доступ.

---

# 15. Nginx

Nginx нужен и как часть Linux-практики, и дальше для Docker/Kubernetes.

## Нужно понимать

- web server;
- reverse proxy;
- upstream;
- virtual host/server block;
- HTTP headers;
- access log / error log.

## Практика

Поднять приложение:

```text
client -> nginx :80 -> app :8080
```

### Требования

- [ ] Nginx слушает 80 порт.
- [ ] Приложение слушает только localhost:8080.
- [ ] Nginx проксирует запросы в приложение.
- [ ] Есть access log.
- [ ] Есть error log.
- [ ] После перезапуска системы всё стартует автоматически.

---

# 16. Базовая безопасность сервера

На тестовой VM выполнить первичную настройку.

## Задача

- [ ] Создать отдельного пользователя.
- [ ] Настроить `sudo`.
- [ ] Настроить SSH по ключу.
- [ ] Проверить, что вход по ключу работает.
- [ ] Отключить password authentication.
- [ ] Отключить root login по SSH.
- [ ] Включить firewall.
- [ ] Открыть только необходимые порты.
- [ ] Проверить логи SSH.

Не копируй blindly команды hardening из интернета. Для каждого параметра понимай, что именно он меняет.

---

# 17. Главная практическая работа по Linux

Это обязательный итог этапа.

## Условие

Развернуть приложение **без Docker**.

Архитектура:

```text
Internet / localhost
        |
        v
      Nginx
        |
        v
     Backend
        |
        v
    PostgreSQL
```

Можно взять своё приложение на Go / Node.js / Python.

## Требования

### Пользователь

- [ ] Приложение работает не от root.

### Backend

- [ ] Приложение запускается как systemd service.
- [ ] `Restart=on-failure` или аналогичная политика настроена.
- [ ] Логи доступны через journalctl.

### Nginx

- [ ] Nginx работает как reverse proxy.
- [ ] Backend не торчит напрямую наружу.

### Конфигурация

- [ ] Конфиг приложения не захардкожен в коде.
- [ ] Используются environment variables или отдельный config.
- [ ] Секреты не лежат в Git.

### PostgreSQL

- [ ] База установлена и запущена.
- [ ] Создан отдельный DB user.
- [ ] Приложение подключается к БД.

### Диагностика

Ты должен уметь намеренно создать и починить минимум такие поломки:

- [ ] backend не запускается;
- [ ] порт уже занят;
- [ ] Nginx отдаёт `502 Bad Gateway`;
- [ ] неверные права на файл;
- [ ] приложение не видит env variable;
- [ ] PostgreSQL недоступен;
- [ ] DNS не работает;
- [ ] firewall блокирует порт.

---

# 18. Troubleshooting — алгоритм DevOps

Когда что-то не работает, не переустанавливай всё подряд.

Иди сверху вниз.

## 1. Процесс существует?

```bash
systemctl status app
ps aux | grep app
```

## 2. Что говорят логи?

```bash
journalctl -u app -n 100
```

## 3. Порт слушается?

```bash
ss -tulpn
```

## 4. Работает локально?

```bash
curl -v http://127.0.0.1:8080
```

## 5. Работает через Nginx?

```bash
curl -v http://127.0.0.1
```

## 6. Что с DNS?

```bash
dig example.com
```

## 7. Что с маршрутом?

```bash
ip route
ping HOST
traceroute HOST
```

## 8. Что с ресурсами?

```bash
top
free -h
df -h
df -i
```

## 9. Что изменилось недавно?

```bash
git diff
git log
journalctl --since "30 minutes ago"
```

---

# 19. Команды, которые должны стать привычными

Не обязательно знать все флаги наизусть, но эти команды не должны пугать:

```text
cd
ls
cp
mv
rm
find
ln
stat
cat
less
head
tail
grep
awk
sed
cut
sort
uniq
xargs
tee
chmod
chown
sudo
ps
pgrep
kill
top
free
df
du
lsblk
mount
systemctl
journalctl
ssh
scp
rsync
ip
ss
curl
dig
ping
traceroute
tcpdump
```

---

# 20. Вопросы для самопроверки / собеседований

Ответь без Google, затем проверь себя.

## Linux

- [ ] Что такое процесс?
- [ ] Чем PID отличается от PPID?
- [ ] Что такое zombie process?
- [ ] Что означают состояния `R`, `S`, `D`, `Z`, `T`?
- [ ] Чем SIGTERM отличается от SIGKILL?
- [ ] Почему SIGKILL нельзя обработать внутри процесса?
- [ ] Что такое Load Average?
- [ ] Почему load average нельзя оценивать без знания количества CPU cores?
- [ ] Что означает `755` для директории?
- [ ] Что такое inode?
- [ ] Почему может быть `No space left on device`, когда свободное место ещё есть?
- [ ] Чем `df` отличается от `du`?
- [ ] Что такое mount point?
- [ ] Что делает `/etc/fstab`?
- [ ] Что такое swap?
- [ ] Что такое OOM Killer?
- [ ] Что находится в `/proc`?
- [ ] Что делает systemd?
- [ ] Что такое unit file?
- [ ] Чем `systemctl restart` отличается от `reload`?
- [ ] Что такое stdout / stderr?
- [ ] Что делает `2>&1`?
- [ ] Что такое `$PATH`?
- [ ] Как найти процесс, который слушает конкретный порт?
- [ ] Как посмотреть последние 100 строк логов сервиса?
- [ ] Как проверить, работает ли DNS?

## Большой вопрос

- [ ] Опиши загрузку Linux от включения компьютера до запуска пользовательского процесса.

Не обязательно знать kernel internals глубоко, но общая цепочка должна быть понятна:

```text
Firmware -> bootloader -> kernel -> initramfs -> root filesystem -> systemd -> services -> login/session
```

---

# 21. Минимум, который нужен перед Docker

Не переходи дальше, пока ты не можешь без подробной инструкции:

- [ ] ориентироваться по Linux filesystem;
- [ ] искать и редактировать файлы;
- [ ] использовать pipes и redirects;
- [ ] фильтровать логи через `grep`/`awk`/`sed`;
- [ ] понимать пользователей и права;
- [ ] находить процессы;
- [ ] отправлять сигналы;
- [ ] понимать состояния процессов;
- [ ] использовать `systemctl`;
- [ ] использовать `journalctl`;
- [ ] писать простой systemd unit;
- [ ] писать простой Bash script;
- [ ] понимать environment variables;
- [ ] проверять CPU/RAM/disk;
- [ ] понимать `df` vs `du` vs `df -i`;
- [ ] подключаться по SSH-ключу;
- [ ] использовать `scp`/`rsync`;
- [ ] смотреть IP, routes и ports;
- [ ] пользоваться `curl`;
- [ ] понимать на базовом уровне TCP, DNS и HTTP;
- [ ] поднять Nginx reverse proxy;
- [ ] развернуть приложение без Docker;
- [ ] найти проблему по логам и симптомам.

---

# 22. Необязательное углубление

Это полезно, но не должно задерживать переход к Docker.

- [ ] `strace`
- [ ] `ltrace`
- [ ] `lsof`
- [ ] `perf`
- [ ] `sar`
- [ ] `iotop`
- [ ] `iftop`
- [ ] `nload`
- [ ] namespaces
- [ ] cgroups
- [ ] LVM руками
- [ ] RAID руками
- [ ] nftables глубже
- [ ] SELinux/AppArmor
- [ ] kernel parameters / `sysctl`

Особенно полезно перед Docker понимать концептуально:

```text
namespaces = изоляция
cgroups    = ограничения ресурсов
```

Именно эти механизмы Linux лежат в основе контейнеризации.

---

# 23. Итоговый checkpoint

Этап Linux считается закрытым, если ты можешь взять чистую Linux VM и самостоятельно:

1. Создать пользователя.
2. Настроить SSH по ключу.
3. Настроить firewall.
4. Установить приложение.
5. Создать systemd unit.
6. Настроить env/config.
7. Поднять PostgreSQL.
8. Поднять Nginx reverse proxy.
9. Проверить логи.
10. Проверить процессы и порты.
11. Сделать приложение доступным по HTTP.
12. Перезагрузить VM и убедиться, что всё поднялось автоматически.
13. Намеренно сломать один компонент и найти причину без переустановки всей системы.

Если это получается — **переходи к Docker**.

---

