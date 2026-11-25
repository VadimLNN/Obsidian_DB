---
tags:
  - devops/kubernetes
  - devops/containers
---

# Pod

[[Kubernetes|Карта тем]] · [[Deployment и ReplicaSet]] · [[Health Checks]]

## Определение и изоляция

Pod — минимальная единица размещения в Kubernetes: один или несколько тесно связанных контейнеров, запускаемых на одной Node. Число контейнеров определяется архитектурой приложения, а не обязательным шаблоном «один Pod — один контейнер».

![Pod в Кубернетес](https://ucarecdn.stepik.net/b5da42dd-fc81-4ed0-8a74-7911ebb1484e/)

Контейнеры Pod используют общий сетевой namespace: IP и пространство портов. Они могут обращаться друг к другу через localhost. Для обмена файлами нужен общий volume, подключённый к нужным контейнерам; корневые файловые системы не становятся общими автоматически. Пространство процессов по умолчанию отдельно; совместное использование PID namespace настраивается дополнительно.

![кубернетес поды](https://ucarecdn.stepik.net/9503aef4-5daa-4c32-af16-e445cf9623c3/)

Все контейнеры Pod находятся на одной Node, но Pod одного Kubernetes Namespace могут быть распределены по разным Nodes. Kubernetes Namespace и [[Cgroups и Namespaces|Linux namespace]] — разные понятия.

## Жизненный цикл

Контейнер внутри существующего Pod может перезапуститься согласно `restartPolicy`, сохранив сам объект Pod. Удаление Pod, наоборот, удаляет объект: он не возвращается самостоятельно. Контроллер может создать **замену** с новым UID и, возможно, другим IP/узлом. Pod не переносится между Nodes как работающая VM.

У самостоятельного Pod нет ReplicaSet, который восстановит желаемое количество после удаления. Для долгоживущего stateless-приложения обычно используется [[Deployment и ReplicaSet|Deployment]], для разовой работы — [[Job и CronJob|Job]]. Stateful-нагрузки и узловые агенты имеют другие контроллеры.

## Манифест отдельного Pod

Namespace `example-app` создаётся по [[Namespace, Labels и Selectors]]. Образ-заглушка и его контракт описаны в [[Управление ресурсами#Императивное управление|соглашении о примерах]]. Файл `pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  namespace: example-app
  labels:
    app: example-pod
spec:
  containers:
    - name: web
      image: registry.example.com/demo/example-app:v0.2
      imagePullPolicy: Always
      ports:
        - name: web
          containerPort: 8080
          protocol: TCP
```

| Поле | Значение |
|---|---|
| `kind: Pod` | Создание Pod без workload-контроллера |
| `metadata.name` | Имя объекта внутри namespace |
| `metadata.labels` | Метки для поиска и selectors |
| `containers[].name` | Имя контейнера внутри Pod |
| `containers[].image` | Ссылка на доступный container image |
| `imagePullPolicy: Always` | Проверка образа в registry при запуске; кэш слоёв всё ещё может использоваться |
| `containerPort` | Декларация порта контейнера; не команда открытия firewall и не публикация наружу |

## Применение и локальный доступ

```bash
kubectl apply -f pod.yaml
kubectl -n example-app get pods
kubectl -n example-app describe pod example-pod
kubectl -n example-app port-forward pod/example-pod 8080:8080
```

Проброс действует, пока работает `port-forward`. В другом терминале:

```bash
curl http://localhost:8080/Kubernetes
```

При указанном контракте приложения ожидается:

```text
Hi there, I love Kubernetes!
```

Статус Running означает, что Pod назначен на Node и контейнеры находятся в процессе выполнения/запуска, но не доказывает готовность приложения обслуживать запросы. Для этого нужны [[Health Checks|readiness checks]] и проверка самого запроса.

## Удаление и восстановление

```bash
kubectl -n example-app delete pod example-pod
kubectl -n example-app get pods
```

![](https://ucarecdn.stepik.net/60cbb040-e68a-4086-9d72-9ad07d1dda2a/)

Этот самостоятельный Pod не будет автоматически создан снова. Повторное применение `pod.yaml` создаст новый объект. При наличии ReplicaSet удалённый Pod заменяется контроллером; подробный эксперимент — в [[Deployment и ReplicaSet]].

Ошибки образа, нехватка ресурсов и циклические рестарты разбираются в [[Диагностика и отладка]].
