---
tags:
  - linux/administration
  - career/interview
---

# Linux / DevOps — вопросы с собеседований

#flashcards/devops/linux/interview

> Формат: Obsidian **Spaced Repetition** (`st3v3nmw/obsidian-spaced-repetition`).
> Карточка: вопрос → отдельная строка `?` → ответ.
> Внутри карточек нет пустых строк, чтобы multi-line parser корректно отделял карточки.

## Как учить

- Сначала проходи **P0**: определения + базовый troubleshooting.
- На сценариях проговаривай не набор команд, а: **симптом → гипотеза → проверка → действие → проверка результата**.
- Не зубри длинные ответы дословно: цель — уметь объяснить ход диагностики.

## P0 — Linux core, permissions, processes, systemd

Что такое Linux: ядро или операционная система?
?
Строго Linux — ядро. В разговорной речи «Linux» часто означает ОС на базе ядра Linux + userspace: shell, coreutils, systemd и другие компоненты.

Чем программа отличается от процесса?
?
Программа — код на диске. Процесс — выполняющийся экземпляр программы со своим PID, адресным пространством, файловыми дескрипторами, environment и состоянием.

Что такое PID?
?
PID — идентификатор процесса в конкретном PID namespace. Информацию процесса можно смотреть в `/proc/<PID>/`.

Что обычно является PID 1 на современном Linux-сервере?
?
Обычно `systemd`. PID 1 запускает и координирует userspace и имеет особую роль в lifecycle процессов.

Что происходит после ввода `ls` в shell и нажатия Enter?
?
Shell парсит строку и подстановки → определяет builtin/внешнюю команду → ищет executable по `PATH` → запускает программу, обычно через создание дочернего процесса и `execve()` → stdout попадает в терминал или redirect.

Как shell ищет внешнюю команду?
?
По каталогам из `$PATH` после учёта alias/functions/builtins. Проверить разрешение команды: `type <cmd>` или `command -V <cmd>`.

Что такое абсолютный и относительный путь?
?
Абсолютный начинается от `/`; относительный вычисляется от текущего рабочего каталога.

Для чего нужен `/etc`?
?
Для системной конфигурации и конфигурации сервисов: например `/etc/ssh`, `/etc/systemd`, `/etc/fstab`.

Для чего нужен `/var`?
?
Для изменяемых данных: logs, spool, cache и state. Типичный каталог логов — `/var/log`.

Что такое `/proc`?
?
Псевдофайловая система, через которую ядро экспортирует данные о процессах и состоянии системы.

Что такое `/sys`?
?
`sysfs` — интерфейс ядра, представляющий устройства, драйверы и другие kernel objects.

Какие основные типы файлов бывают в Linux?
?
Обычный файл, directory, symbolic link, block device, character device, FIFO/pipe, socket.

Что означают права `755`?
?
`7=rwx`, `5=r-x`, `5=r-x`: владелец читает/пишет/выполняет, группа и остальные читают и выполняют.

Что означают права `644`?
?
`6=rw-`, `4=r--`, `4=r--`: владелец читает и пишет, группа и остальные только читают.

Чем `rwx` для директории отличается от `rwx` для файла?
?
Для directory: `r` — читать список имён, `w` — изменять entries, `x` — проходить через directory и обращаться к объектам по имени. Для файла `x` означает возможность исполнения.

Что такое `umask`?
?
Маска прав, которые снимаются при создании новых файлов/директорий. Типичный `022` обычно приводит к `644` для файлов и `755` для директорий.

Что такое SUID?
?
Специальный bit executable, позволяющий процессу получить effective UID владельца файла. Требует осторожности из-за security риска.

Что такое SGID?
?
Для executable влияет на effective GID; для directory обычно заставляет новые объекты наследовать группу родительской директории.

Что такое sticky bit?
?
На общей writable-directory ограничивает удаление/rename чужих файлов. Классический пример — `/tmp`.

Чем `chmod`, `chown`, `chgrp` отличаются?
?
`chmod` меняет mode bits, `chown` — владельца/группу, `chgrp` — группу.

Где хранятся локальные пользователи и группы?
?
Учётные записи — `/etc/passwd`, password hashes — `/etc/shadow`, группы — `/etc/group`.

Чем `sudo` отличается от `su`?
?
`sudo` выполняет разрешённую команду с повышенными правами по policy; `su` переключает пользователя или запускает shell от другого пользователя.

