---
tags:
  - devops/kubernetes
  - devops/deployment
---

# Deployment и ReplicaSet

[[Kubernetes|Карта тем]] · [[Pod]] · [[Стратегии обновления]]

## Контроллеры и reconciliation

Контроллер наблюдает за ресурсами и устраняет расхождение между желаемым и текущим состоянием. Например, при `replicas: 3` и двух работающих управляемых Pod создаётся недостающий экземпляр. Контроллеры не гарантируют мгновенный успех: нужны доступные ресурсы, образ, сеть и работоспособное приложение.

![](https://ucarecdn.stepik.net/7f5156fa-256d-4289-8ee2-714db4507f58/)

**ReplicaSet** поддерживает заданное количество подходящих Pod. **Deployment** управляет ReplicaSet, обновлением Pod template и историей revisions.

![Kubernetes Deployment](https://ucarecdn.stepik.net/0aa88f1d-9e2f-472e-8ea6-e7fa7b6d979a/)

```text
Deployment → ReplicaSet → Pod → контейнеры
```

Фраза «Deployment управляет Pod» допустима как сокращение, но непосредственное поддержание числа Pod выполняет ReplicaSet controller.

## Базовый Deployment

Полный минимальный `deployment.yaml` находится в [[Управление ресурсами#Декларативное управление]]. Для эксперимента с тремя репликами в нём меняется `spec.replicas` с `1` на `3`; остальные поля сохраняются. Namespace — `example-app`, имя — `example-deployment`, selector и Pod labels — `app: example-app`.

```bash
kubectl apply -f deployment.yaml
kubectl -n example-app get deployment
kubectl -n example-app get replicaset
kubectl -n example-app get pods -l app=example-app
```

Пример формы вывода, суффиксы генерируются кластером:

```text
NAME                                  READY   STATUS    RESTARTS
example-deployment-<hash>-<suffix-1>    1/1     Running   0
example-deployment-<hash>-<suffix-2>    1/1     Running   0
example-deployment-<hash>-<suffix-3>    1/1     Running   0
```

Команда `kubectl -n example-app get pods --watch` показывает изменения. После удаления одного выбранного Pod через `kubectl -n example-app delete pod <pod-name>` ReplicaSet создаёт замену. `replicas` остаётся равным трём.

![](https://ucarecdn.stepik.net/6d43cd31-3c38-47d2-a75a-c9b69bd62462/)

![Кубернетес Дейплоймент](https://ucarecdn.stepik.net/cd167df8-6a39-4375-8288-b9be9e5e5d6a/)

## Самостоятельный ReplicaSet

Пример сохраняется отдельно: он показывает поддержание числа Pod **без Deployment**, а не другую запись того же Deployment. `rs.yaml` содержит два объекта:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: example-rs
---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: example-replicaset
  namespace: example-rs
  labels:
    app: example-rs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: example-rs
  template:
    metadata:
      labels:
        app: example-rs
    spec:
      containers:
        - name: web
          image: registry.example.com/demo/example-app:v0.2
          ports:
            - containerPort: 8080
```

Образ необходимо заменить по соглашению в [[Управление ресурсами]].

```bash
kubectl apply -f rs.yaml
kubectl -n example-rs get replicaset
kubectl -n example-rs get pods
```

Ожидаются два Pod вида `example-replicaset-<suffix>`. Удаление одного приводит к созданию нового. Отдельный namespace и selector исключают пересечение с другими примерами.

## Отличие обновлений

![Deployment vs. ReplicaSet](https://ucarecdn.stepik.net/01a2357f-4167-408f-94bc-99568c6e975f/)

| ReplicaSet | Deployment |
|---|---|
| Поддерживает нужное число Pod | Управляет ReplicaSet и rollout |
| Изменение template не заменяет автоматически уже существующие Pod | Изменение Pod template создаёт/выбирает соответствующий ReplicaSet и запускает rollout |
| Новые Pod создаются по актуальному template | Сохраняет старые ReplicaSet для истории и rollback в пределах retention |
| Не предоставляет полноценную стратегию обновления приложения | Поддерживает RollingUpdate/Recreate и команды rollout |

> [!note] Уточнение
> ReplicaSet можно изменять, включая количество реплик и шаблон. Ограничение состоит в отсутствии автоматической замены существующих Pod при изменении шаблона. Формулировка «ReplicaSet вообще не поддерживает обновления» слишком широкая. [Документация ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/).

Labels и selectors разобраны в [[Namespace, Labels и Selectors]], PodTemplateSpec — в [[Управление ресурсами]].

## Просмотр, масштабирование и удаление

```bash
kubectl -n example-app describe deployment example-deployment
kubectl -n example-app scale deployment example-deployment --replicas=5
kubectl -n example-app get deployment
```

`describe` показывает replicas, template, стратегию и events. `scale` изменяет живой объект. Для постоянной конфигурации изменение согласуется с манифестом/GitOps или владельцем поля replicas; подробнее — [[Управление ресурсами#Infrastructure as Code и drift]].

![scale deployment](https://ucarecdn.stepik.net/cbe0d440-48e4-4c75-ba26-f4b943d0c886/)

Два альтернативных способа удаления:

```bash
kubectl delete -f deployment.yaml
```

```bash
kubectl -n example-app delete deployment example-deployment
```

Обычное каскадное удаление Deployment удаляет принадлежащие ему ReplicaSet и Pod. Параметры каскадного удаления могут изменить поведение. Если файл содержит ещё Namespace или Service, `delete -f` относится и к ним — состав файла нужно учитывать.

Deployment типичен для stateless-сервисов, но не заменяет [[DaemonSet]], [[Job и CronJob]] и [[StatefulSet]]. Малый размер образа ускоряет pull/start, но время создания Pod также зависит от планирования, probes, storage и сети.
