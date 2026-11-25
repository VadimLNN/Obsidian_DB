---
tags:
  - education/graduate
  - linux/administration
  - linux/networking
  - virtualization
---
# Практическое задание №1

## Подготовка тестовой инфраструктуры на базе VirtualBox

> Основано на методичке `lab01.pdf`.
>
> Хост: **Arch Linux**  
> Гостевая ОС: **Astra Linux Common Edition 2.12 x86-64**

---

# 1. Что требуется сделать

1. Установить VirtualBox.
2. Установить VirtualBox Extension Pack.
3. Скачать ISO Astra Linux.
4. Установить Astra Linux без графического интерфейса.
5. Установить базовые средства и [[Base/Computer Science/OS/Linux/Administration/SSH и передача файлов|SSH]].
6. Перевести виртуальный диск в режим **Multiattach**.
7. Создать вторую VM на том же базовом диске.
8. Настроить Host-Only сеть VirtualBox.
9. Настроить DHCP для `eth0`.
10. Настроить `/etc/issue` для вывода IP до авторизации.
11. Подключиться к Astra Linux по SSH.
12. Настроить Bash prompt для `root` и обычного пользователя.
13. Написать [[Base/Computer Science/OS/Linux/Terminal/Bash-скрипты|Bash-скрипт]] для управления `pretty hostname`.
14. Сделать скрипт исполняемым и [[Base/Computer Science/OS/Linux/Administration/Права доступа|доступным только владельцу]].
15. Создать дополнительного обычного пользователя.

---

# 2. Итоговая схема

```text
                 Arch Linux
                    Host
                     |
                  vboxnet0
               192.168.56.1
                     |
          +----------+----------+
          |   Host-Only Network |
          |   192.168.56.0/24   |
          +----------+----------+
                     |
            +--------+--------+
            |                 |
         astra1             astra2
            |                 |
      192.168.56.101    192.168.56.102
```

---

# 3. Установка VirtualBox на Arch Linux

## 3.1. Проверить ядро

```bash
uname -r
```

Если обычное ядро Arch:

```bash
sudo pacman -Syu
sudo pacman -S virtualbox virtualbox-host-dkms linux-headers
```

## 3.2. Загрузить модули VirtualBox

`modprobe` - это команда в Linux, которая нужна, чтобы загружать (вставлять) и выгружать (удалять) модули ядра.

```bash
sudo modprobe vboxdrv
sudo modprobe vboxnetadp
sudo modprobe vboxnetflt
```

Проверка:

```bash
lsmod | grep vbox
```

Ожидаются модули примерно:

```text
vboxnetflt
vboxnetadp
vboxdrv
```

## 3.3. Добавить пользователя в `vboxusers`

```bash
sudo usermod -aG vboxusers "$USER"
```

Перезагрузить ПК:

```bash
reboot
```

## 3.4. Проверить VirtualBox

```bash
VBoxManage --version
```

Запуск GUI:

```bash
VirtualBox
```

---

# 4. Extension Pack

Сначала узнать версию VirtualBox:

```bash
VBoxManage --version
```

Версия Extension Pack должна совпадать с версией VirtualBox.

Пример для версии `7.2.18`:

```bash
cd ~/Downloads
curl -LO https://download.virtualbox.org/virtualbox/7.2.18/Oracle_VirtualBox_Extension_Pack-7.2.18.vbox-extpack
```

Установка:

```bash
sudo VBoxManage extpack install \
    Oracle_VirtualBox_Extension_Pack-7.2.18.vbox-extpack
```

Подтвердить лицензию:

```text
y
```

Проверка:

```bash
VBoxManage list extpacks
```

---

# 5. Скачать Astra Linux

```bash
cd ~/Downloads
curl -L \
    -o alce-current.iso \
    https://dl.astralinux.ru/astra/stable/2.12_x86-64/iso/alce-current.iso
```

Проверить:

```bash
ls -lh alce-current.iso
```

---

# 6. Создать первую VM

В VirtualBox выбрать:

```text
New / Создать
```

Рекомендуемые параметры:

```text
Name: astra1
Type: Linux
Version: Debian (64-bit) / Other Linux (64-bit)
RAM: 2048 MB
CPU: 2
Disk: VDI
Disk size: 12–20 GB
Allocation: Dynamically allocated
```

По методичке при установке требуется минимум **2 ГБ RAM**. После установки можно уменьшить до **512 МБ**, но для удобства лучше оставить хотя бы 1024 МБ.

---

# 7. Подключить ISO

```text
astra1
→ Settings
→ Storage
→ Optical Drive
→ Choose a disk file
```