Почему `Permission denied` возможен даже при `chmod 777 file`?
?
Нет `x` на parent directory, filesystem `noexec`, ACL, SELinux/AppArmor, неправильный interpreter/shebang, namespace/container restrictions и другие причины.

Что показывает `ps aux`?
?
Снимок процессов со сведениями о user, PID, CPU, memory, state и command.

Чем `top` отличается от `ps`?
?
`ps` — одноразовый снимок; `top` периодически обновляет процессы и системные CPU/load/memory показатели.

Какие состояния процессов нужно знать?
?
`R` runnable/running; `S` interruptible sleep; `D` uninterruptible sleep; `Z` zombie; `T` stopped/traced.

Что такое zombie-процесс?
?
Процесс уже завершился, но родитель ещё не получил его exit status через `wait()`. Остаётся запись процесса до reaping.

Почему `kill -9` не убирает zombie?
?
Zombie уже не выполняется. Родитель должен вызвать `wait()`; убивать там фактически нечего.

Что такое orphan process?
?
Процесс, родитель которого завершился раньше него; он переподчиняется subreaper, обычно PID 1 соответствующего namespace.

Что делает `fork()`?
?
Создаёт дочерний процесс. В Linux память эффективно разделяется через copy-on-write до изменений.

Что делает `exec()`?
?
Заменяет программу внутри текущего процесса новой. PID при успешном `exec` не меняется.

Для чего нужен `wait()`?
?
Родитель получает статус завершения child process и освобождает его zombie-запись.

Что делает `kill`?
?
Отправляет signal процессу или process group. По умолчанию обычно `SIGTERM`.

Чем `SIGTERM` отличается от `SIGKILL`?
?
`SIGTERM` можно обработать и корректно завершиться; `SIGKILL` нельзя перехватить или игнорировать. Поэтому KILL — крайний вариант.

Что делает `SIGINT`?
?
Обычно отправляется foreground-процессу по `Ctrl+C`; процесс может установить handler.

Для чего часто используют `SIGHUP`?
?
Исторически сигнал потери терминала; многие daemons используют его по соглашению для reload config, но это зависит от конкретной программы.

Что делают `nice` и `renice`?
?
Меняют nice value и тем самым относительный CPU scheduling priority обычных процессов.

Что такое файловый дескриптор?
?
Целое число в процессе, ссылающееся на открытый объект. Стандартно `0` stdin, `1` stdout, `2` stderr.

Что можно увидеть в `/proc/<PID>/fd/`?
?
Открытые file descriptors процесса: files, sockets, pipes и другие объекты в виде ссылок.

Для чего нужен `lsof`?
?
Показывает открытые файлы и процессы. Полезен для портов, deleted-open files и утечек descriptors.

Для чего нужен `strace`?
?
Трассирует system calls и signals. Помогает увидеть `ENOENT`, `EACCES`, зависание на `connect/read/openat` и другие низкоуровневые причины.

Что такое systemd?
?
System and service manager, обычно PID 1. Управляет units, dependencies, services, targets, timers и интегрируется с journal.

Что такое systemd unit?
?
Объект конфигурации systemd. Частые типы: `.service`, `.socket`, `.target`, `.timer`, `.mount`.

Чем `systemctl start` отличается от `systemctl enable`?
?
`start` запускает unit сейчас; `enable` настраивает его активацию при будущей загрузке/через dependencies. `enable --now` делает оба действия.

Когда нужен `systemctl daemon-reload`?
?
После создания/изменения unit-файлов, чтобы systemd перечитал definitions. Это не restart приложения.

Как посмотреть состояние systemd-сервиса?
?
`systemctl status <unit>` и затем `journalctl -u <unit>`.

Как посмотреть live-логи systemd unit?
?
`journalctl -fu <unit>`.

Как посмотреть journal предыдущей загрузки?
?
`journalctl -b -1`; kernel messages предыдущего boot — `journalctl -k -b -1`, если journal сохранён.

Что делает `Restart=on-failure`?
?
Перезапускает long-running service после failure-сценариев согласно правилам systemd. Нужно учитывать restart rate limits.

Чем `After=` отличается от `Requires=`?
?
`After=` задаёт ordering, но не обязательность. `Requires=` задаёт сильную dependency, но сам по себе не определяет порядок.


