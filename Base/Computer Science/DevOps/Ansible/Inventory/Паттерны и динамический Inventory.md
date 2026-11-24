# Паттерны и динамический Inventory

[[Base/Computer Science/DevOps/Ansible/Ansible|Карта тем]] · [[Base/Computer Science/DevOps/Ansible/Inventory/Inventory|Inventory]] · [[Base/Computer Science/DevOps/Ansible/Playbooks/Управление выполнением|Управление выполнением]]

## Паттерны выбора хостов

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

## Статический и динамический Inventory

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
