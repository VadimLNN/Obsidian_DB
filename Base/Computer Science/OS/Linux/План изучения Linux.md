---
tags:
  - linux/administration
  - linux/shell
  - linux/networking
---

# План изучения Linux

[[OS/Linux/Linux|Карта Linux]]

> Основано на блоке **Linux** из приложенной DevOps RoadMap.  
> Цель конспекта — не просто перечислить команды, а собрать в одном месте теорию, команды, практику и типовые вопросы с собеседований из roadmap.

## Почему Linux важен для DevOps

Linux — один из самых важных и больших блоков в DevOps.

С ним DevOps работает постоянно:

- серверы;
- виртуальные машины;
- контейнеры;
- [[Kubernetes]];
- сервисы;
- логи;
- сеть;
- процессы;
- файловые системы;
- мониторинг ресурсов.

Основная задача на этом этапе — научиться не просто пользоваться Linux, а **понимать, что происходит с системой и почему она работает или не работает**.

## Что нужно знать по Linux целиком

Roadmap сводит Linux к следующим областям:

```text
загрузка и инициализация
файловая система
права и пользователи
процессы
службы
сигналы
состояния процессов
bash-скрипты
сеть
nginx
ресурсы и железо
```

## Загрузка Linux

Roadmap указывает популярный вопрос:

> Опиши загрузку Linux от включения питания до появления приглашения в систему.

Эту тему нужно изучить отдельно в рамках:

```text
загрузка и инициализация
```

## Что учить глубже

После базы roadmap предлагает углубиться в:

```text
fork / exec / wait
nice / renice
nohup / screen / tmux
cgroups
namespaces
systemd
strace
ltrace
vmstat
iostat
sar
dstat
pidstat
RSS / VSZ
page cache
swap
ss
netstat
iftop
nload
perf
journalctl
syslog
logrotate
```

## Практический чек-лист

Отмечай по мере изучения.

### Терминал

- [ ] `cd`
- [ ] `mkdir`
- [ ] `touch`
- [ ] `rm`
- [ ] `find`
- [ ] `tree`
- [ ] `ln`
- [ ] `stat`
- [ ] `file`

### Потоки

- [ ] stdin
- [ ] stdout
- [ ] stderr
- [ ] `>`
- [ ] `>>`
- [ ] `2>&1`
- [ ] pipes

### Текст

- [ ] grep
- [ ] awk
- [ ] sed
- [ ] cut
- [ ] sort
- [ ] uniq
- [ ] wc
- [ ] xargs

### Процессы

- [ ] ps aux
- [ ] PID
- [ ] kill
- [ ] SIGTERM
- [ ] SIGKILL
- [ ] состояния процессов
- [ ] zombie
- [ ] fork / exec / wait

### systemd

- [ ] systemctl
- [ ] написать unit
- [ ] enable
- [ ] restart
- [ ] journalctl
- [ ] Restart=always

### Ресурсы

- [ ] top
- [ ] htop
- [ ] load average
- [ ] CPU us/sy/wa/si/id
- [ ] free -h
- [ ] swap
- [ ] page cache
- [ ] OOM Killer

### Диски

- [ ] df
- [ ] du
- [ ] inode
- [ ] mount
- [ ] fstab
- [ ] LVM
- [ ] RAID

### SSH

- [ ] ssh-keygen
- [ ] ssh-copy-id
- [ ] ssh config
- [ ] scp
- [ ] rsync
- [ ] port forwarding

### Диагностика

- [ ] vmstat
- [ ] iostat
- [ ] sar
- [ ] pidstat
- [ ] strace
- [ ] ltrace

### Контейнерный фундамент

- [ ] cgroups
- [ ] namespaces

## Что дальше

Согласно roadmap, после Linux следующий крупный этап:

```text
Linux
   ↓
сети
   ↓
Docker
   ↓
CI/CD
   ↓
Ansible
   ↓
Kubernetes
```

Не нужно пытаться знать Linux полностью перед [[Docker]].

Критерий перехода дальше:

> ты умеешь самостоятельно поднять приложение на Linux, запустить его как сервис и диагностировать базовые проблемы процессов, памяти, диска, логов и сети.

## Команды, которые должны стать привычными

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

## Минимум, который нужен перед Docker

Не переходи дальше, пока ты не можешь без подробной инструкции:

- [ ] ориентироваться по [[Структура каталогов Linux|Linux filesystem]];
- [ ] искать и редактировать файлы;
- [ ] использовать pipes и redirects;
- [ ] фильтровать логи через `grep`/`awk`/`sed`;
- [ ] понимать [[Пользователи и группы|пользователей]] и [[Права доступа|права]];
- [ ] находить [[Процессы и сигналы|процессы]];
- [ ] отправлять сигналы;
- [ ] понимать состояния процессов;
- [ ] использовать `systemctl`;
- [ ] использовать `journalctl`;
- [ ] писать простой [[systemd и сервисы|systemd unit]];
- [ ] писать простой [[Bash-скрипты|Bash script]];
- [ ] понимать [[Bash и Fish|environment variables]];
- [ ] проверять [[Ресурсы и диагностика|CPU/RAM]]/[[Дисковое пространство|disk]];
- [ ] понимать `df` vs `du` vs `df -i`;
- [ ] подключаться по [[SSH и передача файлов|SSH-ключу]];
- [ ] использовать `scp`/`rsync`;
- [ ] смотреть IP, routes и ports;
- [ ] пользоваться `curl`;
- [ ] понимать на базовом уровне TCP, DNS и HTTP;
- [ ] поднять [[Nginx]] reverse proxy;
- [ ] развернуть приложение без Docker;
- [ ] найти проблему по логам и симптомам.

## Необязательное углубление

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

## Связанные темы

- [[OS/Linux/Практика Linux]]
- [[OS/Linux/Вопросы для самопроверки]]
