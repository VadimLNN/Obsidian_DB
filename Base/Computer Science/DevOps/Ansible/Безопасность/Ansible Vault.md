---
tags:
  - devops/ansible
  - devops/secrets
---

# Ansible Vault

[[Base/Computer Science/DevOps/Ansible/Ansible|Карта тем]] · [[Base/Computer Science/DevOps/Ansible/Inventory/Подключение к узлам|Подключение к узлам]] · [[Base/Computer Science/DevOps/Ansible/Playbooks/Переменные и шаблоны|Переменные и шаблоны]] · [[Base/Computer Science/DevOps/Ansible/Практика/LEMP на Ubuntu|LEMP на Ubuntu]]

## SSH-аутентификация

Для аутентификации по ключу:

```bash
ssh-keygen -t ed25519 -C "ansible-control"
ssh-copy-id deployer@192.0.2.10
ssh deployer@192.0.2.10
```

Ключи предпочтительнее паролей в inventory. Публичный ключ добавляется на управляемый сервер; приватный остаётся у управляющей машины.

## Шифрование секретов и безопасность

**Ansible Vault** шифрует данные в репозитории: пароли, API-ключи, приватные параметры. **Он защищает данные в хранилище, но не отменяет риск их утечки во время выполнения**.

```bash
ansible-vault create secrets.yml
ansible-vault edit secrets.yml
ansible-vault encrypt secrets.yml
ansible-vault view secrets.yml
ansible-playbook site.yml --ask-vault-pass
```

Если secrets уже использованы в task, применять `no_log: true`, тщательно настраивать доступ к выводам [[CI CD|CI/CD]] и не отправлять секреты в `debug`, callback-логи и отчёты.

Рекомендуемые меры:

- SSH-ключи и проверка host keys вместо паролей в открытом inventory.
- Минимальные права пользователя/сервисного аккаунта.
- `become` только там, где требуется.
- Отдельный inventory для test/staging/production.
- Защищённый Vault/secret manager; не коммитить пароли, приватные ключи и расшифрованные secrets.
- Проверка сторонних ролей и их версий.
- План восстановления перед массовыми изменениями.
- Не выключать глобально `host_key_checking` ради удобства: это снижает защиту SSH-соединений.

Документация: [Ansible Vault](https://docs.ansible.com/projects/ansible/latest/vault_guide/vault.html).

## Связанные заметки

- [[Vault]]
