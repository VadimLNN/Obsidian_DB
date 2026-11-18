# Job и CronJob

[[Kubernetes|Карта тем]] · [[Pod]] · [[Диагностика и отладка]]

## Job: выполнение до завершения

Job управляет Pod для конечной задачи: расчёта, миграции или обработки данных. Успешное завершение определяется выполнением нужного числа completions, а не поддержанием постоянно работающего приложения.

![](https://ucarecdn.stepik.net/a0b9deeb-caf4-4e9d-a333-bfcdfb4d3b5d/)

> [!note] Повторное выполнение
> Job может создавать замену неуспешному Pod и повторять работу. Она отличается от Deployment целью завершения, а не отсутствием повторных запусков. Задачи с внешними побочными эффектами должны учитывать повторное выполнение. [Документация Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/).

## Пример вычисления числа π

`job.yaml` создаётся в текущем namespace; далее команды явно используют `default`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-job
  namespace: default
spec:
  template:
    spec:
      containers:
        - name: pi
          image: python:3.12
          command: ["python", "-c", "from math import pi; print(f'{pi:.20f}')"]
      restartPolicy: Never
  backoffLimit: 4
```

Образ Python 3.9 из прежнего примера заменён на ветку 3.12; алгоритм не меняется. Для воспроизводимого запуска образ фиксируется проверенным digest. `restartPolicy: Never` запрещает kubelet перезапускать завершившийся контейнер в этом Pod, но Job controller может создать новый Pod. `backoffLimit` ограничивает повторы после ошибок; это не количество успешных completions.

```bash
kubectl apply -f job.yaml
kubectl -n default get jobs
kubectl -n default get pods -l job-name=pi-job
kubectl -n default logs job/pi-job
```

После успешного выполнения Job имеет `COMPLETIONS 1/1`, Pod отображается как `Completed`, вывод:

```text
3.14159265358979311600
```

Это форматирование обычного floating-point значения `math.pi` до 20 знаков, а не вычисление двадцати достоверных десятичных знаков π. Выполненная Job не запускается заново от повторного apply неизменённого объекта; для нового запуска нужен новый объект или явное пересоздание.

## CronJob: создание Job по расписанию

CronJob создаёт Jobs согласно cron-выражению. Каждая Job далее управляет собственными Pod.

![](https://ucarecdn.stepik.net/c9093eb0-c462-4eca-9879-986681477bb3/)

`cronjob.yaml`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pi-cronjob
  namespace: default
spec:
  schedule: "*/2 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: pi
              image: python:3.12
              command: ["python", "-c", "from math import pi; print(f'{pi:.20f}')"]
          restartPolicy: Never
```

```bash
kubectl apply -f cronjob.yaml
kubectl -n default get cronjobs
kubectl -n default get jobs
kubectl -n default get pods
```

`*/2 * * * *` означает запуск каждые две минуты. Поля cron: минута, час, день месяца, месяц, день недели; [crontab.guru](https://crontab.guru/) помогает читать выражения. В выводе появляются Jobs вида `pi-cronjob-<time>` и соответствующие Pod. До первого запуска `LAST SCHEDULE` может быть `<none>`.

## Прикладные сценарии и ограничения

Миграции БД, backup, уведомления и периодические расчёты — типичные задачи. [Пример PostgreSQL backup](https://github.com/pagottoo/pgdump-k8s-cronjob/blob/master/cronjob.yaml) — внешний образец, которому нужны проверка хранения credentials, версии утилит и восстановления backup.

Без дополнительной политики долгие выполнения могут пересекаться. При эксплуатационной настройке учитываются `concurrencyPolicy`, `startingDeadlineSeconds`, часовой пояс и retention завершённых Jobs. CronJob не гарантирует строго однократный запуск в точную секунду; обработчик должен быть устойчив к пропущенным или повторным запускам. [Ограничения CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/).