Выбрать:

```text
alce-current.iso
```

Запустить VM.

---

# 8. Установить Astra Linux

Можно выбрать:

```text
Graphical Install
```

Пример параметров:

```text
Hostname: astra
Username: user
Password: свой пароль
```

---

# 9. Разметка диска

Выбрать:

```text
Guided — use entire disk
```

Затем:

```text
All files in one partition
```

Пример:

```text
/dev/sda
├── /dev/sda1   /
└── swap
```

LVM, RAID и сложная разметка для этой работы не нужны.

---

# 10. Выбор пакетов

Итоговая VM должна быть **без GUI**.

Оставить:

```text
[X] Базовые средства
[X] Средства удалённого доступа SSH
```

Не устанавливать:

```text
[ ] Рабочий стол Fly
```

---

# 11. Завершить установку

Установить GRUB, перезагрузить VM.

После установки отключить ISO:

```text
Devices
→ Optical Drives
→ Remove disk
```

После загрузки ожидается консоль:

```text
Astra Linux

astra login:
```

Войти под созданным пользователем.

---

# 12. Проверить систему

Версия Astra:

```bash
cat /etc/astra_version
```

или:

```bash
lsb_release -a
```

Ядро:

```bash
uname -r
```

Проверить target:

```bash
systemctl get-default
```

Ожидаемо:

```text
multi-user.target
```

---

# 13. Проверить SSH

```bash
systemctl status ssh
```

Если не запущен:

```bash
sudo systemctl enable --now ssh
```

Проверить порт 22:

```bash
ss -lntp | grep :22
```

---

# 14. Выключить VM

```bash
sudo poweroff
```

VM должна быть полностью выключена перед изменением типа диска.

---

# 15. Перевести диск в Multiattach

На Arch:

```bash
VBoxManage list hdds
```

Найти путь к диску, например:

```text
/home/vadim_lnn/VirtualBox VMs/astra1/astra1.vdi
```

Перевести диск:

```bash
VBoxManage modifymedium disk \
    "/home/vadim_lnn/VirtualBox VMs/astra1/astra1.vdi" \
    --type multiattach
```

Проверить:

```bash
VBoxManage showmediuminfo disk \
    "/home/vadim_lnn/VirtualBox VMs/astra1/astra1.vdi"
```

Искать:

```text
multiattach
```

## Что такое Multiattach

Один базовый VDI используется несколькими VM, но изменения каждой VM сохраняются отдельно в differencing image.

```text
             Astra base VDI
                  |
             Multiattach
              /       \
          astra1      astra2
             |          |
         diff disk   diff disk
```

---

# 16. Создать вторую VM

Создать:

```text
Name: astra2
RAM: 512–1024 MB
```

Новый HDD не создавать.

Затем:

```text
astra2
→ Settings
→ Storage
→ SATA Controller
→ Add Hard Disk
→ Choose Existing Disk
```

Выбрать тот же:

```text
astra1.vdi
```

---

# 17. Создать Host-Only сеть

В VirtualBox:

```text
File
→ Tools
→ Network Manager
→ Host-Only Networks
```

Создать сеть:

```text
Name: vboxnet0
IPv4: 192.168.56.1
Mask: 255.255.255.0
```

Включить DHCP, например:

```text
DHCP Server: 192.168.56.100
Mask: 255.255.255.0
Lower address: 192.168.56.101
Upper address: 192.168.56.254
```

---

# 18. Подключить обе VM к Host-Only

Для `astra1`:

```text
Settings
→ Network
→ Adapter 1
→ Enable Network Adapter
→ Attached to: Host-Only Adapter
→ Name: vboxnet0
```

То же сделать для `astra2`.

---

# 19. Проверить Host-Only на Arch

```bash
ip -br addr
```

Ожидаемо:

```text
vboxnet0    UP    192.168.56.1/24
```

Дополнительно:

```bash
VBoxManage list hostonlyifs
```

---

# 20. Проверить интерфейсы Astra

Внутри VM:

```bash
ip -br link
```

```bash
ip -br addr
```

Желательно увидеть:

```text
lo       UNKNOWN   127.0.0.1/8
eth0     UP        192.168.56.101/24
```

Если вместо `eth0` показывается `enp0s3`, система использует predictable network interface names. Для строгого соответствия методичке желательно использовать имена `ethX`.

---

# 21. Настроить `/etc/network/interfaces.d/eth0`

```bash
sudo mkdir -p /etc/network/interfaces.d
sudo nano /etc/network/interfaces.d/eth0
```

Содержимое:

```text
auto eth0
iface eth0 inet dhcp
```

Смысл:

