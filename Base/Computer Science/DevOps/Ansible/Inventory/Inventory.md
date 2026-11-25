---
tags:
  - devops/ansible
---

# Inventory

[[Base/Computer Science/DevOps/Ansible/Ansible|Карта тем]] · [[Base/Computer Science/DevOps/Ansible/Inventory/Подключение к узлам|Подключение к узлам]] · [[Base/Computer Science/DevOps/Ansible/Inventory/Паттерны и динамический Inventory|Паттерны и динамический Inventory]] · [[Base/Computer Science/DevOps/Ansible/Playbooks/Переменные и шаблоны|Переменные и шаблоны]]

**Inventory** описывает, какими узлами управляет Ansible, как до них подключаться и к каким логическим группам они относятся.

Файл по умолчанию может находиться в `/etc/ansible/hosts`, но для проектов предпочтительнее хранить inventory **рядом с кодом** и передавать `-i` или задавать путь в `ansible.cfg`.

## INI inventory

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

## YAML inventory

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

## `group_vars` и `host_vars`

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

Переменные хоста позволяют исключать частные случаи из универсального [[Playbooks|playbook]]. Итоговое значение определяется **правилами приоритета переменных**. Упрощённо: индивидуальные переменные хоста часто перекрывают переменные группы, но в Ansible есть дополнительные уровни приоритета (`play vars`, `role vars`, `-e` и др.). Не полагаться только на это упрощение.

## Команды для inventory

```bash
ansible-inventory -i inventory.ini --list
ansible-inventory -i inventory.ini --graph
ansible-inventory -i inventory.ini --host app01
ansible web -i inventory.ini --list-hosts
ansible web -i inventory.ini -m ansible.builtin.ping
ansible all -i inventory.ini -a 'hostname'
```

Последняя команда — **ad-hoc**: однократный запуск модуля без отдельного playbook. Модуль по умолчанию обычно `command`.

Документация: [Inventory Guide](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html).
