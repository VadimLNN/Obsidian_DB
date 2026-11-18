# Namespace, Labels и Selectors

[[Kubernetes|Карта тем]] · [[Управление ресурсами]] · [[Service]]

## Namespace

Namespace логически группирует namespaced-ресурсы: Pod, Deployment, Service, ConfigMap. Типичные границы — приложение, команда или окружение `dev`, `staging`, `production`. Имена ресурсов одного вида могут повторяться в разных namespaces.

![](https://ucarecdn.stepik.net/58e05dff-25b2-413a-a7f0-6b493e550180/)

![](https://ucarecdn.stepik.net/b30f603b-60d3-4429-ab27-0c904d736d59/)

Namespace не является каталогом файловой системы или Linux network namespace. Pod одного Kubernetes Namespace могут находиться на разных Nodes. Node, PersistentVolume и ClusterRoleBinding относятся к cluster-scoped ресурсам и не помещаются в namespace.

## Создание и выбор пространства имён

Общий namespace примеров, файл `ns.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: example-app
```

```bash
kubectl apply -f ns.yaml
kubectl get namespaces
kubectl -n example-app get pods
```

![](https://ucarecdn.stepik.net/2bf0a5b2-d683-43fd-90bf-c1031e058230/)

Краткое имя ресурса — `ns`. Без `-n` используется namespace текущего context; если он не указан, обычно `default`, а не безусловно `default` при каждой команде.

Императивный вариант для отдельного временного окружения:

```bash
kubectl create ns example-dev
kubectl get ns
kubectl delete ns example-dev
```

Удаление namespace удаляет и находящиеся в нём ресурсы. `kubectl delete -f ns.yaml` удалит `example-app` целиком; это не способ удалить только одну демонстрационную job. Перед операцией проверяются context и выбранный namespace.

## Организация и изоляция

- Раздельные namespaces снижают вероятность случайной операции над ресурсом другой среды.
- RBAC может ограничивать права команды конкретным namespace.
- NetworkPolicy может ограничивать сетевой обмен, если сетевой плагин обеспечивает её исполнение.

> [!important]
> Namespace сам по себе не запрещает другой команде доступ и не блокирует межсетевое взаимодействие. Изоляция требует правил RBAC и сети. Разделение `dev`/`prod` в одном кластере также не устраняет общий отказ узлов или control plane.

## Labels и selectors

**Label** — пара ключ–значение в `metadata.labels`, описывающая объект. **Selector** — условие отбора по меткам. Это один из основных способов связывать объекты Kubernetes.

![Kubernetes labels selector](https://ucarecdn.stepik.net/f46bdfd7-254a-49de-9570-f6097ed9847b/)

В ReplicaSet/Deployment selector должен соответствовать меткам Pod template. Фрагмент `spec` контроллера:

```yaml
selector:
  matchLabels:
    app: example-app
template:
  metadata:
    labels:
      app: example-app
```

Метки `metadata.labels` самого Deployment не заменяют `spec.template.metadata.labels`: это метки разных объектов. Service выбирает Pod в своём namespace напрямую, а не Deployment:

```yaml
selector:
  app: example-app
```

![Kubernetes labels selector](https://ucarecdn.stepik.net/c67889e2-cb2c-4b34-b1fc-048e636eb414/)

Владение Pod со стороны контроллера также отражается в `ownerReferences`; selector — не вся модель владения. Пересекающиеся selectors независимых контроллеров могут создавать конфликт управления. В `apps/v1` selector Deployment не меняется после создания объекта. [ReplicaSet и владение Pod](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/).

Полезные команды:

```bash
kubectl -n example-app get pods --show-labels
kubectl -n example-app get pods -l app=example-app
kubectl -n example-app describe service example-service
```

Пустой набор endpoints у Service может означать несовпадение labels, отсутствие Pod либо их неготовность; подробнее — [[Диагностика и отладка]].