- `auto eth0` — поднимать интерфейс автоматически;
- `iface eth0 inet dhcp` — получать IPv4 через DHCP.

---

# 22. Проверить подключение `interfaces.d`

```bash
cat /etc/network/interfaces
```

Должно быть что-то вроде:

```text
source /etc/network/interfaces.d/*
```

или:

```text
source-directory /etc/network/interfaces.d
```

Если нет:

```bash
echo 'source /etc/network/interfaces.d/*' | \
sudo tee -a /etc/network/interfaces
```

---

# 23. Перезапустить сеть

Проще всего:

```bash
sudo reboot
```

После загрузки:

```bash
ip -4 -br addr
```

Пример:

```text
lo      UNKNOWN   127.0.0.1/8
eth0    UP        192.168.56.101/24
```

---

# 24. Проверить связь

## Astra → Host

```bash
ping -c 4 192.168.56.1
```

## Host → Astra

На Arch:

```bash
ping -c 4 192.168.56.101
```

## Astra1 → Astra2

```bash
ping -c 4 192.168.56.104
```

## Astra2 → Astra1

```bash
ping -c 4 192.168.56.103
```

---

# 25. Настроить `/etc/issue`

```bash
sudo nano /etc/issue
```

Пример:

```text
Astra Linux

lo:   \4{lo}
eth0: \4{eth0}
eth1: \4{eth1}
eth2: \4{eth2}
eth3: \4{eth3}
eth4: \4{eth4}

```

`\4{eth0}` выводит IPv4-адрес интерфейса `eth0` перед приглашением `login:`.

Проверить, выйдя из пользователя:

```bash
exit
```

Пример:

```text
Astra Linux

lo:   127.0.0.1
eth0: 192.168.56.101
eth1:
eth2:
eth3:
eth4:

astra login:
```

После входа сравнить с:

```bash
ip -4 -br addr
```

> В методичке есть несогласованность: DHCP настраивается для `eth0`, а в пункте про `/etc/issue` перечислены `lo`, `eth1`, `eth2`, `eth3`, `eth4`. Если преподаватель требует физическое наличие `eth1–eth4`, нужно добавить дополнительные виртуальные NIC.

---

# 26. Подключиться по SSH

Узнать IP VM:

```bash
ip -4 -br addr
```

Например:

```text
eth0   UP   192.168.56.101/24
```

На Arch:

```bash
ssh user@192.168.56.101
```

При первом подключении подтвердить fingerprint:

```text
yes
```

После ввода пароля получаем:

```text
user@astra:~$
```

---

# 27. Настроить prompt для `root`

```bash
sudo -i
nano /root/.bashrc
```

В конец добавить:

```bash
PS1='\[\e[31m\]\u\[\e[0m\]@\h:\w\$ '
```

Применить:

```bash
source /root/.bashrc
```

Расшифровка:

```text
\e[31m  красный цвет
\u       имя пользователя
\e[0m   сброс цвета
\h       hostname
\w       текущий каталог
```

Имя `root` должно отображаться красным.

---

# 28. Настроить prompt обычного пользователя

Выйти из root:

```bash
exit
```

Открыть:

```bash
nano ~/.bashrc
```

Добавить:

```bash
PS1='Команда: \# | Время: \t\n\u@\h:\w\$ '
```

Применить:

```bash
source ~/.bashrc
```

Пример:

```text
Команда: 15 | Время: 18:42:37
user@astra:~$
```

Где:

```text
\#  номер текущей команды Bash
\t  текущее время HH:MM:SS
\n  новая строка
```

---

# 29. Создать Bash-скрипт для Pretty Hostname

Создать файл:

```bash
nano ~/node-description.sh
```

Содержимое:

```bash
#!/bin/bash

description=""
set_value=false

usage() {
    echo "Использование:"
    echo "  $0 -n <описание> [-s]"
    echo
    echo "Опции:"
    echo "  -n <описание>  Новое описание узла (pretty hostname)"
    echo "  -s             Установить новое значение, если оно отличается"
}

while getopts ":n:s" option; do
    case "$option" in
        n)
            description="$OPTARG"
            ;;
        s)
            set_value=true
            ;;
        :)
            echo "Ошибка: опция -$OPTARG требует аргумент."
            usage
            exit 1
            ;;
        \?)
            echo "Ошибка: неизвестная опция -$OPTARG."
            usage
            exit 1
            ;;
    esac
done

if [[ -z "$description" ]]; then
    echo "Ошибка: необходимо указать -n <описание>."
    usage
    exit 1
fi

current=$(hostnamectl --pretty)

echo "Текущее описание узла: '$current'"
echo "Заданное описание узла: '$description'"

if [[ "$current" == "$description" ]]; then
    echo "Описание узла уже соответствует заданному."
    exit 0
fi

echo "Описание узла отличается."

if [[ "$set_value" == true ]]; then
    if [[ "$EUID" -ne 0 ]]; then
        echo "Ошибка: для изменения hostname запустите скрипт через sudo."
        exit 1
    fi

    hostnamectl set-hostname --pretty "$description"

    echo "Новое описание установлено:"
    hostnamectl --pretty
else
    echo "Изменение не выполнено."
    echo "Для установки нового значения используйте опцию -s."
fi
```

