---
tags:
  - devops/ansible
---

# Ansible Galaxy и коллекции

[[Base/Computer Science/DevOps/Ansible/Ansible|Карта тем]] · [[Base/Computer Science/DevOps/Ansible/Roles/Роли|Роли]] · [[Base/Computer Science/DevOps/Ansible/Playbooks/Модули|Модули]] · [[Base/Computer Science/DevOps/Ansible/Дополнительно/Собственные модули|Собственные модули]]

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
  - name: ansible.mariadb
```

```bash
ansible-galaxy collection install -r requirements.yml
```

**Не подключать незнакомую роль сразу к production.** Проверять её исходники, поддержку текущей ОС и Ansible, лицензии, активность проекта, сетевые запросы, системные изменения и работу с секретами.

Документация: [Ansible Galaxy](https://galaxy.ansible.com/).
