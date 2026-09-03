---
tags:
  - devops
  - gitops
  - argocd
status: todo
---

# ArgoCD

Argo CD реализует GitOps-подход: желаемое состояние Kubernetes хранится в Git, а контроллер синхронизирует с ним кластер.

## Что нужно знать

- Git как source of truth;
- Application и Project;
- sync, health и drift;
- automatic/manual sync;
- Helm через Argo CD;
- App of Apps;
- rollback и история развёртываний;
- управление доступом и секретами.

## Практика

- [ ] Установить Argo CD.
- [ ] Подключить Git-репозиторий.
- [ ] Развернуть приложение из manifests или Helm Chart.
- [ ] Проверить drift и автоматическую синхронизацию.
- [ ] Собрать App of Apps.

## Связанные темы

- [[Git]]
- [[Kubernetes]]
- [[CI CD]]

