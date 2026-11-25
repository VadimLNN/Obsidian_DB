---
tags:
  - devops/secrets/vault
status: todo
---

# Vault

Vault централизованно хранит и выдаёт секреты: пароли, токены, ключи и динамические учётные данные.

## Что нужно знать

- seal/unseal и устройство хранилища;
- KV secrets engine;
- auth methods;
- policies;
- tokens и TTL;
- leases и rotation;
- audit log;
- интеграция с приложениями и Kubernetes.

## Практика

- [ ] Запустить dev-сервер Vault.
- [ ] Сохранить и прочитать секрет в KV.
- [ ] Настроить policy и auth method.
- [ ] Получить секрет из Kubernetes workload.

## Связанные темы

- [[Kubernetes]]
- [[Ansible]]
- [[ArgoCD]]
