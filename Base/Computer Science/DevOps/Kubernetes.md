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

- Control Plane: API Server, etcd, Scheduler, Controller Manager, Cloud Controller Manager;
- Worker Node: kubelet, container runtime, kube-proxy;
- декларативная модель и reconciliation loop.

### Workloads и объекты

- Pod, Deployment, DaemonSet, StatefulSet;
- Job, CronJob;
- ConfigMap, Secret;
- requests/limits, probes, rollout и rollback.

### Networking

- Service и его типы;
- Ingress;
- NetworkPolicy;
- CNI, kube-proxy, CoreDNS.

### Storage

- PV, PVC, StorageClass;
- Access Modes;
- особенности stateful-нагрузок.

### Helm

- Chart, `values.yaml`, templates;
- установка, обновление и откат release.

### Развёртывание кластера

- локально: Minikube, Kind или k3d;
- самостоятельно: kubeadm или kubespray;
- managed Kubernetes.

### Advanced

- RBAC и ServiceAccount;
- HPA;
- Operators;
- Service Mesh;
- backup/restore etcd;
- GitOps;
- multi-cluster;
- cluster troubleshooting.

## Практика

- [ ] Поднять локальный кластер.
- [ ] Развернуть Deployment, Service и ConfigMap.
- [ ] Проверить RollingUpdate, rollback и scaling.
- [ ] Настроить Liveness и Readiness probes.
- [ ] Развернуть Frontend + Backend + PostgreSQL.
- [ ] Упаковать приложение в Helm Chart.
- [ ] Разделить dev/prod по namespaces.

## Собеседование

- [ ] Чем отличаются Deployment, StatefulSet и DaemonSet?
- [ ] Как устроен Control Plane?
- [ ] Какие типы Service существуют?
- [ ] Почему Pod, а не контейнер, является минимальной единицей?
- [ ] Как диагностировать Pod в состоянии Pending или CrashLoopBackOff?

## Связанные темы

- [[Docker]]
- [[Computer Networks]]
- [[Prometheus]]
- [[ArgoCD]]