## P0/P1 — CPU, память, storage, Bash, SSH, networking, containers

Что такое Load Average в Linux?
?
Усреднённое число задач, которые runnable (`R`) или находятся в uninterruptible sleep (`D`, часто ожидание I/O), за 1, 5 и 15 минут. Это не процент CPU.

Почему Load Average нужно оценивать вместе с числом CPU?
?
Одинаковый load по-разному выглядит на 2 и 16 CPU. Но даже при множестве CPU высокий load может быть вызван D-state/I/O, поэтому одного сравнения с числом cores недостаточно.

Как быстро посмотреть Load Average?
?
`uptime`, `top` или `cat /proc/loadavg`.

Высокий Load Average, но CPU почти idle. Что подозревать?
?
Много задач в `D` state и I/O bottleneck: disk/NFS/storage/device. Проверять process states, `vmstat`, `iostat`, `pidstat`, kernel logs.

Что означают CPU-поля `us`, `sy`, `wa`, `id`?
?
`us` userspace, `sy` kernel/system, `wa` I/O wait, `id` idle. В VM также полезно знать `st`/steal.

Что может означать высокий `sy`?
?
Большая доля CPU уходит в kernel: syscalls, networking, filesystem, interrupts, context switching и другие kernel paths. Нужны дополнительные измерения.

Как найти процесс с высоким CPU?
?
`top`/`htop`, `ps` с сортировкой, `pidstat -u`; для threads — `top -H` или `pidstat -t`.

Чем `free` отличается от `available` в `free -h`?
?
`free` — полностью неиспользуемая RAM. `available` оценивает память, которую можно дать приложениям без активного swapping, учитывая reclaimable cache.

Что такое page cache?
?
Кэш файловых данных в RAM, которым ядро ускоряет I/O. Большой cache сам по себе не означает memory leak.

Что такое swap?
?
Область для вытесненных memory pages. Сам факт использования swap не авария; важны интенсивность `si/so`, latency и memory pressure.

Чем RSS отличается от VSZ/VIRT?
?
RSS — физические страницы процесса, находящиеся в RAM сейчас; VSZ/VIRT — размер виртуального адресного пространства. Большой VIRT не означает столько же занятой RAM.

Что такое OOM Killer?
?
Механизм ядра, который при критическом дефиците памяти может завершить процесс, чтобы восстановить систему. Выбор зависит от OOM scoring и cgroup limits.

Как проверить, что процесс убил OOM Killer?
?
Проверить kernel journal: `journalctl -k` и сообщения `Out of memory`/`Killed process`; также container/orchestrator events при необходимости.

Для чего нужен `vmstat`?
?
Быстрый обзор runnable/blocked tasks, memory, swap, I/O, interrupts/context switches и CPU. Хороший первый инструмент для классификации bottleneck.

Для чего нужен `iostat`?
?
Анализ block-device I/O и CPU: throughput, latency/queue-related показатели и utilization.

Для чего нужен `pidstat`?
?
Per-process/per-thread статистика CPU, memory, I/O и context switches.

RAM растёт, но throughput не растёт. Как диагностировать?
?
Сначала понять, что растёт: RSS, page cache, slab, tmpfs или cgroup usage; затем смотреть allocations/GC, open files, swap/page faults и OOM. Не объявлять leak без evidence.

Чем `df` отличается от `du`?
?
`df` показывает usage файловой системы целиком; `du` суммирует доступные ему файлы/директории. Из-за разных уровней измерения значения могут расходиться.

Почему `df` показывает занятое место, которое не видно в `du`?
?
Классический случай: файл удалён из directory tree, но процесс держит его open. Блоки освобождаются только после закрытия последнего descriptor. Искать `lsof +L1`.

Почему `No space left on device`, хотя `df -h` показывает свободное место?
?
Одна из частых причин — закончились inodes. Проверить `df -i`. Также возможны quota, reserved blocks, tmpfs/FS limits и другие ограничения.

Что такое inode?
?
Структура metadata файла: inode number, type/mode, owner, timestamps, link count, size и ссылки на данные в конкретной FS. Имя файла хранится в directory entry, а не в inode.

Как проверить использование inode?
?
`df -i`.

Чем hard link отличается от symlink?
?
Hard link — ещё одно имя того же inode; обычно не пересекает filesystem. Symlink — отдельный файл с путём к target и может стать dangling.

