---
tags:
  - devops/kubernetes
  - computer-science/networking
---

# Service

[[Kubernetes|Карта тем]] · [[DNS и Service Discovery]] · [[Ingress]]

## Стабильный доступ к изменяемому набору Pod

[[Pod]] эфемерны: при замене объекта его IP может измениться. Service отделяет адрес клиента от конкретных экземпляров приложения. Обычный Service предоставляет стабильный виртуальный ClusterIP на время жизни самого Service и DNS-имя.

![](https://ucarecdn.stepik.net/d01ddf16-25ef-4503-8e6e-f770d246c716/)

Selector определяет Pod в namespace Service. Сведения о backend-адресах поддерживаются в EndpointSlice, а сетевые компоненты реализуют направление соединений к этим адресам. Пересоздание самого Service может изменить ClusterIP. Headless Service и ExternalName отличаются от модели «один виртуальный IP».

## Общий backend для примеров

Примеры ClusterIP, NodePort, LoadBalancer и Ingress используют **один** backend: Namespace `example-app` и Deployment `example-deployment` из [[Управление ресурсами#Декларативное управление]]. Образ-заглушка заменяется доступным образом с HTTP на 8080. Pod template содержит `app: example-app`.

Это устраняет повторение одинаковых Deployment-манифестов. Service разных типов остаются разными объектами. Общие файлы — `ns.yaml` и `deployment.yaml`; файлы Service приведены ниже.

```bash
kubectl apply -f ns.yaml
kubectl apply -f deployment.yaml
kubectl -n example-app get deployment
kubectl -n example-app get pods -o wide
```

## ClusterIP

Тип по умолчанию обеспечивает внутрикластерный доступ. Он сам по себе не создаёт внешний endpoint. Это не граница безопасности: сетевые политики, маршруты и proxy могут предоставлять дополнительный доступ.

![](https://ucarecdn.stepik.net/5bff2eb0-d7c2-4afd-b8e6-e3a2c3e1020d/)

![](https://ucarecdn.stepik.net/5bff2eb0-d7c2-4afd-b8e6-e3a2c3e1020d/)

![[Pasted image 20261118150001.png]]

`service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-service
  namespace: example-app
spec:
  type: ClusterIP
  selector:
    app: example-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

`port` — порт Service, `targetPort` — порт backend-контейнера. Selector сопоставляется с Pod labels, а не с именем Deployment или его собственными metadata labels.

```bash
kubectl apply -f service.yaml
kubectl -n example-app get services
kubectl -n example-app describe service example-service
watch kubectl -n example-app get svc
```

В другом терминале `watch kubectl -n example-app get pods -o wide` показывает IP Pod. При удалении одного управляемого Pod ReplicaSet создаёт замену; адрес Service остаётся прежним.

![](https://ucarecdn.stepik.net/3e3ddc5b-cd1e-4a23-b743-2b2f03b394bf/)

### Port-forward и проверка endpoints

```bash
kubectl -n example-app port-forward svc/example-service 8181:80
```

В другом терминале:

```bash
curl http://localhost:8181/services
```

При контракте демонстрационного приложения ожидается `Hi there, I love services!`. Port-forward работает через API и выбирает Pod за Service: он не проверяет весь сетевой путь через ClusterIP, балансировку или доступ от других Pod. Для этого используются запросы из кластера — [[DNS и Service Discovery]].

Наличие backend-адресов проверяется через describe и EndpointSlice:

```bash
kubectl -n example-app get endpointslices -l kubernetes.io/service-name=example-service
```

Отсутствие подходящих ready backend — повод проверить labels, namespace и [[Health Checks|readiness]], а не только сам Service.

## Балансировка

Service распределяет соединения по доступным backend, учитывая настройки dataplane, готовность, affinity и traffic policy. Это обычно уровень сетевых соединений, а не обязательное равномерное чередование HTTP-запросов.

![](https://ucarecdn.stepik.net/a702b30d-2ff3-4e50-a736-620e3a0fcd6f/)

![](https://ucarecdn.stepik.net/a702b30d-2ff3-4e50-a736-620e3a0fcd6f/)

![[Pasted image 20261118145849.png]]

> [!note] Алгоритм зависит от реализации
> Утверждение «Service почти всегда использует round robin» неточно. Например, выбор в iptables-режиме не равен строгому round robin; IPVS и другие реализации имеют свою семантику. Keep-alive может удерживать много запросов на одном соединении и Pod. kube-proxy и CNI/dataplane нужно рассматривать отдельно от API-объекта Service.

## NodePort

NodePort публикует порт на подходящих адресах Nodes и направляет соединения к backend. Диапазон по умолчанию — `30000–32767`, но он настраивается в кластере. Доступ извне зависит от маршрутизации, [[Firewall|firewall]], nodePortAddresses и traffic policy; произвольный внешний клиент не обязательно имеет доступ к каждой Node.

![](https://ucarecdn.stepik.net/f06e8e3b-60e4-4bde-8a75-22efe12f1b83/)

![[Pasted image 20261118150107.png]]

![](https://ucarecdn.stepik.net/40f4f66c-08a4-40d1-97aa-3f0d165f7f99/)

![[Pasted image 20261118150128.png]]

`nodeport.yaml`, использующий общий backend:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-nodeport
  namespace: example-app
spec:
  type: NodePort
  selector:
    app: example-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30050
```

```bash
kubectl apply -f nodeport.yaml
kubectl -n example-app get svc example-nodeport
kubectl get nodes -o wide
```

Ожидаемый порт в выводе — `80:30050/TCP`. При достижимом IP Node запрос имеет вид `http://<node-ip>:30050/nodePort`. `nodePort` должен быть свободен и входить в разрешённый диапазон.

### Minikube с Docker driver

IP контейнерной Node может быть недоступен непосредственно с хоста, особенно на macOS/Windows. Возможные способы проверки:

```bash
minikube service --url -n example-app example-nodeport
```

Команда выдаёт URL, например `http://127.0.0.1:<port>`; если работает туннель, терминал должен оставаться открытым. Запрос выполняется к фактически выданному адресу с путём `/nodePort`.

Другой вариант при существующей Docker-сети `minikube` и доступном из неё Node IP:

```bash
NODE_IP=$(minikube ip)
docker run --rm --network minikube curlimages/curl:latest \
  "http://$NODE_IP:30050/network-check"
```

`latest` здесь — диагностический изменяемый тег, не фиксация окружения. Имя сети и достижимость адреса зависят от driver/profile. [Доступ к приложениям Minikube](https://minikube.sigs.k8s.io/docs/handbook/accessing/).

## LoadBalancer

LoadBalancer запрашивает внешнюю балансировку у поддерживающей реализации. В managed-кластере её может предоставить облачный провайдер; в self-hosted используется отдельная интеграция, например MetalLB. Сам API-объект не создаёт физический балансировщик без такой реализации.

![](https://ucarecdn.stepik.net/3ba33df0-6894-4baf-a25e-0474d02a6a98/)

`loadbalancer.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-loadbalancer
  namespace: example-app
spec:
  type: LoadBalancer
  selector:
    app: example-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

```bash
kubectl apply -f loadbalancer.yaml
kubectl -n example-app get svc example-loadbalancer
```

До выделения адреса `EXTERNAL-IP` может оставаться `<pending>`. Адрес может быть внутренним или публичным, а некоторые реализации возвращают hostname. Для Minikube требуется поддерживаемый способ туннелирования:

```bash
minikube tunnel
```

После выделения адреса запрос направляется на `http://<assigned-address>/loadBalancer`. В Yandex Cloud выделение балансировщика зависит от конфигурации интеграции, сети, квот и прав; внешний IP не гарантирует доступ через security groups.

> [!warning] Устаревшее поле
> `spec.loadBalancerIP`, упоминавшееся как универсальный способ выбрать адрес, deprecated начиная с Kubernetes 1.24 и не переносимо между провайдерами. Для выделенного адреса используются поддерживаемые конкретной реализацией настройки. [Service LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer).

## Выбор типа

| Сценарий | Вариант |
|---|---|
| Связь приложений внутри кластера | ClusterIP |
| Внешний TCP/UDP endpoint при наличии интеграции | LoadBalancer |
| Общий вход HTTP(S) с host/path-маршрутизацией | [[Ingress]] и backend Service |
| Прямой доступ через IP Node и порт | NodePort с учётом маршрутизации и безопасности |
| DNS-псевдоним на внешнее имя | ExternalName: CNAME, без обычного proxy/ClusterIP |
| Обнаружение адресов отдельных экземпляров | Headless Service: `clusterIP: None`; это настройка Service, а не ещё один `type` |

NodePort — допустимый механизм, но открывает дополнительные порты узлов. Привязка DNS к удалённой впоследствии Node приводит к отказам; нестандартный порт неудобен для пользователей. Часто эти проблемы решают внешним балансировщиком или ingress, а не ручным перечислением Nodes в DNS.
