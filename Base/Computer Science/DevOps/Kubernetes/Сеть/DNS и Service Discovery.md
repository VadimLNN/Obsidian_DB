# DNS и Service Discovery

[[Kubernetes|Карта тем]] · [[Service]] · [[Namespace, Labels и Selectors]]

## Обнаружение сервисов

Service discovery позволяет клиентам находить приложение по устойчивому имени, не сохраняя IP отдельных Pod. В типичной установке внутреннюю DNS-службу Kubernetes реализует CoreDNS.

Обычный Service получает имя вида:

```text
<service>.<namespace>.svc.<cluster-domain>
```

Например: `example-service.example-app.svc.cluster.local`. `cluster.local` — распространённое, но настраиваемое доменное окончание. DNS-имя связано с именем Service и namespace, а не с IP конкретного Pod.

## Короткие имена

- В одном namespace обычно достаточно `example-service`.
- Из другого namespace — `example-service.example-app`.
- Возможны `example-service.example-app.svc` и полное имя с cluster domain.

![k8s service discovery](https://ucarecdn.stepik.net/5cff487f-a69f-44fb-a6ce-a58b71544e17/)

![k8s service discovery](https://ucarecdn.stepik.net/54d05987-646e-411b-b71f-a46f08987627/)

Разрешение коротких имён зависит от DNS policy и search domains Pod. Для обычного ClusterIP Service результат указывает на виртуальный адрес Service; headless Service может возвращать адреса backend. Стабильное имя StatefulSet Pod не означает неизменный IP. [DNS для Services и Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).

## Диагностический Pod

Предпосылка — Deployment и `example-service` в namespace `example-app`, описанные в [[Service]]. Файл `curl-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-pod
  namespace: default
  labels:
    app: curl
spec:
  containers:
    - name: curl
      image: curlimages/curl:latest
      command: ["sleep", "3600"]
```

`sleep` оставляет контейнер работающим на час, чтобы выполнять в нём диагностические команды. Наличие curl не означает наличие `dig` или `nslookup`. Для повторяемого стенда вместо изменяемого `latest` нужен проверенный тег/digest.

```bash
kubectl apply -f curl-pod.yaml
kubectl -n default exec -it curl-pod -- sh
```

Команды внутри контейнера:

```bash
curl http://example-service.example-app/practice
curl http://example-service.example-app.svc/practice
curl http://example-service.example-app.svc.cluster.local/practice
```

Полное имя с `cluster.local` подходит только кластеру с таким DNS domain. При контракте приложения из [[Управление ресурсами]] все запросы должны вернуть `Hi there, I love practice!`.

Для проверки короткого имени создаётся второй экземпляр этого манифеста с `metadata.namespace: example-app`, например в `curl-pod-local.yaml`:

```bash
kubectl apply -f curl-pod-local.yaml
kubectl -n example-app exec curl-pod -- curl http://example-service/practice
```

## Диагностика по уровням

1. Ошибка разрешения имени: проверить имя Service, namespace, DNS policy, CoreDNS и сетевой доступ к DNS.
2. Имя разрешилось, соединение не устанавливается: проверить endpoints, port/targetPort, listener приложения и сетевые политики.
3. Ответ получен, но код HTTP неожиданный: проверить маршрут приложения, заголовок Host и сам backend.

Успешный `port-forward` не заменяет DNS-проверку из Pod. Стабильный DNS также не гарантирует наличие ready backend. См. [[Диагностика и отладка]].
