---
tags:
  - devops
  - ansible
status: todo
---

# Ansible

Ansible — agentless-инструмент управления конфигурациями: управляющий узел подключается к целевым хостам по SSH и приводит их к описанному состоянию.

## Что нужно знать

- ad hoc-команды;
- inventory в INI/YAML;
- playbook и его структура;
- модули (`apt`, `copy`, `template`, `file`, `service`, `user`, `git`);
- variables и приоритеты переменных;
- facts;
- handlers;
- `when`, `loop`;
- шаблоны Jinja2;
- подключение по SSH-ключам;
- Ansible Vault.

Идемпотентность: [[Base/Computer Science/DevOps/Ansible/Основы/Обзор Ansible#Идемпотентность|повторяемость и состояния модулей]]. Роли: [[Base/Computer Science/DevOps/Ansible/Roles/Роли|Роли]].

## Практика

- [ ] Написать playbook для первичной настройки сервера.
- [ ] Вынести установку и конфигурацию сервиса в role.
- [ ] Проверить идемпотентность повторным запуском.
- [ ] Спрятать тестовый секрет через Ansible Vault.

## Собеседование

- [ ] Что такое playbook и role?
- [ ] Что такое handler?
- [ ] Как работают приоритеты переменных?
- [ ] Что означает идемпотентность?

## Навигация по заметкам

### Основы

- [[Base/Computer Science/DevOps/Ansible/Основы/Обзор Ansible|Обзор Ansible]]
- [[Base/Computer Science/DevOps/Ansible/Основы/Установка и настройка|Установка и настройка]]
- [[Base/Computer Science/DevOps/Ansible/Основы/YAML|YAML]]

### Inventory

- [[Base/Computer Science/DevOps/Ansible/Inventory/Inventory|Inventory]]
- [[Base/Computer Science/DevOps/Ansible/Inventory/Подключение к узлам|Подключение к узлам]]
- [[Base/Computer Science/DevOps/Ansible/Inventory/Паттерны и динамический Inventory|Паттерны и динамический Inventory]]

### Playbooks

- [[Base/Computer Science/DevOps/Ansible/Playbooks/Playbooks|Playbooks]]
- [[Base/Computer Science/DevOps/Ansible/Playbooks/Модули|Модули]]
- [[Base/Computer Science/DevOps/Ansible/Playbooks/Переменные и шаблоны|Переменные и шаблоны]]
- [[Base/Computer Science/DevOps/Ansible/Playbooks/Условия и циклы|Условия и циклы]]
- [[Base/Computer Science/DevOps/Ansible/Playbooks/Handlers|Handlers]]
- [[Base/Computer Science/DevOps/Ansible/Playbooks/Управление выполнением|Управление выполнением]]
- [[Base/Computer Science/DevOps/Ansible/Playbooks/Отладка и проверка|Отладка и проверка]]

### Roles

- [[Base/Computer Science/DevOps/Ansible/Roles/Роли|Роли]]
- [[Base/Computer Science/DevOps/Ansible/Roles/Ansible Galaxy и коллекции|Ansible Galaxy и коллекции]]

### Безопасность

- [[Base/Computer Science/DevOps/Ansible/Безопасность/Ansible Vault|Ansible Vault]]

### Практика

- [[Base/Computer Science/DevOps/Ansible/Практика/Развёртывание приложений|Развёртывание приложений]]
- [[Base/Computer Science/DevOps/Ansible/Практика/LEMP на Ubuntu|LEMP на Ubuntu]]
- [[Base/Computer Science/DevOps/Ansible/Практика/Практические задания|Практические задания]]

### Дополнительно

- [[Base/Computer Science/DevOps/Ansible/Дополнительно/Собственные модули|Собственные модули]]
- [[Base/Computer Science/DevOps/Ansible/Шпаргалка|Шпаргалка]]

## Связанные темы

- [[Base/Computer Science/OS/Linux/Linux|Linux]]
- [[Base/Computer Science/DevOps/CI CD/CI CD|CI/CD]]
- [[Base/Computer Science/DevOps/Vault|Vault]]

> [!note] Актуальность
> Тематические заметки актуализированы по официальной документации Ansible и PHP; ссылки размещены рядом с соответствующими темами. Учебный LEMP-пример рассчитан на Ubuntu 24.04 LTS. `Ansible_Course.md` оставлен как архивный конспект курса «Ansible для самых маленьких» (Rotoro.cloud); он не актуализирован и может расходиться с тематическими заметками.