Что произойдёт после `rm`, если на inode есть ещё hard link?
?
Удалится только одна directory entry; данные останутся, пока link count не станет 0 и файл не перестанет быть open процессами.

Что делает `du -sh /path`?
?
Показывает суммарное disk usage дерева `/path` в human-readable формате.

Как найти большие файлы?
?
Например `find /path -xdev -type f -size +1G -print`, затем `du`/`stat`; на production учитывать стоимость полного обхода.

Для чего нужен `/etc/fstab`?
?
Описывает filesystem mounts и их options. Ошибка в критичной записи может нарушить boot. После изменения полезно проверять `mount -a`.

Почему в `fstab` часто используют UUID?
?
Device names вроде `/dev/sdb1` могут меняться; UUID filesystem обычно стабильнее.

Что такое LVM?
?
Logical Volume Manager: PV → VG → LV. Даёт слой абстракции над block devices и позволяет гибче управлять storage.

Чем partition, filesystem и mount отличаются?
?
Partition — диапазон block device; filesystem — структура хранения файлов; mount — подключение filesystem в дерево каталогов.

Почему RAID не заменяет backup?
?
RAID повышает доступность/устойчивость к некоторым disk failures, но не защищает от удаления, corruption, ransomware, логической ошибки или потери всего массива.

Что такое stdin, stdout и stderr?
?
Стандартные descriptors: `0` stdin, `1` stdout, `2` stderr.

В чём разница между `>` и `>>`?
?
`>` перезаписывает target; `>>` добавляет в конец.

Что означает `2>&1`?
?
Направить fd 2 туда, куда в этот момент направлен fd 1. Порядок redirect-операций важен.

Что делает pipe `|`?
?
Соединяет stdout команды слева со stdin команды справа.

Почему exit code важен в DevOps?
?
CI/CD, shell automation и service managers используют exit status для определения success/failure. Обычно `0` — success, ненулевое — ошибка/особый результат.

Как посмотреть exit code предыдущей команды?
?
`echo $?`.

Для чего нужны `grep`, `awk`, `sed`?
?
`grep` ищет строки/patterns; `awk` обрабатывает records/fields и агрегирует; `sed` потоково преобразует текст.

Зачем `sort | uniq -c | sort -nr`?
?
Подсчитать частоты повторяющихся значений и отсортировать по убыванию.

Что делает `export VAR=value`?
?
Создаёт/изменяет shell variable и экспортирует её в environment дочерних процессов.

Что такое `PATH`?
?
Список каталогов, в которых shell ищет внешние executable без явного пути.

Почему команда работает в shell, но `command not found` в systemd/CI?
?
Другой `PATH`, user или shell; startup files не загружаются. В automation лучше использовать явные зависимости и absolute paths.

Как в общих чертах работает SSH key authentication?
?
Клиент доказывает владение private key; сервер проверяет соответствующий public key/сертификат и policy. Приватный ключ на сервер не копируют.

Как диагностировать SSH connection failure?
?
`ssh -vvv` → DNS/IP/route → TCP port → firewall → `sshd` status/logs → auth/key permissions/config → access policies.

Почему password auth нельзя отключать до проверки key login?
?
Можно заблокировать себе remote access. Проверять key login в отдельной сессии и иметь console/recovery path.

Чем `rsync` полезнее `scp` для синхронизации дерева?
?
Может передавать только изменения, сохранять metadata и эффективно обновлять большие directory trees.

Как посмотреть интерфейсы и IP?
?
`ip addr` и `ip link`.

Как посмотреть routes?
?
`ip route`; маршрут до конкретного IP — `ip route get <IP>`.

Как посмотреть listening ports и процессы?
?
`ss -lntup` с достаточными правами.

Сервис доступен на localhost, но не снаружи. Что проверить?
?
Bind address (`127.0.0.1` vs `0.0.0.0`/конкретный IP), `ss`, host firewall, cloud/network ACL, route/NAT и reverse proxy.

IP работает, hostname нет. Что подозревать?
?
DNS/name resolution: `getent hosts`, `dig`, resolver config, records и cache/TTL.

Почему `ping` работает, а `curl` нет?
?
Ping — ICMP, HTTP(S) — TCP + приложение. Могут быть закрыты ports, сломан TLS, listener, proxy или application.

