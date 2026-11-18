---
tags:
  - devops
  - kubernetes
status: todo
---

# Kubernetes

Kubernetes оркестрирует контейнерные приложения и управляет их развёртыванием, сетью, конфигурацией, хранением и восстановлением после сбоев.

## Что нужно знать

### Архитектура

- [[Архитектура кластера|Control Plane]]: API Server, etcd, Scheduler, Controller Manager, Cloud Controller Manager;
- [[Архитектура кластера|Worker Node]]: kubelet, container runtime, [[Архитектура кластера|kube-proxy]];
- [[Управление ресурсами|декларативная модель и reconciliation loop]].

### Workloads и объекты

- [[Pod]], [[Deployment и ReplicaSet|Deployment]], [[DaemonSet]], [[StatefulSet]];
- [[Job и CronJob|Job]], [[Job и CronJob|CronJob]];
- [[ConfigMap]], Secret;
- [[Requests, Limits и QoS|requests/limits]], [[Health Checks|probes]], [[Стратегии обновления|rollout]] и [[Стратегии обновления|rollback]].

### Networking

- [[Service]] и его типы;
- [[Ingress]];
- NetworkPolicy;
- CNI, [[Архитектура кластера|kube-proxy]], [[DNS и Service Discovery|CoreDNS]].

### Storage

- PV, PVC, StorageClass;
- Access Modes;
- особенности stateful-нагрузок.

### Helm

- Chart, `values.yaml`, templates;
- установка, обновление и откат release.

### Развёртывание кластера

- локально: [[Локальный кластер|Minikube]], Kind или k3d;
- самостоятельно: kubeadm или kubespray;
- [[Self-hosted и Managed Kubernetes|managed Kubernetes]].

### Advanced

- RBAC и ServiceAccount;
- HPA;
- Operators;
- Service Mesh;
- backup/restore etcd;
- GitOps;
- multi-cluster;
- [[Диагностика и отладка|cluster troubleshooting]].

## Практика

- [ ] Поднять [[Локальный кластер|локальный кластер]].
- [ ] Развернуть [[Deployment и ReplicaSet|Deployment]], [[Service]] и [[ConfigMap]].
- [ ] Проверить [[Стратегии обновления|RollingUpdate]], [[Стратегии обновления|rollback]] и [[Deployment и ReplicaSet|scaling]].
- [ ] Настроить [[Health Checks|Liveness]] и [[Health Checks|Readiness]] [[Health Checks|probes]].
- [ ] Развернуть Frontend + Backend + PostgreSQL.
- [ ] Упаковать приложение в Helm Chart.
- [ ] Разделить dev/prod по [[Namespace, Labels и Selectors|namespaces]].

## Собеседование

- [ ] Чем отличаются [[Deployment и ReplicaSet|Deployment]], [[StatefulSet]] и [[DaemonSet]]?
- [ ] Как устроен [[Архитектура кластера|Control Plane]]?
- [ ] Какие типы [[Service]] существуют?
- [ ] Почему [[Pod]], а не контейнер, является минимальной единицей?
- [ ] Как диагностировать [[Pod]] в состоянии [[Диагностика и отладка|Pending]] или [[Диагностика и отладка|CrashLoopBackOff]]?

## Связанные темы

- [[Docker]]
- [[Computer Networks]]
- [[Prometheus]]
- [[ArgoCD]]

## Навигация по заметкам

- **Основы:** [[Обзор Kubernetes]], [[Архитектура кластера]], [[Управление ресурсами]], [[Namespace, Labels и Selectors]].
- **Рабочие нагрузки:** [[Pod]], [[Deployment и ReplicaSet]], [[Стратегии обновления]], [[DaemonSet]], [[Job и CronJob]], [[StatefulSet]].
- **Сеть:** [[Service]], [[DNS и Service Discovery]], [[Ingress]].
- **Конфигурация и ресурсы:** [[ConfigMap]], [[Requests, Limits и QoS]], [[Health Checks]].
- **Кластеры:** [[Локальный кластер]], [[Self-hosted и Managed Kubernetes]], [[Облачный кластер]].
- **Инструменты и диагностика:** [[kubectl]], [[Диагностика и отладка]], [[Дополнительные инструменты]].

Темы без отдельных материалов сохранены в исходном плане обычным текстом. Примеры с `registry.example.com` требуют замены образа согласно [[Управление ресурсами#Императивное управление|соглашению о примерах]]. Иллюстрации сохранены по исходным адресам; имена на них могут отличаться от унифицированных имён в тексте.
