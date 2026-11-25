---
tags:
  - linux/administration
  - linux/shell
  - linux/filesystems
---

# Linux

Карта знаний по Linux: терминал и оболочки, файловые системы, администрирование, сеть и практика. Объяснения, примеры и задания из конспектов Linux и Linux Devops объединены в тематических заметках.

## Терминал и оболочки

- [[OS/Linux/Terminal/Terminal|Что такое терминал]] — виды терминалов, TTY/PTY и отличие от оболочки.
- [[OS/Linux/Terminal/Основы терминала и Shell]] — основные приёмы работы, установка пакетов в Arch.
- [[OS/Linux/Terminal/Bash и Fish]] — переменные окружения, конфигурация и различия оболочек.
- [[OS/Linux/Terminal/Bash-скрипты]] — условия, циклы, функции, коды завершения и упражнения.
- [[OS/Linux/Terminal/Навигация и файловые операции]] — пути, создание, копирование, перемещение и удаление.
- [[OS/Linux/Terminal/Просмотр файлов]] — `cat`, `less`, `more`, `head`, `tail`.
- [[OS/Linux/Terminal/Поиск и обработка текста]] — `find`, `locate`, `grep`, `awk`, `sed`, `sort`, `uniq`, `wc`.
- [[OS/Linux/Terminal/Потоки и перенаправления]] — stdin, stdout, stderr и запись в файлы.
- [[OS/Linux/Terminal/Конвейеры и операторы]] — цепочки команд и условное выполнение.
- [[OS/Linux/Terminal/История команд и горячие клавиши]] — поиск истории и редактирование ввода.
- [[OS/Linux/Terminal/Справка и автодополнение]] — `man`, `help`, `apropos`, дополнения Fish.
- [[OS/Linux/Terminal/Архивирование и сжатие]] — `tar`, `zip`, `unzip`.
- [[OS/Linux/Terminal/Nano]] и [[OS/Linux/Terminal/Vim]] — редакторы.
- [[OS/Linux/Terminal/Шпаргалка]] — команды и основные флаги для быстрого повторения.

## Файловая система и диски

- [[OS/Linux/File System/Структура каталогов Linux]] — назначение системных каталогов.
- [[OS/Linux/File System/Inodes и ссылки]] — метаданные, жёсткие и символические ссылки.
- [[OS/Linux/File System/Дисковое пространство]] — `df`, `du`, место и inode.
- [[OS/Linux/File System/Диски и разделы]] — блочные устройства, MBR/GPT, обзор LVM и RAID.
- [[OS/Linux/File System/Файловые системы и монтирование]] — `mkfs`, `mount`, UUID и `fstab`.

## Пользователи и безопасность

- [[OS/Linux/Administration/Пользователи и группы]] — учётные записи и группы.
- [[OS/Linux/Administration/Root и Sudo]] — привилегии, `sudoers`, `wheel` и `visudo`.
- [[OS/Linux/Administration/Права доступа]] — `rwx`, `chmod`, упражнения.
- [[OS/Linux/Administration/Владельцы файлов]] — `chown` и `chgrp`.
- [[OS/Linux/Administration/Специальные права и umask]] — SUID, SGID, sticky bit и маска создания файлов.
- [[OS/Linux/Administration/Безопасность сервера]] — первоначальная настройка и проверка доступа.

## Процессы, службы и ресурсы

- [[OS/Linux/Administration/Процессы и сигналы]] — PID, состояния, сигналы, приоритеты, фоновые задания и `/proc`.
- [[OS/Linux/Administration/systemd и сервисы]] — управление службами, unit-файлы и автозапуск.
- [[OS/Linux/Administration/Логи и journalctl]] — системный журнал, файлы логов и ротация.
- [[OS/Linux/Administration/Ресурсы и диагностика]] — CPU, RAM, Load Average, OOM и диагностические утилиты.
- [[OS/Linux/Administration/Cgroups и Namespaces]] — ограничения ресурсов и изоляция.

## Сеть и удалённая работа

- [[OS/Linux/Administration/Сеть Linux]] — адреса, маршруты, DNS, порты и инструменты диагностики.
- [[OS/Linux/Administration/SSH и передача файлов]] — ключи, SSH config, туннели, `scp` и `rsync`.
- [[OS/Linux/Administration/Firewall]] — фильтрация сетевого доступа.
- [[OS/Linux/Administration/Nginx]] — веб-сервер и reverse proxy.

## Изучение и практика

- [[OS/Linux/План изучения Linux]] — что изучать, тематические чек-листы и подготовка к [[Docker]].
- [[OS/Linux/Практика Linux]] — развёртывание приложения без Docker и итоговая проверка.
- [[OS/Linux/Вопросы для самопроверки]] — вопросы по основным темам Linux.
- [[OS/Linux/Administration/Диагностика проблем]] — последовательность поиска неисправностей.

Рекомендуемый порядок: терминал → файловая система → пользователи и права → процессы и systemd → ресурсы и логи → сеть и SSH → практическое развёртывание приложения.