Для чего нужен `tcpdump`?
?
Packet capture: понять, доходят ли SYN/DNS/ICMP packets и где ломается network flow.

Что такое cgroup?
?
Механизм ядра для иерархической организации процессов и resource control/accounting через controllers, например CPU и memory.

Что такое Linux namespace?
?
Механизм изоляции view ресурсов для группы процессов. Важные namespaces: PID, mount, network, UTS, IPC, user, cgroup.

Чем cgroups отличаются от namespaces?
?
Namespaces в основном дают isolation/visibility; cgroups — grouping, accounting и limits. Контейнеры используют оба механизма.

Почему container не равен VM?
?
Container processes используют kernel host с namespaces/cgroups; VM имеет собственный guest kernel поверх hypervisor.


## P0 — Scenario-based troubleshooting и реальные форматы интервью

Сайт недоступен. Как системно диагностировать проблему?
?
Идти слоями: DNS → route/reachability → TCP 80/443 → listener/firewall → reverse proxy → application → dependencies → resources/logs. На каждом шаге проверять конкретную гипотезу, а не хаотично restart-ить всё.

CPU внезапно 95–100%. Что делать?
?
Подтвердить saturation → найти process/thread (`top`, `pidstat`) → определить `us/sy` → сопоставить с deploy/traffic → logs → profiling (`perf`) или `strace` по ситуации. Сначала собрать evidence, потом вмешиваться.

Disk usage достиг 100%. Какой порядок действий?
?
`df -h` → определить filesystem → `du`/`find` → `df -i` → `lsof +L1` → logs/backups/temp → безопасно освободить/расширить space → добавить prevention: logrotate, alerts, lifecycle.

Удалили огромный log, но `df` не изменился. Почему?
?
Процесс может держать deleted file open. Проверить `lsof +L1`; заставить приложение корректно reopen logs или перезапустить конкретный service.

Load Average высокий, а CPU низкий. Что делать?
?
Проверить D-state и I/O: process states, `vmstat`, `iostat -xz`, `pidstat -d`, storage/NFS health и kernel logs.

Много процессов в `D` state. Что это означает?
?
Они ждут завершения uninterruptible kernel operation, часто I/O. Это симптом storage/NFS/device/driver проблемы; `kill -9` может не сработать, пока kernel operation не вернётся.

Systemd показывает `active (running)`, но клиент не подключается. Что проверять?
?
Listening socket/bind (`ss`), локальный `curl`, firewall, route, reverse proxy, TLS, dependencies и application logs. `active` не гарантирует functional health.

Systemd service попал в restart loop. Как диагностировать?
?
`systemctl status` + `journalctl -u`; exit code/signal; `Restart=` и rate limits; при необходимости остановить loop; воспроизвести `ExecStart` от того же user/env; исправить root cause.

После deploy сервер стал медленным. Как действовать?
?
Сопоставить время деградации с deploy → сравнить CPU/RAM/I/O/network/latency → logs → оценить rollback → сохранить evidence → восстановить service → RCA.

Процесс внезапно исчез. Как понять причину?
?
Service/app logs, systemd exit status, kernel journal на OOM, `coredumpctl` при наличии, audit/orchestrator events. Exit code и terminating signal — ключевые подсказки.

Приложение пишет `Too many open files`. Что проверять?
?
Количество FD (`/proc/<PID>/fd`, `lsof`), limits (`/proc/<PID>/limits`, `ulimit`, systemd `LimitNOFILE=`), динамику FD и возможную leak sockets/files.

После изменения `sshd_config` SSH перестал пускать. Что делать?
?
Не закрывать рабочую session; проверить `sshd -t`, logs, listener/firewall, auth policy и permissions; исправить и controlled reload. Если доступа уже нет — console/recovery.

После изменения `/etc/fstab` система вошла в emergency mode. Что делать?
?
Получить rescue/emergency shell → проверить UUID/device/options → remount root rw при необходимости → исправить fstab → `mount -a` → continue/reboot.

Inodes закончились. Как найти причину?
?
`df -i` → найти directory tree с огромным количеством мелких файлов → определить producer: cache/sessions/temp/logs → безопасно очистить → исправить lifecycle и monitoring.

Web работает по IP, но не по hostname. Что проверять кроме DNS?
?
Сначала DNS, затем HTTP virtual host и TLS SNI/certificate: по IP и hostname reverse proxy может выбирать разные vhosts.

