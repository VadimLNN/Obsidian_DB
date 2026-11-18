# kubectl

[[Kubernetes|Карта тем]] · [[Управление ресурсами]] · [[Диагностика и отладка]]

## Модель команды

`kubectl` — CLI-клиент Kubernetes API. Типичная структура:

```text
kubectl [context/namespace flags] <действие> <тип ресурса> [имя] [параметры]
```

Команда выполняется с правами identity из kubeconfig. Наличие доступа к API не означает право изменять любой ресурс.

## Контексты и namespace

Context связывает кластер, пользователя и namespace по умолчанию.

```bash
kubectl config get-contexts
kubectl config current-context
kubectl config use-context example-cluster
kubectl -n example-app get pods
```

`example-cluster` должен быть существующим context. Переключение влияет на последующие команды, если они не задают другой context явно. Namespace можно закрепить для текущего context:

```bash
kubectl config set-context --current --namespace=example-app
```

Перед apply/delete проверяются и cluster, и namespace. Cluster-scoped ресурсы не ограничиваются `-n`.

## Создание и изменение

```bash
kubectl apply -f manifest.yaml
kubectl create deployment example-nginx --image=nginx
kubectl -n example-app edit deployment example-deployment
kubectl -n example-app edit pod example-pod
```

`apply` создаёт или обновляет поддерживаемые поля по манифесту. `create deployment` — императивный пример; `nginx` без тега не фиксирует версию образа. `edit` изменяет живой объект; многие поля Pod неизменяемы, и правка может быть отвергнута API. Для управляемого приложения обычно изменяется Pod template в Deployment, а не отдельный Pod.

Принципы IaC и последствия drift — [[Управление ресурсами]].

## Просмотр состояния

```bash
kubectl get namespaces
kubectl -n example-app get deployments
kubectl -n example-app get replicasets
kubectl -n example-app get pods -o wide
kubectl -n example-app get serviceaccounts
kubectl get clusterrolebindings
kubectl -n example-app describe pod example-pod
kubectl -n example-app top pods
```

`get` даёт обзор, `-o wide` — дополнительные поля, включая Node и IP для Pod. `describe` показывает детали, события и причины сбоев. `top` требует Metrics API и показывает измеренное потребление, а не requests.

Сокращения: `ns`, `po`, `deploy`, `rs`, `svc`, `cm`, `sts`. Доступные в конкретном кластере ресурсы показывает `kubectl api-resources`. Объекты из CRD существуют только после установки соответствующих API.

## Логи и shell

```bash
kubectl -n example-app logs -f example-pod
kubectl -n example-app logs example-pod -c web
kubectl -n example-app logs example-pod -c web --previous
kubectl -n example-app exec -it example-pod -c web -- sh
```

`--previous` запрашивает логи предыдущего экземпляра контейнера после restart, а не предыдущего удалённого Pod. `--` отделяет параметры kubectl от исполняемой команды. Shell должен существовать в образе: не каждый контейнер содержит `sh` или `bash`. Для distroless/minimal images нужен другой способ диагностики, см. [[Диагностика и отладка]].

## Копирование файлов

```bash
kubectl cp example-app/example-pod:/tmp/report.txt ./report.txt
kubectl cp ./config.yaml example-app/example-pod:/tmp/config.yaml
```

Первый пример копирует **из** Pod, второй — **в** Pod. Для многоконтейнерного Pod указывается `-c <container>`. `kubectl cp` обычно требует `tar` внутри контейнера; отсутствие утилиты — частая причина ошибки. Ручная правка файлов контейнера не заменяет изменение образа или [[ConfigMap]].

## Port-forward

```bash
kubectl -n example-app port-forward pod/example-pod 8080:8080
kubectl -n example-app port-forward svc/example-service 8181:80
```

Слева локальный порт, справа порт объекта назначения. Для MongoDB аналогичный пример — локальный `28015` к Pod/Service порту `27017`, при наличии соответствующего приложения. Процесс работает до остановки команды; замена выбранного Pod может оборвать соединение.

Port-forward предназначен для диагностики и локального доступа, а не постоянной внешней публикации. Его успешность не подтверждает сетевой путь через ClusterIP.

## Описание API и удаление

```bash
kubectl explain deployment.spec
kubectl explain deployment.spec.template
kubectl explain replicaset.spec.template
kubectl explain pod.spec
```

`explain` читает схему API, доступную клиенту/кластеру. Это помогает отличить поля Deployment от Pod template.

```bash
kubectl -n example-app delete pod example-pod
kubectl delete -f deployment.yaml
```

Удаление управляемого Pod обычно вызывает его замену контроллером. Удаление самостоятельного Pod не восстанавливает его. `delete -f` относится ко всем объектам файла. [[Namespace, Labels и Selectors]] отдельно описывает последствия удаления Namespace.

Для безопасного предварительного просмотра поддерживаются client/server dry-run, но они проверяют разные уровни: `--dry-run=client` не подтверждает admission, quota и права API, а `--dry-run=server` требует подключения к кластеру. Наличие корректного YAML не доказывает успешное развёртывание.