---

# 30. Разбор скрипта

## Shebang

```bash
#!/bin/bash
```

Запускать файл через Bash.

## Переменные

```bash
description=""
set_value=false
```

- `description` — новое описание;
- `set_value` — передана ли опция `-s`.

## Обработка аргументов

```bash
while getopts ":n:s" option
```

`n:` означает, что `-n` требует аргумент.

Пример:

```bash
-n "Astra laboratory server"
```

`-s` аргумента не требует.

## Получение текущего Pretty Hostname

```bash
current=$(hostnamectl --pretty)
```

## Сравнение

```bash
if [[ "$current" == "$description" ]]
```

## Установка нового значения

```bash
hostnamectl set-hostname --pretty "$description"
```

---

# 31. Сделать скрипт исполняемым и закрыть доступ другим

```bash
chmod 700 ~/node-description.sh
```

Проверить:

```bash
ls -l ~/node-description.sh
```

Ожидаемо:

```text
-rwx------ 1 user user ...
```

`700` означает:

```text
владелец: rwx
группа:   ---
остальные:---
```

---

# 32. Проверить скрипт

## Текущее значение

```bash
hostnamectl --pretty
```

## Запуск без `-s`

```bash
./node-description.sh -n "Astra laboratory server"
```

Пример:

```text
Текущее описание узла: ''
Заданное описание узла: 'Astra laboratory server'
Описание узла отличается.
Изменение не выполнено.
Для установки нового значения используйте опцию -s.
```

## Запуск с `-s`

```bash
sudo ./node-description.sh \
    -n "Astra laboratory server" \
    -s
```

Проверить:

```bash
hostnamectl
```

Ожидаемо:

```text
Pretty hostname: Astra laboratory server
```

## Повторный запуск

```bash
sudo ./node-description.sh \
    -n "Astra laboratory server" \
    -s
```

Теперь ожидается сообщение, что описание уже совпадает.

---

# 33. Создать дополнительного пользователя

```bash
sudo adduser student
```

Задать пароль (123).

Проверить:

```bash
id student
```

Войти:

```bash
su - student
```

Проверить:

```bash
whoami
```

Ожидаемо:

```text
student
```

---

# 34. Финальная проверка

## На Arch

```bash
VBoxManage --version
VBoxManage list extpacks
VBoxManage list hostonlyifs
VBoxManage list hdds
```

## В Astra

```bash
cat /etc/astra_version
uname -r
systemctl get-default
ip -4 -br addr
cat /etc/network/interfaces.d/eth0
systemctl status ssh
cat /etc/issue
hostnamectl
ls -l ~/node-description.sh
id student
```

## Проверка SSH

На Arch:

```bash
ssh user@192.168.56.101
```

---

# 35. Что показать преподавателю

## VirtualBox

```bash
VBoxManage --version
VBoxManage list extpacks
VBoxManage list hostonlyifs
```

Показать две VM:

```text
astra1
astra2
```

## Astra Linux

```bash
cat /etc/astra_version
```

## Отсутствие GUI

```bash
systemctl get-default
```

## Сеть

```bash
ip -4 -br addr
cat /etc/network/interfaces.d/eth0
```

## `/etc/issue`

```bash
cat /etc/issue
```

Затем выйти и показать IP перед `login:`.

## SSH

```bash
ssh user@192.168.56.101
```

## Root prompt

```bash
sudo -i
```

Показать красное имя `root`.

## User prompt

Показать:

```text
Команда: <номер> | Время: <HH:MM:SS>
user@astra:~$
```

## Скрипт

```bash
ls -l ~/node-description.sh
./node-description.sh -n "Test server"
sudo ./node-description.sh -n "Test server" -s
hostnamectl
```

## Пользователь

```bash
id student
```

---

# 36. Финальный чек-лист