Ping работает, TCP port нет. Как выяснить, где блокировка?
?
Проверить listener через `ss`, соединение через `nc`/`curl`, host/network firewall и route; `tcpdump` покажет SYN/SYN-ACK/RST и поможет локализовать drop.

RAM почти полностью `used`. Это обязательно проблема?
?
Нет. Linux использует RAM под page cache. Смотреть `available`, swap activity, memory pressure, OOM и process/cgroup usage.

Swap активно растёт и latency ухудшается. Что делать?
?
Выяснить memory pressure: RSS consumers, leak/working set, `vmstat` si/so, cgroup limits и OOM risk. Убирать причину/масштабировать, а не просто отключать swap.

Один CPU core 100%, остальные почти idle. Почему?
?
Hot path может быть single-threaded или bottleneck в одном thread. Смотреть per-thread CPU (`top -H`, `pidstat -t`) и profiling.

После logrotate `df` и `du` расходятся. Что подозревать?
?
Service продолжает писать в rotated/deleted inode. Проверить `lsof +L1`; исправить postrotate/reload/reopen behavior.

Nginx возвращает 502. Как диагностировать?
?
Проверить Nginx error log → upstream process/listener → socket permissions → DNS/network → timeouts → application logs. 502 часто означает, что proxy не получил нормальный ответ от upstream.

После reboot service не стартовал, хотя вручную вчера работал. Что проверить?
?
Unit enabled? Boot dependencies/order? mounts/network готовы? environment/secrets? permissions? Смотреть journal текущего boot. Manual start не гарантирует корректный boot graph.

Файл существует, но приложение получает `No such file or directory`. Какие причины возможны?
?
Wrong working directory/symlink, mount namespace/chroot, отсутствующий interpreter из shebang или dynamic loader, race/case mismatch. `strace` помогает увидеть конкретный `ENOENT`.

Как отвечать на вопрос «сервер медленный, что проверишь?»
?
Не перечислять 20 команд подряд. Сказать: подтвержу symptom → классифицирую CPU/RAM/I/O/network → выберу измерение → найду виновный process/service → проверю logs/change history → восстановлю service → prevention.

Как отвечать, если не помнишь флаг Linux-команды?
?
Сказать, что знаешь нужный инструмент и цель, а точный flag уточнишь через `man`/`--help`. В production важнее корректно сформулировать гипотезу и безопасно её проверить, чем изображать память всех flags.

Что такое GRUB и где он участвует в boot process?
?
GRUB — распространённый bootloader. После firmware он загружает выбранные kernel/initramfs и передаёт kernel command line.

Тебе дали вывод `vmstat` и спросили «что не так?». Как читать?
?
Смотреть несколько samples: runnable/blocked tasks, swap `si/so`, I/O, interrupts/context switches, CPU `us/sy/id/wa/st`. Не делать вывод по первой строке без baseline и context.

Какой практический Linux-вопрос хорошо показывает уровень DevOps?
?
Сломанный web service/VM: service не стартует или недоступен. Кандидат должен пройти process → systemd/logs → permissions → ports/network → resources и объяснять гипотезы.

Почему scenario-вопросы важнее trivia вроде «назови 30 команд»?
?
Они проверяют decomposition, troubleshooting, evidence, risk management и понимание связей Linux/network/application. Именно такой формат часто рекомендуют DevOps/SRE interviewers.

Что сделать перед `kill -9` production-процесса?
?
Понять impact и владельца service, собрать минимум evidence, попробовать graceful shutdown (`SIGTERM`/service manager), проверить dependencies и только затем использовать SIGKILL, если это действительно нужно.


---

# Источники исследования

Файл — не копия одного списка из интернета. Вопросы собраны из нескольких типов источников, а ответы нормализованы по официальной документации.

## 1. Твоя DevOps RoadMap

Из приложенной карты взяты ключевые Linux-направления и явно перечисленные interview topics: Load Average, inode, Zombie process, `No space left on device`, boot process, signals, права `755`, LVM, процессы, systemd, OOM Killer, `/proc`, Bash, SSH, `df`/`du`, monitoring и cgroups/namespaces.

## 2. Obsidian Spaced Repetition

Официальный репозиторий:
https://github.com/st3v3nmw/obsidian-spaced-repetition

