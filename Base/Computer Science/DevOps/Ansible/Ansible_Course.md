---
tags:
  - devops/ansible
  - linux/administration
---

# Ansible Course

> Конспект по курсу **«[[Ansible]] для самых маленьких»** (Rotoro.cloud), переработанный в техническую заметку.  
> Формат: теория → синтаксис → практические примеры → типичные ошибки.  
> Примеры ориентированы преимущественно на Linux и современные версии Ansible. Отдельно разобраны Windows и Arch Linux.
>
> **Примечание об актуальности (ноябрь 2026):** исходный курс содержит устаревшие инструменты и окружения (Atom IDE, CentOS Linux 7, старый синтаксис `include`, старые названия модулей). Здесь они объяснены с указанием современных альтернатив. Перед применением к production сверять документацию установленной версии Ansible.

## Содержание

- [1. Введение](#1-введение)
  - [Что такое Ansible и для чего он нужен](#11-что-такое-ansible-и-для-чего-он-нужен)
  - [Принцип работы и архитектура](#12-принцип-работы-и-архитектура)
  - [Преимущества, недостатки и границы применимости](#13-преимущества-недостатки-и-границы-применимости)
  - [Установка и подготовка окружения](#14-установка-и-подготовка-окружения)
  - [YAML: синтаксис и структуры данных](#15-yaml-синтаксис-и-структуры-данных)
- [2. Основные понятия](#2-основные-понятия)
  - [Inventory](#21-inventory)
  - [Управление Windows](#22-управление-windows)
  - [Playbooks](#23-playbooks)
  - [Modules](#24-modules)
  - [Идемпотентность](#25-идемпотентность)
  - [Variables и шаблоны](#26-variables-и-шаблоны)
  - [Редактор и инструменты разработки](#27-редактор-и-инструменты-разработки)
- [3. Механика Ansible](#3-механика-ansible)
  - [Conditionals](#31-conditionals)
  - [Loops](#32-loops)
  - [Handlers](#33-handlers)
  - [Повторное использование задач](#34-повторное-использование-задач)
  - [Roles](#35-roles)
  - [App Deploy: развёртывание приложения](#36-app-deploy-развёртывание-приложения)
- [4. Дополнительные темы](#4-дополнительные-темы)
  - [Ansible Galaxy и коллекции](#41-ansible-galaxy-и-коллекции)
  - [Паттерны выбора хостов](#42-паттерны-выбора-хостов)
  - [Dynamic inventory](#43-dynamic-inventory)
  - [Custom modules и плагины](#44-custom-modules-и-плагины)
  - [Ansible Vault и безопасность](#45-ansible-vault-и-безопасность)
  - [Отладка, тестирование и диагностика](#46-отладка-тестирование-и-диагностика)
  - [Дополнительные возможности исполнения](#47-дополнительные-возможности-исполнения)
  - [Best practices](#48-best-practices)
  - [Команды-шпаргалка](#49-команды-шпаргалка)
  - [Практические задания](#410-практические-задания)
  - [Источники](#411-источники)

---

# 1. Введение

## 1.1. Что такое Ansible и для чего он нужен

**Ansible** — инструмент автоматизации IT-инфраструктуры. Он позволяет описывать **желаемое состояние** серверов и инфраструктурных компонентов в виде файлов, а затем приводить реальные системы к этому состоянию.

Основные направления применения:

| Направление | Задачи |
|---|---|
| Configuration Management | Установка пакетов, конфигурация служб, пользователей, [[SSH и передача файлов|SSH]], firewall, файлов |
| Provisioning | Подготовка виртуальных машин, облачных ресурсов и сетевых компонентов через соответствующие коллекции/API |
| Application Deployment | Доставка приложения, настройка окружения, миграции, запуск сервисов |
| Orchestration | Координация действий между несколькими серверами в заданном порядке |
| Security / Compliance | Настройка политик, применение обновлений, проверка требуемых параметров |
| Operations | Регламентные работы, обслуживание, перезапуски, обновления |

Пример задачи: настроить 30 веб-серверов одинаково. Вместо ручных команд на каждом узле в playbook описываются пакеты, конфиги и состояния служб. [[Inventory]] определяет, на каких узлах выполнить работу.

**Ansible — не замена всем инструментам DevOps.** Terraform/OpenTofu удобнее для декларативного управления жизненным циклом облачных ресурсов; Ansible особенно силён в настройке ОС и приложений. В реальном проекте их часто используют совместно: сначала создать серверы, затем настроить их через Ansible.

## 1.2. Принцип работы и архитектура

```text
                    Control node
                  (Ansible + Python)
                          |
               inventory + playbooks
                          |
              +-----------+-----------+
              |           |           |
            SSH         SSH        WinRM/PSRP
              |           |           |
          Linux-01    Linux-02     Windows-01
             managed nodes / target hosts
```

**Control node** — управляющая машина, на которой установлен Ansible. **Managed node** — управляемая машина. На обычный Linux-хост Ansible обычно подключается через SSH, на Windows — через WinRM/PSRP либо поддерживаемую конфигурацию SSH.

Обычный сценарий выполнения:

1. Прочитать настройки Ansible, inventory и playbook.
2. Определить целевые хосты по полю `hosts`.
3. Установить соединение с управляемыми узлами.
4. При необходимости собрать **facts** (сведения об ОС, сети, архитектуре и т. д.).
5. Выполнить задачи, используя модули и плагины Ansible.
6. Вернуть результат каждой задачи: `ok`, `changed`, `failed`, `skipped`, `unreachable`.

**Agentless** означает, что на целевых системах обычно не нужен постоянно работающий специализированный агент Ansible. Это **не** означает отсутствие зависимостей: типичным Linux-модулям нужен совместимый Python на управляемом хосте; для отдельных действий используются другие механизмы. На Windows применяются PowerShell-модули.

Ansible, как правило, действует по **push-модели**: управляющая машина инициирует изменения. У него нет обязательной центральной базы состояния, аналогичной [[Terraform]] state. Проверка состояния производится при исполнении задач.

### Ключевые термины

| Термин | Назначение |
|---|---|
| `inventory` | Источник хостов, групп и параметров подключения |
| `playbook` | YAML-файл со сценарием автоматизации |
| `play` | Набор задач для определённых хостов |
| `task` | Одна логическая операция внутри play |
| `module` | Реализация операции: установить пакет, скопировать файл и т. д. |
| `variable` | Параметр сценария, различающий окружения/хосты |
| `fact` | Информация, собранная о целевой системе |
| `role` | Переиспользуемый компонент со стандартной структурой каталогов |
| `handler` | Задача, запускаемая по уведомлению при изменении |
| `collection` | Пакет ролей, модулей, плагинов и другого содержимого Ansible |
| `plugin` | Расширение поведения Ansible: соединение, inventory, lookup и др. |

## 1.3. Преимущества, недостатки и границы применимости

**Плюсы:**

- Понятные YAML-файлы, пригодные для Git и code review.
- Нет необходимости разворачивать собственный агент на каждом Linux-сервере.
- Широкая библиотека модулей и коллекций.
- Поддержка разных ОС, сетевых устройств, облаков и API.
- **Идемпотентные** модули позволяют безопасно повторять типовые операции.
- Управление многими хостами одной командой.
- Поддержка ролей, шаблонов, секретов, проверок и интеграции с CI/CD.

**Минусы и ограничения:**

- Требуются сетевой доступ и права на целевых системах.
- Сложные playbooks могут превращаться в плохо поддерживаемые сценарии.
- Обнаружение drift обычно происходит при запуске автоматизации, а не непрерывно.
- Для масштабной инфраструктуры нужно управлять параллелизмом, временем выполнения и отказами.
- Работа с Windows, специализированными сетевыми устройствами и облаками имеет собственные требования.
- `shell`/`command` сами по себе не дают желаемого уровня декларативности и идемпотентности.
- Неправильная автоматизация способна массово изменить или нарушить работу серверов.

**Когда применять:** настройка ОС после создания VM, развёртывание сервисов, поддержание общей конфигурации группы машин, повторяемые эксплуатационные процедуры.

## 1.4. Установка и подготовка окружения

### Установка Ansible на Arch Linux

```bash
sudo pacman -Syu
sudo pacman -S ansible

ansible --version
ansible-playbook --version
ansible-galaxy --version
```

Пакет `ansible` включает `ansible-core` и набор дополнительных коллекций, `ansible-core` — минимальный движок и базовые компоненты. Альтернатива для компактной установки:

```bash
sudo pacman -S ansible-core
```

Для изоляции версии Ansible в конкретном проекте можно использовать отдельное Python-окружение / `pipx`; не смешивать бесконтрольно системные и пользовательские установки.

### Что требуется на Linux-хосте

- SSH-сервер, доступный с control node.
- Пользователь с нужными правами; для системных изменений — работающий `sudo`/`become`.
- Совместимый Python для большинства модулей (за исключением некоторых bootstrap-сценариев, например `raw`).
- Доступ к репозиториям пакетов, если требуется их установка.

Пример проверки доступа **до Ansible**:

```bash
ssh deployer@192.0.2.10
ssh deployer@192.0.2.10 'python3 --version'
```

IP `192.0.2.10` — примерный адрес из диапазона для документации, его необходимо заменить.

Для аутентификации по ключу:

```bash
ssh-keygen -t ed25519 -C "ansible-control"
ssh-copy-id deployer@192.0.2.10
ssh deployer@192.0.2.10
```

Ключи предпочтительнее паролей в inventory. Публичный ключ добавляется на управляемый сервер; приватный остаётся у управляющей машины.

### Первый локальный проект

```text
ansible-lab/
├── ansible.cfg
├── inventory.ini
└── ping.yml
```

`inventory.ini`:

```ini
[local]
localhost ansible_connection=local
```

`ansible.cfg`:

```ini
[defaults]
inventory = ./inventory.ini
retry_files_enabled = False
host_key_checking = True
```

`ping.yml`:

```yaml
---
- name: Check local Ansible execution
  hosts: local
  gather_facts: false
  tasks:
    - name: Check connection
      ansible.builtin.ping:
```

Проверка:

```bash
ansible-inventory --graph
ansible local -m ansible.builtin.ping
ansible-playbook ping.yml
```

Модуль `ping` **не** отправляет ICMP-пакеты. Он проверяет способность Ansible выполнить модуль на целевой машине. Для Windows используется `ansible.windows.win_ping`.

## 1.5. YAML: синтаксис и структуры данных

**[[YAML]]** — человекочитаемый формат представления данных. В Ansible на YAML записываются playbooks, inventory, переменные, роли и шаблоны конфигурации. Это **не язык программирования**: алгоритмическое поведение обеспечивается Ansible и его модулями.

### Пары «ключ: значение» и словари (mapping / dictionary)

```yaml
service_name: nginx
service_port: 80
enabled: true

server:
  name: app01
  address: 192.0.2.10
  ssh_port: 22
```

`server` — **словарь** с именованными свойствами. Используется для описания одной сущности или набора связанных параметров.

### Списки (sequence / list / array)

```yaml
packages:
  - nginx
  - git
  - curl
```

Список применяется, когда есть **несколько элементов одного типа**. Порядок элементов сохраняется и обычно значим при последовательном исполнении задач.

### Список словарей

```yaml
servers:
  - name: app01
    address: 192.0.2.10
    role: web
  - name: db01
    address: 192.0.2.20
    role: database
```

Каждый элемент списка — отдельный словарь. Это удобно для серверов, пользователей, сервисов, firewall-правил и прочих структур со множеством полей.

### Словарь списков

```yaml
package_groups:
  web:
    - nginx
    - php-fpm
  database:
    - mariadb-server
    - python3-pymysql
```

Это удобная модель, когда параметры именованных категорий содержат несколько значений.

### Как выбирать структуру

| Задача | Подходящая структура |
|---|---|
| Один параметр | `key: value` |
| Параметры одного сервиса | Словарь |
| Набор пакетов | Список строк |
| Несколько серверов с параметрами | Список словарей |
| Группы серверов и их состав | Словарь списков |
| Сложное описание инфраструктуры | Вложенные словари и списки |

### Типы данных

```yaml
service: nginx              # строка
port: 8080                  # целое число
ratio: 0.75                 # число с плавающей точкой
enabled: true               # булево значение
optional_value: null        # отсутствующее значение
literal_string: "true"     # строка, не boolean
```

### Правила и типичные ошибки

1. Для вложенности используются **пробелы**, не табуляция. Обычно берут 2 пробела на уровень.
2. В `key: value` после двоеточия нужен пробел (когда двоеточие отделяет ключ от значения).
3. У элементов одного уровня должны совпадать отступы.
4. `-` означает элемент последовательности.
5. `#` начинает комментарий (вне строковых литералов).
6. `---` обозначает начало YAML-документа; часто используется для наглядности.
7. Строки с `: `, `#`, спецсимволами или неоднозначным содержимым лучше брать в кавычки.
8. Если значение начинается с `{{ ... }}`, заключать весь шаблон в кавычки.
9. Словари логически не зависят от порядка ключей; список задач — **зависит**.
10. Имена переменных Ansible рекомендуется писать в `snake_case`.

Пример **ошибки**:

```yaml
# Неверно: элементы списка имеют разные отступы.
packages:
  - nginx
   - git
```

Правильный вариант:

```yaml
packages:
  - nginx
  - git
```

**YAML, JSON, INI:** inventory Ansible часто пишут в INI, а playbooks — в YAML; JSON также является допустимой формой структурированных данных, но для чтения playbooks обычно менее удобен.

---

# 2. Основные понятия

## 2.1. Inventory

**Inventory** описывает, какими узлами управляет Ansible, как до них подключаться и к каким логическим группам они относятся.

Файл по умолчанию может находиться в `/etc/ansible/hosts`, но для проектов предпочтительнее хранить inventory **рядом с кодом** и передавать `-i` или задавать путь в `ansible.cfg`.

### INI inventory

`inventory.ini`:

```ini
[web]
app01 ansible_host=192.0.2.10
app02 ansible_host=192.0.2.11

[db]
db01 ansible_host=192.0.2.20

[linux:children]
web
db

[linux:vars]
ansible_user=deployer
ansible_port=22
ansible_connection=ssh
```

`app01` — **алиас** хоста; реальный адрес берётся из `ansible_host`. Группа `[linux:children]` включает другие группы. Встроенная группа `all` содержит все хосты.

### YAML inventory

`inventory.yml`:

```yaml
---
all:
  children:
    web:
      hosts:
        app01:
          ansible_host: 192.0.2.10
        app02:
          ansible_host: 192.0.2.11
    db:
      hosts:
        db01:
          ansible_host: 192.0.2.20
  vars:
    ansible_user: deployer
    ansible_port: 22
```

### Параметры соединения

| Переменная | Значение |
|---|---|
| `ansible_host` | IP/FQDN, используемый при подключении |
| `ansible_user` | Удалённый пользователь |
| `ansible_port` | Порт соединения |
| `ansible_connection` | `ssh`, `local`, `winrm`, `psrp`, другие плагины |
| `ansible_ssh_private_key_file` | Путь к приватному SSH-ключу |
| `ansible_python_interpreter` | Python на Linux-цели, если нужен явный путь |
| `ansible_become` | Разрешение повысить привилегии |
| `ansible_become_user` | Пользователь, от имени которого выполняется задача |
| `ansible_password` | Пароль подключения (хранить как секрет, не открытым текстом) |

**Важно:** Ansible **не обязан** подключаться под `root` по умолчанию. Если пользователь не задан, на него влияют конфигурация Ansible и пользователь, от имени которого запускается управление. Лучше явно задавать `ansible_user`.

### `group_vars` и `host_vars`

Переменные рекомендуется выносить из INI-строк:

```text
inventory/
├── hosts.yml
├── group_vars/
│   ├── all.yml
│   ├── web.yml
│   └── db.yml
└── host_vars/
    └── app01.yml
```

`inventory/group_vars/web.yml`:

```yaml
---
web_port: 8080
web_server_package: nginx
```

`inventory/host_vars/app01.yml`:

```yaml
---
web_port: 8081
```

Переменные хоста позволяют исключать частные случаи из универсального playbook. Итоговое значение определяется **правилами приоритета переменных**. Упрощённо: индивидуальные переменные хоста часто перекрывают переменные группы, но в Ansible есть дополнительные уровни приоритета (`play vars`, `role vars`, `-e` и др.). Не полагаться только на это упрощение.

### Команды для inventory

```bash
ansible-inventory -i inventory.ini --list
ansible-inventory -i inventory.ini --graph
ansible-inventory -i inventory.ini --host app01
ansible web -i inventory.ini --list-hosts
ansible web -i inventory.ini -m ansible.builtin.ping
ansible all -i inventory.ini -a 'hostname'
```

Последняя команда — **ad-hoc**: однократный запуск модуля без отдельного playbook. Модуль по умолчанию обычно `command`.

## 2.2. Управление Windows

Windows может выступать **managed node**, хотя обычная нативная Windows не используется как полноценный Ansible control node; для разработки на Windows применяют Linux-окружение, например WSL.

Варианты подключения:

- **WinRM** (`ansible_connection: winrm`) — классический путь через Windows Remote Management.
- **PSRP** (`ansible_connection: psrp`) — PowerShell Remoting Protocol поверх WinRM.
- **SSH** — поддерживаемый вариант для современных Windows и совместимых версий OpenSSH; официальная поддержка Windows over SSH в Ansible появилась в `ansible-core 2.18`.

Для WinRM/PSRP на управляющей машине нужны соответствующие Python-библиотеки (`pywinrm` или `pypsrp`), на Windows — настроенный remoting и подходящая аутентификация. В корпоративной сети предпочтительны Kerberos/сертификаты или другие согласованные защищённые механизмы. **Не включать бездумно Basic через HTTP и не отключать проверку TLS-сертификатов в production.**

Пример inventory (условные параметры):

```yaml
---
all:
  children:
    windows:
      hosts:
        win01:
          ansible_host: 192.0.2.30
      vars:
        ansible_connection: winrm
        ansible_port: 5986
        ansible_winrm_scheme: https
        ansible_winrm_transport: ntlm
        ansible_user: 'LAB\\ansible-user'
        # ansible_password: из Vault, не из открытого inventory
```

Пример модуля:

```yaml
---
- name: Configure Windows service
  hosts: windows
  gather_facts: false
  tasks:
    - name: Check Windows connectivity
      ansible.windows.win_ping:

    - name: Ensure Windows Time is running
      ansible.windows.win_service:
        name: W32Time
        state: started
```

При необходимости установить коллекции:

```bash
ansible-galaxy collection install ansible.windows
```

**Важно:** стандартные Python-модули для Linux нельзя автоматически запускать на Windows. Для Windows применяются специализированные модули PowerShell. Подготовка ОС и настройка WinRM/SSH — отдельный этап; Ansible сам по себе не является установщиком Windows на «голое железо».

## 2.3. Playbooks

**[[Playbooks|Playbook]]** — YAML-документ, содержащий **список plays**. Каждый play определяет целевые хосты и набор выполняемых задач.

```text
playbook.yml
└── Play (hosts: web)
    ├── Task: установить пакет
    ├── Task: скопировать конфигурацию
    └── Task: запустить сервис
```

Пример для Debian/Ubuntu:

```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  gather_facts: true

  tasks:
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Enable and start Nginx
      ansible.builtin.service:
        name: nginx
        enabled: true
        state: started
```

- `name` — читаемое имя play/task.
- `hosts` — целевая группа/хост/паттерн.
- `become: true` — повышение привилегий (например, через `sudo`).
- `gather_facts: true` — сбор сведений о системе.
- `tasks` — **упорядоченный** список задач.
- `ansible.builtin.apt` / `ansible.builtin.service` — имена модулей с указанием коллекции (**FQCN**).

При стандартной стратегии `linear` Ansible выполняет очередную задачу на хостах текущего play, затем переходит к следующей; степень параллельности ограничивается настройкой `forks`. Это не значит, что каждый сервер обязательно обрабатывается целиком по очереди.

### Два plays в одном файле

```yaml
---
- name: Configure database nodes
  hosts: db
  become: true
  tasks:
    - name: Install database server
      ansible.builtin.package:
        name: mariadb-server
        state: present

- name: Configure web nodes
  hosts: web
  become: true
  tasks:
    - name: Install Nginx
      ansible.builtin.package:
        name: nginx
        state: present
```

Порядок **plays** также имеет значение. Например, можно сначала подготовить БД, затем frontend.

### Запуск

```bash
ansible-playbook -i inventory.ini site.yml
ansible-playbook -i inventory.ini site.yml --syntax-check
ansible-playbook -i inventory.ini site.yml --list-hosts
ansible-playbook -i inventory.ini site.yml --list-tasks
ansible-playbook -i inventory.ini site.yml --check --diff
ansible-playbook -i inventory.ini site.yml -K
```

`-K` (`--ask-become-pass`) запрашивает пароль для повышения привилегий, если он необходим. `--check` — проверочный режим; он не симулирует идеально все эффекты, особенно для командных модулей и при зависимостях между задачами.

## 2.4. Modules

**Модуль** отвечает за конкретную операцию. Модули группируются по назначению: управление системой, файлами, пакетами, сетью, базами данных, облаками и Windows.

### Часто используемые модули

| Модуль | Для чего |
|---|---|
| `ansible.builtin.ping` | Проверить выполнение модуля на Linux-хосте |
| `ansible.builtin.command` | Запустить исполняемый файл без shell-интерпретации |
| `ansible.builtin.shell` | Выполнить команду через оболочку; пайпы, перенаправления и пр. |
| `ansible.builtin.script` | Передать локальный скрипт и выполнить на удалённом узле |
| `ansible.builtin.raw` | Отправить команду без обычной подсистемы модулей, например при bootstrap |
| `ansible.builtin.package` | Установить пакет через менеджер ОС |
| `ansible.builtin.apt` | Пакеты Debian/Ubuntu |
| `ansible.builtin.dnf` | Пакеты RPM-систем с DNF |
| `community.general.pacman` | Пакеты на Arch Linux (коллекция `community.general`) |
| `ansible.builtin.service` | Состояние службы |
| `ansible.builtin.systemd_service` | Управление systemd |
| `ansible.builtin.copy` | Скопировать файл или текст на хост |
| `ansible.builtin.template` | Создать файл по шаблону Jinja2 |
| `ansible.builtin.file` | Создать каталог, симлинк, изменить права и состояние файла |
| `ansible.builtin.lineinfile` | Управлять отдельной строкой файла |
| `ansible.builtin.replace` | Заменить фрагменты текста по регулярному выражению |
| `ansible.builtin.find` | Найти файлы на целевом хосте |
| `ansible.builtin.unarchive` | Распаковать архив |
| `ansible.builtin.user` / `group` | Управление пользователями и группами |
| `ansible.builtin.get_url` | Загрузить файл |
| `ansible.builtin.git` | Получить исходный код из Git |
| `ansible.builtin.debug` | Вывести отладочную информацию |
| `ansible.builtin.assert` | Проверить инварианты |
| `ansible.windows.win_copy` | Копирование файлов в Windows |
| `ansible.mysql.mysql_db` | Управление БД MySQL/MariaDB (коллекция `ansible.mysql`; в старых проектах `community.mysql`) |

**FQCN** (`namespace.collection.module`) — полное имя модуля. Для новых проектов лучше писать его явно, например `ansible.builtin.copy`, а не просто `copy`.

Изучение справки:

```bash
ansible-doc -l
ansible-doc ansible.builtin.copy
ansible-doc ansible.builtin.service
ansible-doc -s ansible.builtin.command
```

### `command` и `shell`

```yaml
- name: Read system release
  ansible.builtin.command:
    cmd: cat /etc/os-release
  register: os_release
  changed_when: false

- name: Show kernel information
  ansible.builtin.shell:
    cmd: uname -r | tr -d '\n'
  register: kernel_release
  changed_when: false
```

**Различие:** `command` не обрабатывает `|`, `>`, `&&`, подстановки shell как обычная оболочка; `shell` — обрабатывает. Для простых вызовов использовать `command`, а для операций, имеющих штатный модуль, — **штатный модуль**.

`command` также поддерживает:

```yaml
- name: Initialize only if marker is absent
  ansible.builtin.command:
    cmd: /usr/local/bin/init-app
    creates: /etc/demo-app/initialized
```

Параметр `creates` не даст повторно запускать команду, если файл уже существует. Это **частичный способ** добиться повторяемости, но лучше выбирать декларативный модуль.

### `script`

```yaml
- name: Run local maintenance script on host
  ansible.builtin.script:
    cmd: files/check-storage.sh --summary
  changed_when: false
```

Скрипт исходно лежит **на control node** и передаётся на управляемый хост. Зависимости и формат скрипта должны подходить целевой системе.

### Работа с файлами

```yaml
- name: Create application directory
  ansible.builtin.file:
    path: /opt/demo
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: Copy static configuration
  ansible.builtin.copy:
    src: files/app.conf
    dest: /etc/demo/app.conf
    owner: root
    group: root
    mode: '0644'

- name: Set one configuration line
  ansible.builtin.lineinfile:
    path: /etc/demo/app.conf
    regexp: '^log_level='
    line: 'log_level=info'
```

Для сложных конфигураций предпочтительнее `template`, а не десятки `lineinfile`. Не редактировать вручную автоматически генерируемые системой файлы без понимания их жизненного цикла (например, `/etc/resolv.conf` может управляться NetworkManager или systemd-resolved).

## 2.5. Идемпотентность

Операция **идемпотентна**, если повторное применение не меняет результат после достижения нужного состояния.

```yaml
- name: Ensure Nginx is started
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```

Если Nginx уже запущен и включён в автозагрузку — Ansible обычно вернёт `ok`, не делая лишних изменений. Если нет — исправит состояние и вернёт `changed`.

Сравнение:

```yaml
# Декларативно: обеспечить требуемое состояние.
- name: Ensure the package is installed
  ansible.builtin.package:
    name: nginx
    state: present

# Императивно: просто выполнить команду (поведение зависит от команды).
- name: Execute package command
  ansible.builtin.shell:
    cmd: some-install-command
```

Типовые значения `state`:

- `present` / `absent` — объект существует / отсутствует.
- `started` / `stopped` — служба запущена / остановлена.
- `restarted` — **принудительно** перезапустить службу; это действие **не идемпотентно в смысле отсутствия побочных эффектов**.
- `latest` — установить наиболее новую доступную версию пакета; повторный запуск после обновления репозиториев может внести новые изменения.

Идемпотентность зависит от конкретного модуля, его параметров и внешних условий. Не все операции Ansible идемпотентны.

## 2.6. Variables и шаблоны

Переменные нужны, чтобы разделять **логику сценария** и **данные окружения**.

```yaml
---
- name: Configure application
  hosts: web
  become: true
  vars:
    app_name: demo
    app_port: 8080
    app_directory: /opt/demo

  tasks:
    - name: Create application directory
      ansible.builtin.file:
        path: "{{ app_directory }}"
        state: directory
        mode: '0755'

    - name: Show selected port
      ansible.builtin.debug:
        msg: "{{ app_name }} listens on port {{ app_port }}"
```

### Способы задания переменных

- `vars` внутри play.
- `vars_files` для подключения отдельного YAML-файла.
- `group_vars` / `host_vars` для групп и отдельных хостов.
- `defaults/main.yml` и `vars/main.yml` внутри ролей.
- `register` для результата задачи.
- `set_fact` для вычисленных значений.
- Facts (`ansible_facts`) — информация о целевой ОС.
- `-e` / `--extra-vars` — переменные командной строки, обладающие очень высоким приоритетом.

Пример подключения файла:

```yaml
- name: Load application variables
  hosts: web
  vars_files:
    - vars/app.yml
  tasks:
    - name: Show application version
      ansible.builtin.debug:
        msg: "{{ app_version }}"
```

`vars/app.yml`:

```yaml
---
app_version: '1.4.2'
```

### Jinja2 templating

Для вставки переменной используется синтаксис `{{ variable }}`.

```yaml
app_bind: "{{ app_host }}:{{ app_port }}"
app_log_path: "/var/log/{{ app_name }}.log"
```

Фильтры:

```yaml
# Значение по умолчанию:
resolved_port: "{{ app_port | default(8080) }}"

# Преобразование к целому числу:
workers: "{{ cpu_count | int }}"

# Объединение строк:
full_path: "{{ base_dir }}/{{ app_name }}"
```

В `when` не нужны обрамляющие `{{ }}` — там указывается выражение напрямую.

### Facts

```yaml
- name: Print operating system information
  ansible.builtin.debug:
    msg: "{{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}"
```

Доступ к facts требует их сбора (`gather_facts: true`) или другого способа предварительно получить нужные данные.

### `register`

```yaml
- name: Check Nginx version
  ansible.builtin.command: nginx -v
  register: nginx_version
  changed_when: false
  failed_when: false

- name: Show result
  ansible.builtin.debug:
    var: nginx_version.stderr
```

Результат модуля часто содержит `stdout`, `stderr`, `rc`, `changed`, `failed` и другие поля (зависит от модуля). `nginx -v` выводит версию в `stderr`, поэтому показано именно это поле.

### Приоритет переменных

У Ansible сложная модель приоритета. **Общее практическое правило**: не определять одну и ту же переменную в пяти местах, явно разделять конфигурацию окружения и значения по умолчанию, учитывать, что `--extra-vars` перекрывает большинство других источников. Полная схема — в официальной документации.

## 2.7. Редактор и инструменты разработки

В исходном курсе упоминается **Atom IDE**. Atom был архивирован в декабре 2022 года и для новых проектов не рекомендуется.

Подходящие современные варианты:

- VS Code + расширения YAML и Ansible.
- Neovim / LazyVim + YAML LSP, ansible-lint, snippets.
- Любой редактор с подсветкой YAML и проверкой отступов.

Полезно проверять не только YAML-синтаксис, но и **Ansible-specific** правила: существование модулей, структуру задач, FQCN, потенциально опасные конструкции.

```bash
ansible-playbook --syntax-check site.yml
ansible-lint site.yml   # если ansible-lint установлен
```

---

# 3. Механика Ansible

## 3.1. Conditionals

**Conditionals** — условия, позволяющие выполнять задачу только при выполнении выражения. Основная директива — `when`.

### Простое условие

```yaml
- name: Install Nginx only on Debian family
  ansible.builtin.apt:
    name: nginx
    state: present
  when: ansible_facts['os_family'] == 'Debian'
```

### Логические операции

| Оператор | Значение |
|---|---|
| `==` | Равно |
| `!=` | Не равно |
| `>` / `<` / `>=` / `<=` | Сравнение |
| `and` | И |
| `or` | ИЛИ |
| `not` | НЕ |
| `in` / `not in` | Принадлежность коллекции |
| `is defined` | Переменная определена |

```yaml
when: ansible_facts['os_family'] in ['Debian', 'RedHat']
```

Несколько условий списком объединяются через `and`:

```yaml
when:
  - ansible_facts['os_family'] == 'Debian'
  - app_enabled | bool
```

### Выбор менеджера пакетов

У Ansible есть универсальный `package`, который выбирает пакетный менеджер автоматически, если нужная операция доступна. При необходимости тонкой настройки можно выбрать специализированный модуль:

```yaml
- name: Install Nginx on Debian-like hosts
  ansible.builtin.apt:
    name: nginx
    state: present
  when: ansible_facts['os_family'] == 'Debian'

- name: Install Nginx on RedHat-like hosts
  ansible.builtin.dnf:
    name: nginx
    state: present
  when: ansible_facts['os_family'] == 'RedHat'
```

**Не переносить автоматически старые примеры `yum` на новые платформы**: проверять доступный пакетный менеджер, его версию и фактическое имя пакета.

### Условие по результату предыдущей задачи

```yaml
- name: Check whether application directory exists
  ansible.builtin.stat:
    path: /opt/demo
  register: app_dir

- name: Create directory if absent
  ansible.builtin.file:
    path: /opt/demo
    state: directory
    mode: '0755'
  when: not app_dir.stat.exists
```

**Замечание:** пример показывает `when` + `register`, но сам модуль `file` уже идемпотентен. В обычном проекте дополнительная проверка `stat` здесь не требуется.

Другие удобные проверки:

```yaml
when: some_result.rc == 0
when: "'down' in service_status.stdout"
when: app_version is defined
when: package_enabled | bool
```

Не писать `when: "{{ app_enabled }}"`: `when` ожидает выражение, а не шаблон в фигурных скобках.

## 3.2. Loops

Циклы нужны для обработки набора однотипных элементов без копирования задач.

### Цикл по списку строк

```yaml
- name: Install common packages
  ansible.builtin.package:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - curl
    - unzip
```

На каждой итерации `item` принимает очередное значение. Для установки пакетов часто эффективнее **передать список непосредственно модулю**, а не создавать отдельную операцию на каждый пакет:

```yaml
- name: Install common packages in one operation
  ansible.builtin.package:
    name:
      - git
      - curl
      - unzip
    state: present
```

### Цикл по списку словарей

```yaml
- name: Create service accounts
  ansible.builtin.user:
    name: "{{ item.name }}"
    uid: "{{ item.uid }}"
    create_home: false
    shell: /usr/sbin/nologin
    state: present
  loop:
    - name: monitor
      uid: 2101
    - name: deploysvc
      uid: 2102
```

Теперь `item` — словарь, а `item.name` и `item.uid` — значения его полей.

### `loop` + `when`

```yaml
- name: Install selected packages
  ansible.builtin.package:
    name: "{{ item.name }}"
    state: present
  loop:
    - name: git
      required: true
    - name: htop
      required: false
    - name: curl
      required: true
  when: item.required | bool
```

### Управление именем переменной цикла

```yaml
- name: Create directories
  ansible.builtin.file:
    path: "{{ directory }}"
    state: directory
    mode: '0755'
  loop:
    - /opt/app
    - /opt/app/releases
  loop_control:
    loop_var: directory
    label: "{{ directory }}"
```

`loop_control.loop_var` помогает избегать конфликтов `item` во вложенных включениях.

### Старый синтаксис `with_*`

В старых playbooks встречается:

```yaml
with_items:
  - git
  - curl
```

Для новых сценариев обычно лучше использовать `loop`, а для сложных источников данных — lookup/query-плагины и фильтры. `with_*` — историческое семейство конструкций, его важно узнавать при чтении чужого кода.

## 3.3. Handlers

**Handler** — задача, которую Ansible выполняет **по уведомлению об изменении** (`notify`). Типичный случай: конфиг Nginx изменился → нужно проверить и перечитать/перезапустить службу.

```yaml
---
- name: Configure Nginx
  hosts: web
  become: true

  tasks:
    - name: Deploy Nginx configuration
      ansible.builtin.template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: '0644'
      notify: Reload Nginx

  handlers:
    - name: Reload Nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

Handler обычно запускается в конце соответствующего этапа обработки tasks, если был уведомлён; несколько уведомлений одного handler обычно сворачиваются в один запуск. Это избавляет от ненужных рестартов.

При необходимости вызвать ожидающие handlers раньше:

```yaml
- name: Apply pending handlers
  ansible.builtin.meta: flush_handlers
```

Для изменения конфигов полезно предварительно валидировать шаблон через параметр `validate` у `template`/`copy`, если приложение поддерживает проверку файла.

## 3.4. Повторное использование задач

Разбивать большой сценарий на части полезно для сопровождения, тестирования и повторного использования.

В старом курсе используется обобщённая директива `include`. В современных playbooks используются **отдельные явные конструкции**:

| Конструкция | Назначение |
|---|---|
| `ansible.builtin.import_playbook` | Статически импортировать playbook на верхнем уровне |
| `ansible.builtin.import_tasks` | Статически импортировать задачи |
| `ansible.builtin.include_tasks` | Динамически включить задачи во время выполнения |
| `ansible.builtin.import_role` | Статически подключить роль |
| `ansible.builtin.include_role` | Динамически подключить роль |
| `vars_files` | Подключить файлы переменных |
| `ansible.builtin.include_vars` | Загрузить переменные как задачу |

Пример:

```yaml
---
- name: Configure application nodes
  hosts: web
  become: true
  tasks:
    - name: Install packages from separate task file
      ansible.builtin.import_tasks: tasks/packages.yml

    - name: Configure only when required
      ansible.builtin.include_tasks: tasks/configure.yml
      when: app_configure | bool
```

`import_tasks` разбирается статически, а `include_tasks` — динамически во время исполнения. Это влияет на работу тегов, условий, циклов и `--list-tasks`. `import_playbook` применяется на верхнем уровне списка plays, а не как обычная task.

## 3.5. Roles

**Role** — переиспользуемый набор задач, переменных, обработчиков, файлов и шаблонов. Роли позволяют разделять автоматизацию на независимые функциональные части: `nginx`, `database`, `monitoring`, `firewall`, `application`.

### Типовая структура

```text
roles/
└── nginx/
    ├── defaults/
    │   └── main.yml
    ├── vars/
    │   └── main.yml
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── templates/
    │   └── nginx.conf.j2
    ├── files/
    │   └── index.html
    ├── meta/
    │   └── main.yml
    └── README.md
```

| Каталог | Назначение |
|---|---|
| `tasks/` | Основные операции роли |
| `handlers/` | Обработчики уведомлений |
| `defaults/` | Значения по умолчанию с низким приоритетом |
| `vars/` | Переменные роли с более высоким приоритетом |
| `templates/` | Файлы-шаблоны Jinja2 |
| `files/` | Статические файлы |
| `meta/` | Метаданные и зависимости роли |

**Создать каркас:**

```bash
ansible-galaxy role init roles/nginx
# Также встречается краткая историческая форма: ansible-galaxy init nginx
```

**Подключить в playbook:**

```yaml
---
- name: Configure application hosts
  hosts: web
  become: true
  roles:
    - nginx
    - application
```

В проектах роль обычно лежит в `./roles/` рядом с playbook. Другие пути можно задавать через `roles_path` в `ansible.cfg` или переменную окружения.

### Передача параметров роли

```yaml
roles:
  - role: nginx
    vars:
      nginx_listen_port: 8080
```

Нужно понимать различие между параметрами, определёнными в `defaults/main.yml` (удобны для переопределения) и `vars/main.yml` (перекрываются сложнее). **Настраиваемые значения рекомендуется помещать в `defaults`.**

### Преимущества ролей

- Стандартная структура и единая точка ответственности.
- Меньше копирования YAML между проектами.
- Удобное тестирование и публикация.
- Возможность собирать инфраструктурный стек из нескольких ролей.
- Возможность использовать проверенные роли сообщества после аудита.

## 3.6. App Deploy: развёртывание приложения

### Суть финального задания из исходного курса

Исходный курс предлагает развернуть веб-приложение для аренды недвижимости на **LEMP-стеке**:

| Слой | Компоненты | Что настраивается |
|---|---|---|
| L | Linux | ОС, пользователи, права, firewall |
| E | Nginx | Веб-сервер, virtual host, взаимодействие с PHP-FPM |
| M | MariaDB / MySQL | СУБД, база и учётная запись приложения |
| P | PHP / Laravel | PHP, зависимости, код приложения, конфигурация, миграции |

Предлагаемая архитектура:

```text
              HTTP/HTTPS
                  |
                Nginx
                  |
                PHP-FPM
                  |
              Laravel/PHP
                  |
                MariaDB
```

В курсе предусмотрено сначала разместить всё на **одном хосте**, а затем вынести БД на отдельный сервер. Роли для MySQL, Nginx/PHP и приложения предлагается переиспользовать; роль firewall написать самостоятельно.

**Актуализация:** оригинальный стенд использует **CentOS Linux 7**, поддержка которого закончилась 30 июня 2024 года. В современных лабораторных лучше выбрать поддерживаемую ОС. Старые внешние роли и команды из курса не следует считать работоспособными и безопасными без проверки зависимостей, версий и исходного кода. Ниже — самостоятельный минимальный вариант на **Ubuntu Server 24.04 LTS**.

### Практический вариант: рабочий LEMP с Ansible Roles

**Цель:** одним playbook на Ubuntu 24.04 LTS настроить MariaDB, PHP-FPM, Nginx и маленькое PHP-приложение, проверяющее соединение с БД. Этот пример специально упрощён: он показывает **архитектуру автоматизации**, а не является готовым production-шаблоном Laravel.

**Предусловия:** пустая Ubuntu 24.04 LTS, SSH-доступ пользователем `deployer` с sudo, Python 3, разрешённые сетевые подключения. В примере SSH работает на порту 22. Изменить имена, IP и параметры для собственного стенда.

#### 1. Структура проекта

```text
lemp-deploy/
├── ansible.cfg
├── inventory.yml
├── site.yml
├── vars.yml
├── secrets.yml                 # создаётся через ansible-vault
└── roles/
    ├── database/tasks/main.yml
    ├── php/tasks/main.yml
    ├── nginx/tasks/main.yml
    ├── nginx/handlers/main.yml
    ├── nginx/templates/demo.conf.j2
    ├── application/tasks/main.yml
    ├── application/templates/config.php.j2
    ├── application/templates/index.php.j2
    └── firewall/tasks/main.yml
```

#### 2. Конфигурация и inventory

`ansible.cfg`:

```ini
[defaults]
inventory = ./inventory.yml
roles_path = ./roles
host_key_checking = True
```

`inventory.yml`:

```yaml
---
all:
  children:
    lemp:
      hosts:
        app01:
          ansible_host: 192.0.2.10
          ansible_user: deployer
```

`vars.yml`:

```yaml
---
app_db_name: demoapp
app_db_user: demoapp
app_directory: /var/www/demo/public
php_fpm_socket: /run/php/php8.3-fpm.sock
ssh_port: 22
```

`php_fpm_socket` соответствует типичному PHP 8.3 из Ubuntu 24.04; в другой ОС или версии PHP проверь фактический сокет: `ls /run/php/`.

`secrets.yml` — **создать зашифрованным**, а не записывать пароль открытым текстом в Git:

```bash
ansible-vault create secrets.yml
```

Его содержимое **до шифрования**, вводимое в редакторе Vault:

```yaml
---
app_db_password: REPLACE_WITH_LONG_RANDOM_SECRET
```

#### 3. Главный playbook

`site.yml`:

```yaml
---
- name: Deploy demo LEMP application
  hosts: lemp
  become: true
  gather_facts: true
  vars_files:
    - vars.yml
    - secrets.yml
  roles:
    - database
    - php
    - nginx
    - application
    - firewall
```

Порядок ролей здесь осознанный: сначала БД и PHP-FPM, затем веб-сервер, файлы приложения и firewall.

#### 4. Роль `database`

`roles/database/tasks/main.yml`:

```yaml
---
- name: Install MariaDB and Python driver
  ansible.builtin.apt:
    name:
      - mariadb-server
      - python3-pymysql
    state: present
    update_cache: true

- name: Ensure MariaDB is enabled and started
  ansible.builtin.service:
    name: mariadb
    enabled: true
    state: started

- name: Create application database
  ansible.mysql.mysql_db:
    name: "{{ app_db_name }}"
    state: present
    login_unix_socket: /run/mysqld/mysqld.sock

- name: Create database user with limited permissions
  ansible.mysql.mysql_user:
    name: "{{ app_db_user }}"
    password: "{{ app_db_password }}"
    host: 127.0.0.1
    priv: "{{ app_db_name }}.*:SELECT,INSERT,UPDATE,DELETE"
    state: present
    login_unix_socket: /run/mysqld/mysqld.sock
  no_log: true
```

Для модулей из отдельной коллекции установить на **control node** (современное пространство имён `ansible.mysql`; в старых материалах встречается `community.mysql`):

```bash
ansible-galaxy collection install ansible.mysql
```

Роль MariaDB использует локальную unix-сокет-аутентификацию администратора; она подходит для типичного свежего Ubuntu-стенда при выполнении задач с `become: true`. Для уже настроенной БД или другого дистрибутива параметры подключения могут отличаться.

#### 5. Роль `php`

`roles/php/tasks/main.yml`:

```yaml
---
- name: Install PHP-FPM and MariaDB driver
  ansible.builtin.apt:
    name:
      - php-fpm
      - php-mysql
    state: present

- name: Ensure PHP-FPM is enabled and running
  ansible.builtin.service:
    name: php8.3-fpm
    enabled: true
    state: started
```

Для Ubuntu 24.04 обычно используется PHP-FPM 8.3; на другой версии Ubuntu адаптировать имя systemd-unit и путь к сокету.

#### 6. Роль `nginx`

`roles/nginx/tasks/main.yml`:

```yaml
---
- name: Install Nginx
  ansible.builtin.apt:
    name: nginx
    state: present

- name: Disable default Nginx site
  ansible.builtin.file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  notify: Reload Nginx

- name: Deploy application virtual host
  ansible.builtin.template:
    src: demo.conf.j2
    dest: /etc/nginx/sites-available/demo.conf
    owner: root
    group: root
    mode: '0644'
  notify: Reload Nginx

- name: Enable application virtual host
  ansible.builtin.file:
    src: /etc/nginx/sites-available/demo.conf
    dest: /etc/nginx/sites-enabled/demo.conf
    state: link
  notify: Reload Nginx

- name: Ensure Nginx is enabled and running
  ansible.builtin.service:
    name: nginx
    enabled: true
    state: started
```

`roles/nginx/templates/demo.conf.j2`:

```nginx
server {
    listen 80 default_server;
    server_name _;
    root {{ app_directory }};
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:{{ php_fpm_socket }};
    }

    location ~ /\. {
        deny all;
    }
}
```

`roles/nginx/handlers/main.yml`:

```yaml
---
- name: Reload Nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded
```

Для более безопасного обновления конфигурации Nginx в серьёзном проекте добавить предварительную проверку `nginx -t` перед reload и продумать порядок первого запуска.

#### 7. Роль `application`

`roles/application/tasks/main.yml`:

```yaml
---
- name: Create application document root
  ansible.builtin.file:
    path: "{{ app_directory }}"
    state: directory
    owner: root
    group: www-data
    mode: '0755'

- name: Create private application configuration directory
  ansible.builtin.file:
    path: /etc/demo-app
    state: directory
    owner: root
    group: www-data
    mode: '0750'

- name: Deploy private database configuration
  ansible.builtin.template:
    src: config.php.j2
    dest: /etc/demo-app/config.php
    owner: root
    group: www-data
    mode: '0640'
  no_log: true

- name: Deploy demo PHP application
  ansible.builtin.template:
    src: index.php.j2
    dest: "{{ app_directory }}/index.php"
    owner: root
    group: www-data
    mode: '0644'
```

`roles/application/templates/config.php.j2`:

```php
<?php
return [
    'host' => '127.0.0.1',
    'database' => {{ app_db_name | to_json }},
    'username' => {{ app_db_user | to_json }},
    'password' => {{ app_db_password | to_json }},
];
```

`roles/application/templates/index.php.j2`:

```php
<?php
$config = require '/etc/demo-app/config.php';

try {
    $dsn = sprintf('mysql:host=%s;dbname=%s;charset=utf8mb4',
        $config['host'], $config['database']);
    $db = new PDO($dsn, $config['username'], $config['password'], [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    ]);
    $db->query('SELECT 1');
    header('Content-Type: text/plain; charset=UTF-8');
    echo "LEMP application OK; database connection OK\n";
} catch (Throwable $error) {
    http_response_code(503);
    header('Content-Type: text/plain; charset=UTF-8');
    echo "Application is not ready\n";
}
```

Пароль находится вне публичного webroot. Для production лучше подключать секреты через специализированные secret-management решения и защищённые переменные окружения / файловые монтирования. **Нельзя** оставлять необработанные секреты в HTML, логах или stdout задачи.

#### 8. Роль `firewall`

Ниже пример **только для Ubuntu с UFW**. В исходном курсе используется `firewalld`, что зависит от ОС. Задача firewall — оставить доступным SSH и открыть HTTP.

`roles/firewall/tasks/main.yml`:

```yaml
---
- name: Install UFW
  ansible.builtin.apt:
    name: ufw
    state: present

- name: Allow SSH before enabling firewall
  community.general.ufw:
    rule: allow
    port: "{{ ssh_port }}"
    proto: tcp

- name: Allow HTTP
  community.general.ufw:
    rule: allow
    port: '80'
    proto: tcp

- name: Enable UFW
  community.general.ufw:
    state: enabled
```

Зависимость:

```bash
ansible-galaxy collection install community.general
```

**Предупреждение:** если SSH использует другой порт, есть IPv6-особенности, отдельные правила доступа или облачный firewall, предварительно адаптировать политику. Ошибка в firewall может оборвать доступ к серверу. В production безопаснее проверять правила и доступ через независимый канал управления.

#### 9. Запуск и проверка

```bash
ansible-galaxy collection install ansible.mysql community.general
ansible-inventory --graph
ansible lemp -m ansible.builtin.ping
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --ask-vault-pass
```

После запуска:

```bash
curl -fsS http://192.0.2.10/
# Ожидаемый ответ:
# LEMP application OK; database connection OK
```

На целевой машине:

```bash
sudo systemctl status mariadb
sudo systemctl status nginx
sudo systemctl status php8.3-fpm
sudo nginx -t
sudo ss -ltnp | grep ':80'
```

Запусти playbook **повторно**: большинство задач должны показать `ok`, а не `changed`, если конфигурация не изменялась. Изменение Nginx-шаблона должно уведомить handler.

#### 10. Как перейти от демо к Laravel и двум серверам

Для реального Laravel-проекта дополнительно потребуется:

1. Получить исходники из Git или артефакта CI; желательно фиксировать конкретный release/tag/commit.
2. Установить необходимые PHP-модули и Composer; управлять версиями зависимостей.
3. Создать `.env` безопасно, не раскрывая секреты в Git/logs.
4. Настроить права на `storage/` и `bootstrap/cache`.
5. Выполнить `composer install --no-dev`, команды `php artisan ...` и контролируемые миграции.
6. Настроить Nginx для `public/`, PHP-FPM и HTTPS.
7. При выносе БД в `db01` заменить `127.0.0.1` на **внутренний** адрес БД, ограничить её сетевой доступ и отдельно настроить пользователей/привилегии.
8. Продумать отказоустойчивость, миграции, резервное копирование и возврат к предыдущему релизу.

**Смысл финального задания:** не набор конкретных команд для CentOS 7, а разделение автоматизации на независимые роли и правильное назначение ролей хостам.

---

# 4. Дополнительные темы

## 4.1. Ansible Galaxy и коллекции

**Ansible Galaxy** — каталог сообщества и инструмент `ansible-galaxy` для работы с ролями и коллекциями.

Роль — структурированный набор tasks, handlers, templates и других файлов. **Collection** — более крупная единица доставки, включающая роли, модули, плагины и документацию.

Команды:

```bash
# Найти роли (поиск по Galaxy).
ansible-galaxy role search nginx

# Установить роль, пример имени следует заменить реальным.
ansible-galaxy role install namespace.role_name

# Установить роль в каталог проекта.
ansible-galaxy role install namespace.role_name -p ./roles

# Установить коллекцию.
ansible-galaxy collection install community.general

# Список ролей / коллекций.
ansible-galaxy role list
ansible-galaxy collection list
```

Для воспроизводимости фиксировать зависимости в `requirements.yml` и, где возможно, их версии:

```yaml
---
collections:
  - name: community.general
  - name: ansible.mysql
```

```bash
ansible-galaxy collection install -r requirements.yml
```

**Не подключать незнакомую роль сразу к production.** Проверять её исходники, поддержку текущей ОС и Ansible, лицензии, активность проекта, сетевые запросы, системные изменения и работу с секретами.

## 4.2. Паттерны выбора хостов

В `hosts` и некоторых CLI-операциях можно задавать **patterns** — выражения для выбора целевых машин.

| Паттерн | Значение |
|---|---|
| `web` | Вся группа `web` |
| `app01` | Один хост |
| `all` | Все хосты |
| `web:db` | Объединение групп (OR) |
| `web:&production` | Пересечение групп (AND) |
| `web:!maintenance` | Веб-хосты кроме группы maintenance |
| `app*` | Хосты по шаблону имени |

```bash
ansible 'web:&production' -i inventory.ini --list-hosts
ansible 'web:!maintenance' -i inventory.ini --list-hosts
ansible-playbook -i inventory.ini site.yml --limit app01
```

Для сложных паттернов в shell использовать одинарные кавычки, чтобы оболочка не интерпретировала `*`, `!` и другие спецсимволы.

Поле `hosts` можно задавать как строку-паттерн; важно не путать с параметром `--limit`, который **дополнительно сужает** множество хостов, выбранное в playbook.

## 4.3. Dynamic inventory

**Static inventory** — зафиксированный список хостов в INI/YAML. **Dynamic inventory** получает актуальные адреса и группы из внешнего источника: облака, гипервизора, CMDB, API.

Применяется, когда инфраструктура часто меняется: VM создаются и удаляются, адреса выдаются автоматически, группы определяются метками.

Сейчас предпочтительно использовать **inventory plugins** поддерживаемых коллекций, например для AWS, Azure, VMware и других платформ, вместо самописных скриптов, если есть готовый плагин.

Концептуально:

```text
Cloud / CMDB / API
       |
 inventory plugin
       |
 Ansible inventory
       |
      playbook
```

Команды для проверки:

```bash
ansible-inventory -i cloud_inventory.yml --graph
ansible-inventory -i cloud_inventory.yml --list
ansible-playbook -i cloud_inventory.yml site.yml
```

Исторически dynamic inventory часто реализовывали исполняемыми скриптами Python, возвращающими JSON. Такой вариант по-прежнему полезен для специальных интеграций, но имеет больше кода и ответственности по поддержке.

## 4.4. Custom modules и плагины

Если нет подходящего модуля, можно написать **собственный модуль** (часто на Python для Linux-целей). Он должен принимать аргументы, возвращать структурированный результат и правильно сообщать `changed` / `failed`.

Минимальная идея:

```python
from ansible.module_utils.basic import AnsibleModule


def main():
    module = AnsibleModule(
        argument_spec={
            "name": {"type": "str", "required": True},
        },
        supports_check_mode=True,
    )
    name = module.params["name"]
    module.exit_json(changed=False, message=f"Checked {name}")


if __name__ == "__main__":
    main()
```

Это **демонстрация интерфейса**, а не модуль, который действительно конфигурирует сервис. Полноценный модуль должен уметь получать текущее состояние, рассчитывать изменения, правильно работать в check mode, валидировать параметры и возвращать корректные ошибки.

**Plugins** — иной механизм расширения: inventory, connection, lookup, callback, filter и другие. Нужный тип расширения выбирается по задаче; не всякая автоматизация требует написания собственного модуля.

## 4.5. Ansible Vault и безопасность

**[[Ansible Vault]]** шифрует данные в репозитории: пароли, API-ключи, приватные параметры. **Он защищает данные в хранилище, но не отменяет риск их утечки во время выполнения**.

```bash
ansible-vault create secrets.yml
ansible-vault edit secrets.yml
ansible-vault encrypt secrets.yml
ansible-vault view secrets.yml
ansible-playbook site.yml --ask-vault-pass
```

Если secrets уже использованы в task, применять `no_log: true`, тщательно настраивать доступ к выводам CI/CD и не отправлять секреты в `debug`, callback-логи и отчёты.

Рекомендуемые меры:

- SSH-ключи и проверка host keys вместо паролей в открытом inventory.
- Минимальные права пользователя/сервисного аккаунта.
- `become` только там, где требуется.
- Отдельный inventory для test/staging/production.
- Защищённый Vault/secret manager; не коммитить пароли, приватные ключи и расшифрованные secrets.
- Проверка сторонних ролей и их версий.
- План восстановления перед массовыми изменениями.
- Не выключать глобально `host_key_checking` ради удобства: это снижает защиту SSH-соединений.

## 4.6. Отладка, тестирование и диагностика

### Полезные режимы

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check
ansible-playbook site.yml --check --diff
ansible-playbook site.yml -v
ansible-playbook site.yml -vvv
ansible-playbook site.yml --start-at-task 'Install Nginx'
ansible-playbook site.yml --step
ansible-playbook site.yml --list-tags
```

`--check` **не гарантирует** полноценную проверку на свежем хосте: задачи, которым требуются созданные ранее файлы/пакеты, могут быть пропущены или завершиться ошибкой. Проверять изменения на отдельном тестовом стенде.

### `failed_when` и `changed_when`

```yaml
- name: Read service state
  ansible.builtin.command: systemctl is-active nginx
  register: nginx_state
  changed_when: false
  failed_when: nginx_state.rc not in [0, 3]
```

**Назначение:** `failed_when` задаёт критерий ошибки, `changed_when` — критерий изменения. Не использовать `ignore_errors: true` как универсальный способ «починить» сломанную автоматизацию.

### `block` / `rescue` / `always`

```yaml
- name: Attempt deployment
  block:
    - name: Run deployment command
      ansible.builtin.command:
        cmd: /usr/local/bin/deploy-demo
  rescue:
    - name: Report deployment failure
      ansible.builtin.debug:
        msg: Deployment failed
  always:
    - name: Record deployment completion
      ansible.builtin.debug:
        msg: Deployment attempt finished
```

`rescue` полезен для обработки ошибок, однако не даёт автоматического **transaction rollback**. Откат приложения и БД нужно проектировать отдельно.

### Основные причины ошибок

| Ошибка / симптом | Что проверить |
|---|---|
| `UNREACHABLE` | IP, DNS, маршрут, firewall, порт SSH, ключ, `ansible_user` |
| `Permission denied` | SSH-ключи/пароль, sudo, `become` |
| `No module named ...` | Python и библиотеки на нужной стороне (controller или target) |
| `Could not find ... module` | Установлена ли коллекция; правильно ли написан FQCN |
| `YAML parsing error` | Отступы, двоеточия, табы, кавычки |
| `Undefined variable` | Имя, область видимости, файлы переменных, facts |
| `No hosts matched` | Inventory, паттерны, `hosts`, `--limit` |
| Служба не запускается | Имя службы, конфиг, журналы systemd |
| Повторный запуск всегда `changed` | `shell`/`command`, некорректные параметры, нестабильный шаблон |

### Диагностика от контроллера к целевому хосту

```bash
ansible-inventory -i inventory.yml --host app01
ssh deployer@192.0.2.10
ansible app01 -i inventory.yml -m ansible.builtin.ping -vvv
ansible app01 -i inventory.yml -m ansible.builtin.setup
```

## 4.7. Дополнительные возможности исполнения

### Теги

```yaml
- name: Install Nginx
  ansible.builtin.package:
    name: nginx
    state: present
  tags:
    - packages
    - web
```

```bash
ansible-playbook site.yml --tags packages
ansible-playbook site.yml --skip-tags packages
```

### Rolling update с `serial`

```yaml
- name: Update application hosts in batches
  hosts: web
  serial: 1
  tasks:
    - name: Ensure application package is present
      ansible.builtin.package:
        name: nginx
        state: present
```

`serial: 1` обрабатывает хосты по одному в пределах play (но сама по себе не гарантирует zero-downtime: нужна интеграция с load balancer и health checks).

### Делегирование и `run_once`

```yaml
- name: Check control node context
  ansible.builtin.debug:
    msg: "Preparing deployment for {{ inventory_hostname }}"
  delegate_to: localhost
```

`delegate_to` перенаправляет выполнение подходящей задачи на другой узел. `run_once: true` применяется, когда операция должна быть выполнена один раз в пределах соответствующей группы выполнения; с `serial` и пакетным исполнением нужно учитывать нюансы.

### Параллелизм

```bash
ansible-playbook site.yml --forks 10
ansible-playbook site.yml --limit 'web:&staging'
```

`forks` регулирует число одновременных worker-процессов на controller. Для задач с общими внешними ресурсами учитывать гонки и ограничения API.

## 4.8. Best practices

1. **Декларативность:** штатные модули вместо длинных `shell`/`command` там, где это возможно.
2. **Идемпотентность:** второй запуск без изменений должен давать преимущественно `ok`.
3. **Структура:** одна роль — одна ограниченная зона ответственности.
4. **Параметризация:** настройки в `group_vars`/`host_vars`/`defaults`, а не в коде tasks.
5. **Секреты:** Vault или внешний secret manager; `no_log` на чувствительных операциях.
6. **Предсказуемость:** фиксировать поддерживаемые версии ОС, зависимостей, коллекций, образов.
7. **Читаемость:** содержательные `name` для каждой task, единый стиль YAML, FQCN.
8. **Изменения служб:** `notify` + handlers вместо безусловных рестартов.
9. **Безопасные изменения:** сначала стенд, `--syntax-check`, `ansible-lint`, затем staging и production.
10. **Минимальные права:** целевые пользователи и права `become` только по необходимости.
11. **Проверка результата:** после конфигурации использовать health checks и `assert`.
12. **Управление риском:** для БД — бэкапы, для приложений — rollback, для сети/firewall — резервный доступ.
13. **Окружения:** отдельно хранить inventory для dev, staging и prod; не путать параметры.
14. **Коллекции:** не считать роли Galaxy безопасными только потому, что они опубликованы.
15. **Git:** хранить playbooks, шаблоны и requirements в репозитории, проводить code review.

**Антипаттерн `bashsible`:** большой playbook из десятков задач вида `shell: apt install ...`, `shell: echo ... >> file`, `shell: systemctl restart ...`. Это теряет значительную часть преимуществ Ansible.

## 4.9. Команды-шпаргалка

| Команда | Назначение |
|---|---|
| `ansible --version` | Версия Ansible |
| `ansible-inventory -i inventory.yml --graph` | Дерево хостов и групп |
| `ansible all -m ansible.builtin.ping` | Проверка управляемых Linux-хостов |
| `ansible web -a 'hostname'` | Ad-hoc команда |
| `ansible-playbook site.yml` | Запуск playbook |
| `ansible-playbook site.yml --syntax-check` | Проверка синтаксиса |
| `ansible-playbook site.yml --check --diff` | Проверка предполагаемых изменений |
| `ansible-playbook site.yml --limit app01` | Ограничить целевые хосты |
| `ansible-playbook site.yml --tags web` | Выбранные задачи по тегам |
| `ansible-playbook site.yml -K` | Запрос пароля sudo/become |
| `ansible-playbook site.yml -vvv` | Подробный debug вывод |
| `ansible-doc ansible.builtin.copy` | Справка по модулю |
| `ansible-galaxy role init roles/myrole` | Создать роль |
| `ansible-galaxy collection list` | Посмотреть коллекции |
| `ansible-vault create secrets.yml` | Создать зашифрованный файл |
| `ansible-config dump --only-changed` | Активные переопределения настроек |
| `ansible-lint site.yml` | Проверка стиля/практик, если установлен ansible-lint |

## 4.10. Практические задания

Задачи возрастают по сложности; рекомендовано выполнять их на отдельных VM/контейнерах и фиксировать результаты в Git.

**Уровень 1 — основы:**

1. Создать inventory с `localhost`, локально выполнить `ansible.builtin.ping`.
2. Добавить два Linux-хоста, объединить в группы `web` и `db`; посмотреть `--graph` и `--list-hosts`.
3. Написать playbook, который устанавливает `git`, `curl` и создаёт каталог `/opt/lab`.
4. Вынести имя каталога и список пакетов в переменные.
5. Запустить playbook дважды и сравнить `ok` / `changed`.

**Уровень 2 — механика:**

6. Использовать `when` для разных семейств Linux.
7. Создать нескольких сервисных пользователей через `loop` и список словарей.
8. Настроить Nginx через `template`, добавить handler для reload.
9. Вынести параметры для `dev` и `staging` в `group_vars`.
10. Разбить большой playbook на tasks и role, проверить `--syntax-check`, `--check --diff`.

**Уровень 3 — проект:**

11. Развернуть базовый LEMP из раздела 3.6 на одной Ubuntu VM.
12. Разделить приложение и MariaDB на два узла. Ограничить БД локальной сетью и настроить права.
13. Добавить зашифрованный Vault, `no_log`, smoke test через `uri`/`curl` и тест повторяемости.
14. Настроить `serial` для обновления нескольких frontend-узлов без одновременного перезапуска всех машин.
15. Добавить `ansible-lint` в CI и документировать процедуру rollback.

## 4.11. Источники

**Основной материал:** PDF курса «Ansible для самых маленьких», Rotoro.cloud (предоставлен с запросом). Темы курса: обзор, установка, YAML, inventory, playbooks, modules, variables, conditionals, loops, roles, deployment LEMP, Windows, Galaxy, patterns, dynamic inventory, custom modules.

**Актуальная документация:**

- [Ansible Documentation](https://docs.ansible.com/projects/ansible/latest/)
- [Установка на разных ОС, включая Arch Linux](https://docs.ansible.com/projects/ansible/latest/installation_guide/installation_distros.html)
- [Inventory Guide](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html)
- [Playbook Guide](https://docs.ansible.com/projects/ansible/latest/playbook_guide/index.html)
- [Module Index](https://docs.ansible.com/projects/ansible/latest/collections/index_module.html)
- [Variable Precedence](https://docs.ansible.com/projects/ansible/latest/reference_appendices/general_precedence.html)
- [Roles and Reuse](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
- [Windows Management](https://docs.ansible.com/projects/ansible/latest/os_guide/intro_windows.html)
- [Windows SSH](https://docs.ansible.com/projects/ansible/latest/os_guide/windows_ssh.html)
- [Ansible Vault](https://docs.ansible.com/projects/ansible/latest/vault_guide/vault.html)
- [Check Mode / Diff Mode](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_checkmode.html)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Ansible Lint](https://ansible.readthedocs.io/projects/lint/)
- [CentOS Linux 7 EOL (Red Hat)](https://www.redhat.com/en/topics/linux/centos-linux-eol)
- [Atom sunset (GitHub)](https://github.blog/news-insights/product-news/sunsetting-atom/)

---

> **Главная идея Ansible:** описывать **желаемое состояние**, разделять **инвентарь, параметры и действия**, собирать переиспользуемую автоматизацию в **роли**, а перед применением к рабочим системам проверять её **повторяемость, безопасность и результат**.