- [ ] VirtualBox установлен
- [ ] Extension Pack установлен
- [ ] Astra Linux установлена
- [ ] GUI отсутствует
- [ ] SSH установлен и запущен
- [ ] VDI переведён в Multiattach
- [ ] Созданы `astra1` и `astra2`
- [ ] Создан `vboxnet0`
- [ ] DHCP включён
- [ ] Astra получает IP
- [ ] Host видит VM
- [ ] VM видят друг друга
- [ ] `/etc/network/interfaces.d/eth0` существует
- [ ] `/etc/issue` показывает IP
- [ ] SSH с Arch работает
- [ ] `root` красный в prompt
- [ ] обычный пользователь видит номер команды
- [ ] обычный пользователь видит время
- [ ] скрипт принимает `-n`
- [ ] скрипт принимает `-s`
- [ ] скрипт показывает текущий pretty hostname
- [ ] скрипт меняет pretty hostname
- [ ] права скрипта `700`
- [ ] создан дополнительный пользователь
- [ ] пользователю задан пароль

---

# 37. Ключевые команды

```bash
# VirtualBox
VBoxManage --version
VBoxManage list extpacks
VBoxManage list hostonlyifs
VBoxManage list hdds

# Astra
cat /etc/astra_version
uname -r
systemctl get-default

# Сеть
ip -br link
ip -4 -br addr
cat /etc/network/interfaces.d/eth0

# SSH
systemctl status ssh
sudo systemctl enable --now ssh
ss -lntp | grep :22
ssh user@192.168.56.101

# Pretty hostname
hostnamectl
hostnamectl --pretty
sudo hostnamectl set-hostname --pretty "Astra laboratory server"

# Скрипт
chmod 700 ~/node-description.sh
ls -l ~/node-description.sh
./node-description.sh -n "Astra laboratory server"
sudo ./node-description.sh -n "Astra laboratory server" -s

# Пользователь
sudo adduser student
id student
su - student
whoami
```

---

# 38. Основные конфигурацинные файлы

```text
/etc/network/interfaces
/etc/network/interfaces.d/eth0
/etc/issue
/root/.bashrc
/home/user/.bashrc
/home/user/node-description.sh
```

---

# 39. Краткая теория для защиты

## VirtualBox

Система виртуализации для запуска гостевых ОС внутри хостовой системы.

## VDI

Формат виртуального диска VirtualBox.

## Multiattach

Несколько VM используют один базовый диск, но изменения каждой VM сохраняются отдельно.

## Host-Only

Изолированная виртуальная сеть:

```text
Host ↔ VM ↔ VM
```

## DHCP

Автоматически выдаёт сетевые параметры, в том числе IP-адрес.

## `/etc/issue`

Файл, содержимое которого выводится перед локальной авторизацией.

## SSH

Secure Shell — защищённый удалённый доступ к командной строке Linux.

## `.bashrc`

Конфигурация интерактивного Bash.

## `PS1`

Переменная, задающая вид приглашения Bash.

## `getopts`

Встроенный механизм Bash для разбора параметров командной строки.

## `hostnamectl`

Утилита systemd для управления hostname.

## `chmod 700`

Владелец может читать, изменять и запускать файл, остальные пользователи доступа не имеют.

---

# 40. Порядок выполнения лабораторной одним списком

```text
1. Установить VirtualBox
2. Установить Extension Pack
3. Скачать Astra Linux ISO
4. Создать astra1
5. Выделить 2 ГБ RAM
6. Подключить ISO
7. Установить Astra Linux
8. Использовать весь виртуальный диск
9. Все файлы разместить в одном разделе
10. Не устанавливать Fly
11. Установить базовые средства
12. Установить SSH
13. Проверить консольную загрузку
14. Проверить SSH
15. Выключить astra1
16. Перевести VDI в Multiattach
17. Создать astra2
18. Подключить тот же VDI
19. Создать vboxnet0
20. Включить DHCP
21. Подключить astra1 к Host-Only
22. Подключить astra2 к Host-Only
23. Проверить интерфейсы
24. Создать /etc/network/interfaces.d/eth0
25. Настроить DHCP
26. Проверить IP
27. Проверить ping Host ↔ Astra
28. Проверить ping Astra1 ↔ Astra2
29. Настроить /etc/issue
30. Проверить IP до авторизации
31. Подключиться по SSH
32. Настроить root PS1
33. Настроить user PS1
34. Создать node-description.sh
35. Реализовать -n
36. Реализовать -s
37. Получать pretty hostname
38. Сравнивать значения
39. Устанавливать новое значение
40. Сделать chmod 700
41. Проверить скрипт без -s
42. Проверить скрипт с -s
43. Проверить повторный запуск
44. Создать пользователя student
45. Назначить пароль
46. Провести финальную проверку
```
