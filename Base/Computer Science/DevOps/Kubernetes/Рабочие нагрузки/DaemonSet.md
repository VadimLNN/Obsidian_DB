# DaemonSet

[[Kubernetes|Карта тем]] · [[Архитектура кластера]] · [[Deployment и ReplicaSet]]

## Назначение

DaemonSet обеспечивает запуск экземпляра Pod на каждой **подходящей Node** или на выбранной группе узлов. В отличие от Deployment, количество экземпляров определяется набором узлов, на которых должна работать служба, а не фиксированным `replicas`.

![](https://ucarecdn.stepik.net/a7b4c78e-a2f9-4f89-ad8e-aafff2e7aa03/)

Типичные задачи — обслуживание самой платформы: сбор метрик, сбор логов, сетевые агенты. При добавлении подходящей Node контроллер создаёт на ней Pod; при удалении Node соответствующий экземпляр исчезает.

## Примеры применения

| Задача | Пример |
|---|---|
| Метрики узла | [node-exporter DaemonSet](https://github.com/prometheus-operator/kube-prometheus/blob/main/manifests/nodeExporter-daemonset.yaml) |
| Сбор логов с узлов | [Vector](https://github.com/vectordotdev/vector), [Fluentd](https://github.com/fluent/fluentd) |
| Диагностика связности | [k8s-pinger](https://github.com/MadEngineX/k8s-pinger) |
| Размещение входного proxy на узлах | Некоторые варианты развёртывания Ingress controller |

Метрики node-exporter обычно опрашиваются системой мониторинга и затем используются в Prometheus/VictoriaMetrics и Grafana. Наличие DaemonSet не настраивает dashboards и alerts автоматически.

## Ограничения

«На каждой Node» не означает безусловный запуск на всех узлах кластера: учитываются node selectors, affinity, taints/tolerations и возможность планирования. Например, control-plane узлы часто имеют ограничения на обычные workloads.

Размещение Ingress controller как DaemonSet — архитектурный выбор: меняются привязка к узлам, использование host ports и способ внешней балансировки. Сам DaemonSet не создаёт внешний LoadBalancer и не заменяет [[Service]] или [[Ingress]].

Готового универсального манифеста агента здесь нет: полномочия, host mounts и доступ к сети определяются конкретной задачей. Использование чужого инфраструктурного DaemonSet требует проверки его привилегий.
