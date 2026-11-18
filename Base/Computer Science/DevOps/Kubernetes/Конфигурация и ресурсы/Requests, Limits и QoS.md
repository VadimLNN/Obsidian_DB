# Requests, Limits и QoS

[[Kubernetes|Карта тем]] · [[Архитектура кластера]] · [[Health Checks]]

## Requests: планирование и доля ресурсов

Requests описывают потребность контейнера в CPU и памяти. Scheduler использует requests при выборе Node, сопоставляя их с доступной для планирования ёмкостью. Это позволяет не размещать workloads только по текущей низкой загрузке, которая может быстро вырасти.

Фрагмент контейнера:

```yaml
resources:
  requests:
    memory: "100M"
    cpu: "200m"
```

`200m` CPU — 0,2 CPU, а не 200 мегабайт. `100M` памяти — 100 000 000 байт; `100Mi` — 104 857 600 байт. `M` и `Mi` нельзя считать взаимозаменяемыми единицами.

> [!note] Граница гарантии
> Requests — не обещание безусловно выделенной физической памяти и неизменной производительности при любом отказе/перегрузе. CPU request влияет на долю при конкуренции, память учитывается при планировании и управлении давлением ресурсов. Реальные механизмы зависят от kubelet/runtime и cgroups. Pod может быть вытеснен при нехватке ресурсов. [Управление ресурсами контейнеров](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/).

## Limits: ограничение потребления

```yaml
resources:
  limits:
    memory: "150Mi"
    cpu: "300m"
```

- **CPU limit** ограничивает доступное CPU-время. При достижении ограничения возникает throttling, а не OOM; latency приложения может вырасти.
- **Memory limit** ограничивает память cgroup. При давлении памяти ядро может завершить процесс OOM killer. Если завершается контейнер, часто наблюдается `OOMKilled`; kubelet может перезапустить контейнер согласно restartPolicy.

Перезапуск контейнера не равен пересозданию объекта Pod. Eviction на Node — другой механизм, при котором Pod прекращает работу, а workload-контроллер может создать замену.

## Манифест с requests и limits

Пример `deployment-resources.yaml` в заранее созданном namespace `example-app`. Образ заменяется согласно [[Управление ресурсами]]. Отдельное имя и label позволяют не пересекаться с базовым Deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-resources
  namespace: example-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example-resources
  template:
    metadata:
      labels:
        app: example-resources
    spec:
      containers:
        - name: web
          image: registry.example.com/demo/example-app:v0.2
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "100Mi"
              cpu: "200m"
            limits:
              memory: "150Mi"
              cpu: "300m"
```

Requests не должны превышать соответствующие limits. Слишком большой request может оставить Pod Pending даже при низкой наблюдаемой загрузке, если заявленные requests уже заняли allocatable Node.

```bash
kubectl apply -f deployment-resources.yaml
kubectl -n example-app get pods -l app=example-resources
kubectl -n example-app top pods
```

`top` требует работающего Metrics API, обычно Metrics Server. `describe pod <pod-name>` показывает requests, limits, QoS и события; фактическое потребление не равно request.

## QoS-классы

Для **обычной модели ресурсов на уровне контейнеров**, используемой в примере:

| Класс | Условие |
|---|---|
| Guaranteed | У каждого контейнера заданы ненулевые CPU и memory requests/limits; request равен limit для каждого из этих ресурсов |
| Burstable | Не Guaranteed, но хотя бы у одного контейнера задан CPU или memory request/limit |
| BestEffort | Ни у одного контейнера нет CPU/memory requests и limits |

Показанный Deployment имеет класс Burstable. Guaranteed не означает, что приложение постоянно использует ровно заявленное количество; оно может потреблять меньше. QoS влияет на решения при ресурсном давлении, но не является эквивалентом `PriorityClass` и не даёт абсолютной защиты от eviction/OOM.

> [!note] Версионные особенности
> В новых версиях существует механизм ресурсов на уровне Pod; его доступность и влияние на QoS проверяются для версии кластера. Таблица выше описывает традиционную контейнерную конфигурацию, а не все новые варианты. [QoS Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/).

## Выбор значений

Requests/limits задаются по измерениям нормального режима, старта и пиков. Слишком низкий memory limit вызывает OOM, низкий CPU limit — throttling и срабатывание слишком строгих probes. Слишком высокие requests снижают плотность размещения и могут препятствовать rollout с `maxSurge`.

Не существует универсального правила «всегда ставить одинаковые CPU limits всем приложениям». Нужны требования к изоляции и latency. HPA увеличивает число replicas, VPA меняет ресурсные рекомендации/значения, Cluster Autoscaler — число Nodes; их настройка не происходит автоматически от указания requests/limits. См. [[Стратегии обновления]] и [[Диагностика и отладка]].
