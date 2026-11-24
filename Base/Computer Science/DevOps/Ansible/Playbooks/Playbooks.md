# Playbooks

[[Base/Computer Science/DevOps/Ansible/Ansible|Карта тем]] · [[Base/Computer Science/DevOps/Ansible/Основы/YAML|YAML]] · [[Base/Computer Science/DevOps/Ansible/Playbooks/Модули|Модули]] · [[Base/Computer Science/DevOps/Ansible/Roles/Роли|Роли]] · [[Base/Computer Science/DevOps/Ansible/Playbooks/Управление выполнением|Управление выполнением]]

**Playbook** — YAML-документ, содержащий **список plays**. Каждый play определяет целевые хосты и набор выполняемых задач.

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

## Два plays в одном файле

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

## Запуск

```bash
ansible-playbook -i inventory.ini site.yml
ansible-playbook -i inventory.ini site.yml --syntax-check
ansible-playbook -i inventory.ini site.yml --list-hosts
ansible-playbook -i inventory.ini site.yml --list-tasks
ansible-playbook -i inventory.ini site.yml --check --diff
ansible-playbook -i inventory.ini site.yml -K
```

`-K` (`--ask-become-pass`) запрашивает пароль для повышения привилегий, если он необходим. `--check` — проверочный режим; он не симулирует идеально все эффекты, особенно для командных модулей и при зависимостях между задачами.

## Повторное использование задач

Разбивать большой сценарий на части полезно для сопровождения, тестирования и повторного использования.

Обобщённый `ansible.builtin.include` [удалён из ansible-core](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/include_module.html). Для актуальных playbooks используются **отдельные явные конструкции**:

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

Документация: [Playbook Guide](https://docs.ansible.com/projects/ansible/latest/playbook_guide/index.html).