Плагин поддерживает:
- deck tags вида `#flashcards/...`;
- single-line `Question::Answer`;
- multi-line `Question` + строка `?` + `Answer`.

В этом файле используется multi-line формат.

## 3. Linux man-pages и kernel docs

Load Average:
https://man7.org/linux/man-pages/man5/proc_loadavg.5.html

Signals / `kill(2)`:
https://man7.org/linux/man-pages/man2/kill.2.html

`/proc/<PID>`:
https://man7.org/linux/man-pages/man5/proc_pid.5.html

`chmod`:
https://man7.org/linux/man-pages/man1/chmod.1.html

`df`:
https://man7.org/linux/man-pages/man1/df.1.html

`du`:
https://man7.org/linux/man-pages/man1/du.1.html

cgroup v2:
https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html

## 4. systemd documentation

`journalctl`:
https://www.freedesktop.org/software/systemd/man/latest/journalctl.html

systemd:
https://systemd.io/

Использовано для PID 1, units, journal, service troubleshooting и boot/service lifecycle.

## 5. Red Hat — deleted but open files

https://access.redhat.com/solutions/2316

Подтверждает классический production case: `rm` удаляет directory entry, но если процесс продолжает держать файл открытым, blocks остаются занятыми. Это одна из причин расхождения `df` и `du`.

## 6. Публичные отчёты и обсуждения DevOps/SRE/Linux интервью

r/devops — How would you test Linux proficiency in an interview?
https://www.reddit.com/r/devops/comments/1nk4xzu/how_would_you_test_linux_proficiency_in_an/

r/linuxadmin — Early-career Linux/Fleet Ops interview prep
https://www.reddit.com/r/linuxadmin/comments/1uclgw7/earlycareer_linuxfleet_ops_interview_prep_what/

r/devops — интервью с вопросами PID 1, GRUB, directory permissions
https://www.reddit.com/r/devops/comments/1jnzxom/

r/sysadmin — реальный Linux/DevOps interview: boot process + systemd/upstart + `vmstat`
https://www.reddit.com/r/sysadmin/comments/47njuf/i_have_an_linux_interview_coming_up_im_not/

r/devops — discussion: process lifecycle, zombie, links, inode, pipes и troubleshooting web service
https://www.reddit.com/r/devops/comments/18dj0i9/

Общий вывод этих обсуждений: для DevOps/SRE интервью особенно важен **troubleshooting**. Интервьюеру полезнее услышать, как кандидат диагностирует «server slow», «disk full», «service unreachable», чем только определения и flags.

## 7. Современные подборки 2025–2026

Top 25 Linux Interview Questions for DevOps, Cloud, and SRE Engineers:
https://medium.com/@preetjit82/top-25-linux-interview-questions-for-devops-cloud-and-sre-engineers-74b139c2c016

Linux Troubleshooting Questions for SRE/DevOps Interviews:
https://medium.com/@sre-devops-interview/linux-troubleshooting-questions-for-sre-interviews-01f9db93f51c

EY DevOps Engineer Interview Experience + Questions:
https://medium.com/@sre-devops-interview/ey-devops-engineer-interview-experience-questions-8d8a7606082d

Повторяющиеся современные темы: slow server, CPU/RAM, disk I/O, disk/inode exhaustion, service troubleshooting, boot/fstab recovery и secure access.

---

# Шаблон сильного ответа на scenario-вопрос

1. Уточнить **symptom и impact**.
2. Проверить, **когда началось** и были ли deploy/change/traffic events.
3. Собрать **evidence**: metrics, logs, process/service state.
4. Классифицировать слой: **CPU / RAM / disk / network / service / application**.
5. Сформулировать **гипотезу**.
6. Проверить её конкретной командой и объяснить, **что именно ожидаешь увидеть**.
7. Минимизировать риск: не начинать с reboot, `kill -9` или удаления файлов.
8. Восстановить service: rollback/restart/failover/cleanup — когда понятна причина или это оправданная mitigation.
9. Проверить, что service действительно восстановлен.
10. Назвать prevention: alerts, monitoring, logrotate, capacity, tests, limits, runbook.

На интервью `top → free → df → systemctl` без объяснения логики — слабый ответ. Сильный ответ связывает каждую команду с гипотезой.

## Связанные заметки

- [[Linux]]
- [[Диагностика проблем]]
- [[Вопросы для самопроверки]]
