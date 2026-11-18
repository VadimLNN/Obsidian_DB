# ConfigMap

[[Kubernetes|Карта тем]] · [[Pod]] · [[Deployment и ReplicaSet]]

## Конфигурация отдельно от образа

ConfigMap — API-объект для **несекретной** конфигурации в виде ключей и значений. Он позволяет подавать одному образу разные настройки окружений без изменения исходного кода или сборки отдельного образа для dev/prod.

Пример содержимого прикладного `config.yaml`:

```yaml
app:
  name: example-app
  version: 1.0.0
server:
  port: 8080
  host: 0.0.0.0
database:
  host: db.example-app.svc.cluster.local
  port: 5432
  user: db_user
  name: example_database
logging:
  level: info
```

Это формат приложения, не Kubernetes-манифест. Приложение должно самостоятельно читать этот файл. Пароль не включён: credentials и приватные TLS-ключи не должны находиться в ConfigMap. Для них используются Secret или внешнее secret-хранилище с контролем доступа.

![](https://ucarecdn.stepik.net/68a97426-ef57-45a0-a2a9-45cf3decb690/)

Встраивание конфигурации в образ технически возможно, но связывает её жизненный цикл со сборкой. ConfigMap делает конфигурацию отдельным объектом; он не является системой шифрования или автоматической перезагрузки приложения.

## Манифест и команды

`ns.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: example-config
```

`configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-config
  namespace: example-config
data:
  config.yaml: |
    log_level: "debug"
    database_address: db.example-app.svc.cluster.local
```

Ключ `config.yaml` содержит многострочное строковое значение. В примере DNS-адрес — конфигурационная заглушка; никакая БД автоматически не создаётся.

```bash
kubectl apply -f ns.yaml
kubectl apply -f configmap.yaml
kubectl -n example-config get configmap
kubectl -n example-config get cm
kubectl -n example-config describe cm example-config
```

`cm` — сокращение ConfigMap. `describe` показывает содержимое, ещё одна причина не хранить здесь секреты.

## Монтирование в отдельный Pod

`pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-config
  namespace: example-config
spec:
  containers:
    - name: app
      image: busybox:1.37
      command: ["/bin/sh", "-c", "cat /etc/config/* && sleep 3600"]
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: example-config
```

`volumes` описывает источник на уровне Pod, `volumeMounts` — место подключения в конкретном контейнере. Один volume может быть подключён к нескольким контейнерам Pod. Ключ ConfigMap становится файлом `/etc/config/config.yaml`.

```bash
kubectl apply -f pod.yaml
kubectl -n example-config describe pod pod-with-config
kubectl -n example-config logs pod-with-config
```

![](https://ucarecdn.stepik.net/c1424b6d-3c03-4df9-9c3d-63514199bfb3/)

Ожидаемый вывод приложения:

```text
log_level: "debug"
database_address: db.example-app.svc.cluster.local
```

Команда выводит файл один раз и затем спит. Смена содержимого ConfigMap не создаёт новый вывод в логах этого процесса.

## Монтирование в Deployment

Отдельный пример показывает перенос полей из `Pod.spec` в `Deployment.spec.template.spec`. `deployment-config.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-with-config
  namespace: example-config
spec:
  replicas: 2
  selector:
    matchLabels:
      app: config-files
  template:
    metadata:
      labels:
        app: config-files
    spec:
      containers:
        - name: app
          image: busybox:1.37
          command: ["/bin/sh", "-c", "cat /etc/config/* && sleep 3600"]
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
      volumes:
        - name: config-volume
          configMap:
            name: example-config
```

```bash
kubectl apply -f deployment-config.yaml
kubectl -n example-config exec deployment/deployment-with-config -- cat /etc/config/config.yaml
```

![](https://ucarecdn.stepik.net/229da3cf-c2ba-4343-93bd-9cf165bff364/)

Оба Pod используют один ConfigMap, но у каждого свой контейнер и mount. `exec deployment/...` выбирает один из Pod этого Deployment; для конкретного экземпляра используется его фактическое имя.

## Создание из файла и dry-run

Несекретный локальный файл можно преобразовать в ConfigMap императивно:

```bash
kubectl -n example-config create configmap generated-config --from-file=config.yaml
```

Для получения манифеста без создания ресурса:

```bash
kubectl -n example-config create configmap generated-config \
  --from-file=config.yaml --dry-run=client -o yaml > generated-config.yaml
```

Полученный файл можно проверить, сохранить в Git и применить через `kubectl apply -f generated-config.yaml`. `--dry-run=client` не подтверждает серверные политики и наличие зависимостей.

### Приватный TLS-ключ не является ConfigMap

Сохранён механизм генерации ключа и передачи файла, но исправлен неподходящий тип ресурса:

```bash
openssl genrsa -out server.key 2048
chmod 600 server.key
kubectl -n example-config create secret generic example-private-key --from-file=server.key
```

Это Secret с одним private key, **не** готовый `kubernetes.io/tls` Secret с сертификатом. Ключ не публикуется в Git, логах или ConfigMap. Base64 в Secret не является шифрованием; безопасность зависит от RBAC, хранения и выдачи секрета. Шаблон `create configmap ... --from-file=server.key` не переносится как рабочая рекомендация.

## Передача через переменные окружения

`cm-env.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: example-config
data:
  LOG_LEVEL: "debug"
  DATABASE_URL: "postgresql://db.example-app.svc.cluster.local:5432/example_database"
```

`deployment-env.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-with-env
  namespace: example-config
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-env
  template:
    metadata:
      labels:
        app: config-env
    spec:
      containers:
        - name: app
          image: busybox:1.37
          command: ["/bin/sh", "-c", "echo \"Log Level: $LOG_LEVEL, Database URL: $DATABASE_URL\" && sleep 3600"]
          envFrom:
            - configMapRef:
                name: env-config
```

```bash
kubectl apply -f cm-env.yaml
kubectl apply -f deployment-env.yaml
kubectl -n example-config logs deployment/deployment-with-env
```

`envFrom.configMapRef` переносит ключи ConfigMap в окружение контейнера. Лог содержит `Log Level: debug` и демонстрационный URL без credentials. Вывод реального URL с паролем недопустим. У двух Deployment разные selectors (`config-files` и `config-env`), чтобы они не пересекались.

## Обновления и ограничения

ConfigMap и использующий его Pod должны находиться в одном namespace. Отсутствующее обязательное имя/значение может не дать контейнеру стартовать. При монтировании на существующий каталог его исходное содержимое может быть скрыто mount.

Изменения обычного ConfigMap volume распространяются не мгновенно; приложение должно перечитать файл. `subPath` mount не получает такие обновления. Значения environment не обновляются в уже запущенном контейнере: требуется его перезапуск. Deployment сам по себе не запускает rollout от изменения ConfigMap. [Поведение ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/).
