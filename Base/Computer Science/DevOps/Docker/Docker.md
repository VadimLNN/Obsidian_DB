---
title: Docker
aliases:
  - Docker конспект
tags:
  - devops
  - docker
  - containers
  - linux
  - networking
  - compose
---
## 1. Что такое Docker

**Docker** — платформа для сборки, доставки и запуска приложений в **контейнерах**.

Контейнер упаковывает приложение вместе с его зависимостями и запускает его в изолированном окружении.

Условно:

```text
Application
├── runtime
├── libraries
├── dependencies
└── configuration
```

При этом контейнер **не содержит отдельное полноценное ядро ОС**, а использует ядро хостовой системы.

На Linux:

```text
┌──────────────────────────┐
│ Container A              │
│ App + dependencies       │
├──────────────────────────┤
│ Container B              │
│ App + dependencies       │
├──────────────────────────┤
│ Linux Kernel             │
├──────────────────────────┤
│ Hardware                 │
└──────────────────────────┘
```

На Windows и macOS Linux-контейнеры обычно запускаются через небольшую Linux-VM.

---

# 2. Контейнеризация и виртуализация

## Виртуальная машина

Каждая виртуальная машина содержит собственную гостевую операционную систему и собственное ядро.

```text
App        App
│          │
Guest OS   Guest OS
│          │
VM         VM
└──── Hypervisor ────┘
        Host OS
```

## Контейнер

```text
App        App
│          │
Container  Container
└──── Linux Kernel ────┘
          Host
```

## Основные отличия

| Контейнер | Виртуальная машина |
|---|---|
| Использует ядро хоста | Имеет своё ядро |
| Обычно легче | Обычно тяжелее |
| Быстро запускается | Загружается дольше |
| Меньший overhead | Больший overhead |
| Изоляция процессов | Изоляция полноценной ОС |

> [!important]
> Контейнеризация не заменяет виртуализацию. В реальной инфраструктуре Docker часто работает **внутри виртуальных машин**.

Например:

```text
Physical Server
        ↓
Hypervisor
        ↓
Virtual Machine
        ↓
Linux
        ↓
Docker
        ↓
Containers
```

---

# 3. За счёт чего работают контейнеры

Docker использует возможности ядра [[Linux]].

Две фундаментальные технологии:

- **namespaces** — изоляция;
- **cgroups** — управление ресурсами.

---

## 3.1 Namespaces

**Namespace** ограничивает то, что процесс может видеть.

Основные namespaces:

| Namespace | Что изолирует |
|---|---|
| `pid` | процессы и PID |
| `net` | сетевые интерфейсы, IP, маршруты |
| `mnt` | точки монтирования |
| `uts` | hostname |
| `ipc` | межпроцессное взаимодействие |
| `user` | пользователей и UID/GID |

Например, процесс внутри контейнера может видеть себя как:

```text
PID 1
```

хотя на хосте у него будет другой PID.

### Важная идея

```text
namespace = что контейнер видит
```

---

## 3.2 cgroups

**Control Groups — cgroups** позволяют ограничивать ресурсы процессов.

Можно ограничить:

- CPU;
- RAM;
- I/O;
- число процессов;
- другие системные ресурсы.

Пример:

```bash
docker run --memory 512m --cpus 1 nginx
```

Контейнер сможет использовать максимум примерно:

```text
RAM: 512 MB
CPU: 1 CPU
```

### Важная идея

```text
cgroups = сколько контейнер может использовать
```

Итого:

```text
namespaces → изоляция
cgroups    → лимиты ресурсов
```

---

# 4. Архитектура Docker

Упрощённая архитектура:

```text
User
 │
 │ docker run nginx
 ▼
Docker CLI
 │
 │ Docker API
 ▼
Docker Daemon
 │
 ▼
containerd
 │
 ▼
runc
 │
 ▼
Linux Kernel
```

---

## 4.1 Docker CLI

CLI — команда:

```bash
docker
```

Примеры:

```bash
docker ps
docker run
docker build
docker pull
```

CLI сам контейнеры не запускает. Он отправляет запросы Docker Engine.

---

## 4.2 Docker Daemon

Главный демон:

```text
dockerd
```

Он управляет:

- контейнерами;
- образами;
- сетями;
- томами;
- сборками;
- взаимодействием с registry.

Проверить состояние Docker:

```bash
systemctl status docker
```

Запустить:

```bash
sudo systemctl start docker
```

Добавить в автозапуск:

```bash
sudo systemctl enable docker
```

---

## 4.3 Docker API

Docker CLI общается с Docker Daemon через API.

```text
docker ps
   ↓
Docker CLI
   ↓
Docker API
   ↓
dockerd
```

Благодаря этому Docker можно использовать не только из терминала, но и из:

- CI/CD;
- IDE;
- скриптов;
- панелей управления;
- оркестраторов.

---

## 4.4 containerd и runc

Современный Docker использует более низкоуровневые компоненты.

### containerd

Отвечает за жизненный цикл контейнеров:

- загрузку image;
- создание контейнеров;
- запуск;
- остановку;
- snapshot'ы файловой системы.

### runc

Низкоуровневый OCI runtime, который непосредственно создаёт контейнер средствами ядра Linux.

Упрощённо:

```text
Docker
  ↓
containerd
  ↓
runc
  ↓
Linux namespaces + cgroups
```

---

# 5. Docker Image

**Image** — неизменяемый шаблон, из которого создаются контейнеры.

Примеры:

```text
nginx:latest
nginx:1.28
postgres:17
ubuntu:24.04
golang:1.25
```

Один image может использоваться многими контейнерами:

```text
              nginx:1.28
                 │
       ┌─────────┼─────────┐
       ↓         ↓         ↓
 container1 container2 container3
```

---

# 6. Image и Container

Очень важно не путать:

```text
Image     → шаблон
Container → запущенный или остановленный экземпляр image
```

Условная аналогия:

```text
класс  → Image
объект → Container
```

Но это только аналогия.

---

# 7. Слои Docker Image

Docker image состоит из **слоёв**.

Например:

```Dockerfile
FROM ubuntu:24.04
RUN apt-get update
RUN apt-get install -y nginx
COPY index.html /var/www/html/
```

Логически получится примерно:

```text
Layer 4: COPY index.html
Layer 3: install nginx
Layer 2: apt-get update
Layer 1: ubuntu:24.04
```

Слои image обычно read-only.

При запуске контейнера сверху добавляется отдельный writable layer:

```text
Container writable layer
--------------------------
Image layer 4
Image layer 3
Image layer 2
Image layer 1
```

> [!important]
> Изменения внутри контейнера не меняют исходный image.

---

# 8. Copy-on-Write

Docker использует идею **Copy-on-Write**.

Если контейнер читает файл — используется файл из image layer.

Если контейнер изменяет файл — Docker копирует его в writable layer контейнера и меняет уже там.

Это позволяет нескольким контейнерам эффективно использовать одни и те же слои image.

---

# 9. Docker cache

Docker кэширует результаты сборки.

Пример:

```Dockerfile
FROM golang:1.25

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN go build -o app .
```

Такой порядок хороший.

Почему?

Потому что зависимости меняются реже, чем исходный код.

При изменении `.go`-файла Docker может повторно использовать слой:

```Dockerfile
RUN go mod download
```

---

## Плохой вариант

```Dockerfile
COPY . .
RUN go mod download
RUN go build -o app .
```

При изменении любого файла инвалидируется кэш после `COPY . .`.

---

# 10. Build context

Команда:

```bash
docker build -t myapp .
```

Точка:

```text
.
```

означает **build context**, а не просто местонахождение Dockerfile.

Docker получает доступ к файлам внутри этого контекста.

Например:

```text
project/
├── Dockerfile
├── src/
├── README.md
├── .git/
└── node_modules/
```

Если build context — `project/`, все эти файлы потенциально могут попасть в контекст сборки.

Поэтому нужен `.dockerignore`.

---

# 11. .dockerignore

`.dockerignore` исключает ненужные файлы из build context.

Пример:

```gitignore
.git
.gitignore

node_modules
vendor

*.log
*.tmp

.env
.env.*

.idea
.vscode

build
dist
```

Это:

- уменьшает build context;
- ускоряет сборку;
- снижает размер image;
- уменьшает риск случайно положить секреты в image.

> [!warning]
> Никогда не копируй `.env`, SSH-ключи, API tokens и другие секреты в image.

---

# 12. Dockerfile

**Dockerfile** — декларативное описание сборки Docker image.

Пример:

```Dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "main.py"]
```

Собрать:

```bash
docker build -t myapp:1.0 .
```

---

# 13. Основные инструкции Dockerfile

## FROM

Задаёт базовый image.

```Dockerfile
FROM ubuntu:24.04
```

или:

```Dockerfile
FROM golang:1.25
```

Почти каждый Dockerfile начинается с `FROM`.

---

## WORKDIR

Устанавливает рабочую директорию:

```Dockerfile
WORKDIR /app
```

После этого:

```Dockerfile
COPY . .
RUN ./build.sh
```

будут выполняться относительно `/app`.

---

## COPY

Копирует файлы из build context:

```Dockerfile
COPY main.py /app/main.py
```

или:

```Dockerfile
COPY . .
```

---

## ADD

Похож на `COPY`, но имеет дополнительные возможности.

Например, может распаковывать локальный tar-архив.

В большинстве случаев лучше использовать:

```Dockerfile
COPY
```

потому что его поведение проще и очевиднее.

---

## RUN

Выполняет команду **во время сборки image**.

```Dockerfile
RUN apt-get update
```

Например:

```Dockerfile
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

---

## ENV

Создаёт переменную окружения:

```Dockerfile
ENV APP_ENV=production
```

Она будет доступна внутри контейнера.

---

## ARG

Переменная только для этапа сборки:

```Dockerfile
ARG APP_VERSION=1.0
```

Использование:

```Dockerfile
RUN echo "$APP_VERSION"
```

Передача:

```bash
docker build \
  --build-arg APP_VERSION=2.0 \
  -t myapp .
```

---

## ENV vs ARG

| ARG | ENV |
|---|---|
| Используется при build | Используется и при runtime |
| Обычно не предназначен для runtime | Доступен контейнеру |
| Передаётся через `--build-arg` | Можно переопределить через `-e` |

---

## EXPOSE

Документирует порт приложения:

```Dockerfile
EXPOSE 8080
```

Но **не публикует порт наружу**.

Чтобы открыть доступ с хоста:

```bash
docker run -p 8080:8080 myapp
```

---

## USER

Указывает пользователя:

```Dockerfile
USER app
```

Хорошая практика — не запускать приложение от root.

---

## LABEL

Добавляет метаданные:

```Dockerfile
LABEL version="1.0"
LABEL description="Example backend"
```

---

## VOLUME

Объявляет точку монтирования:

```Dockerfile
VOLUME ["/data"]
```

На практике volumes чаще задаются при `docker run` или в Compose.

---

# 14. CMD и ENTRYPOINT

Одна из самых частых тем на собеседованиях.

## CMD

Задаёт команду по умолчанию:

```Dockerfile
CMD ["python", "main.py"]
```

Можно переопределить:

```bash
docker run myapp python test.py
```

---

## ENTRYPOINT

Определяет основной executable контейнера:

```Dockerfile
ENTRYPOINT ["python", "main.py"]
```

Команда после имени image обычно становится аргументами.

---

## Вместе

```Dockerfile
ENTRYPOINT ["python", "main.py"]
CMD ["--port", "8000"]
```

По умолчанию:

```text
python main.py --port 8000
```

Можно изменить аргументы:

```bash
docker run myapp --port 9000
```

Получится:

```text
python main.py --port 9000
```

### Кратко

```text
ENTRYPOINT → что запускаем
CMD        → аргументы по умолчанию
```

---

# 15. Exec form и shell form

## Exec form

```Dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

Предпочтительный вариант.

---

## Shell form

```Dockerfile
CMD nginx -g "daemon off;"
```

Фактически запускается через shell.

Например:

```text
/bin/sh -c nginx ...
```

Exec form лучше передаёт Linux signals основному процессу.

---

# 16. PID 1 внутри контейнера

Главный процесс контейнера обычно становится:

```text
PID 1
```

Например:

```Dockerfile
CMD ["./server"]
```

Когда этот процесс завершается — контейнер останавливается.

> [!important]
> Контейнер — это не маленькая VM. Обычно он существует ровно столько, сколько работает его основной процесс.

---

# 17. Сборка image

Команда:

```bash
docker build -t myapp .
```

С тегом:

```bash
docker build -t myapp:1.0 .
```

С другим Dockerfile:

```bash
docker build -f Dockerfile.prod -t myapp:prod .
```

---

# 18. Список images

```bash
docker images
```

или:

```bash
docker image ls
```

---

# 19. Удаление image

```bash
docker rmi myapp:1.0
```

или:

```bash
docker image rm myapp:1.0
```

---

# 20. Docker history

Посмотреть слои image:

```bash
docker history nginx
```

Полезно для понимания:

- размера слоёв;
- команд сборки;
- структуры image.

---

# 21. Запуск контейнера

Самый простой запуск:

```bash
docker run nginx
```

Docker:

1. проверяет наличие image локально;
2. если его нет — скачивает;
3. создаёт контейнер;
4. создаёт writable layer;
5. настраивает namespaces;
6. подключает сеть;
7. подключает volumes;
8. запускает основной процесс.

---

# 22. Полезные параметры docker run

## Имя

```bash
docker run --name web nginx
```

---

## Detached mode

```bash
docker run -d nginx
```

Контейнер работает в фоне.

---

## Автоматическое удаление

```bash
docker run --rm nginx
```

После остановки контейнер удаляется.

Удобно для временных контейнеров.

---

## Интерактивный режим

```bash
docker run -it ubuntu bash
```

Где:

```text
-i → stdin
-t → pseudo-TTY
```

---

## Переменные окружения

```bash
docker run \
  -e APP_ENV=production \
  myapp
```

---

## Ограничение памяти

```bash
docker run --memory 512m myapp
```

---

## CPU

```bash
docker run --cpus 1.5 myapp
```

---

# 23. docker ps

Работающие контейнеры:

```bash
docker ps
```

Все:

```bash
docker ps -a
```

---

# 24. Жизненный цикл контейнера

```text
create
  ↓
running
  ↓
stopped
  ↓
removed
```

Основные команды:

```bash
docker create
docker start
docker stop
docker restart
docker kill
docker rm
```

---

# 25. docker logs

Посмотреть stdout/stderr контейнера:

```bash
docker logs web
```

Следить в реальном времени:

```bash
docker logs -f web
```

Последние 100 строк:

```bash
docker logs --tail 100 web
```

---

# 26. docker exec

Запустить команду внутри работающего контейнера:

```bash
docker exec web ls
```

Открыть shell:

```bash
docker exec -it web sh
```

или, если есть bash:

```bash
docker exec -it web bash
```

> [!note]
> `docker exec` не создаёт новый контейнер. Он запускает дополнительный процесс внутри существующего.

---

# 27. docker inspect

Очень важная команда для диагностики:

```bash
docker inspect web
```

Показывает:

- IP;
- mounts;
- networks;
- environment;
- command;
- image;
- container state.

---

# 28. Docker Networking

Каждый контейнер получает собственный network namespace.

Внутри него могут быть свои:

```text
interface
IP
routing table
ports
```

На Linux Docker обычно использует:

- network namespaces;
- virtual Ethernet (`veth`);
- bridge;
- маршрутизацию;
- NAT.

---

# 29. Bridge network

Стандартная Docker-сеть:

```text
Host
│
├── docker0 bridge
│      │
│      ├── container A
│      └── container B
│
└── eth0
```

Посмотреть сети:

```bash
docker network ls
```

---

# 30. Типы Docker Network

Основные:

```text
bridge
host
none
overlay
macvlan
```

---

## bridge

Основной вариант для контейнеров на одном Docker host.

Создать:

```bash
docker network create backend
```

Запустить контейнер:

```bash
docker run \
  --network backend \
  --name api \
  myapp
```

---

## host

```bash
docker run --network host nginx
```

Контейнер использует сеть хоста напрямую.

Нет отдельного network namespace в обычном смысле для сетевого стека.

---

## none

```bash
docker run --network none alpine
```

Контейнер практически без сети.

---

## overlay

Используется для сети между несколькими Docker hosts.

Часто встречается в:

```text
Docker Swarm
```

В Kubernetes используется другая модель через CNI.

---

## macvlan

Контейнер может выглядеть в физической сети как отдельное устройство со своим MAC-адресом.

Используется реже.

---

# 31. User-defined bridge network

Лучше создавать собственную bridge network:

```bash
docker network create app-network
```

Контейнеры:

```bash
docker run -d \
  --name db \
  --network app-network \
  postgres

docker run -d \
  --name backend \
  --network app-network \
  myapp
```

Теперь backend может обращаться к БД по имени:

```text
db:5432
```

Docker предоставляет встроенный DNS.

---

# 32. localhost внутри контейнера

Очень важный момент.

Внутри контейнера:

```text
localhost
```

означает **сам контейнер**.

Например:

```text
backend container
localhost:5432
```

не указывает на PostgreSQL в другом контейнере.

Нужно:

```text
db:5432
```

если контейнер БД называется `db` и находится в той же Docker network.

---

# 33. Публикация портов

Контейнер может слушать:

```text
0.0.0.0:8080
```

внутри своей сети.

Но хост автоматически доступа к этому порту не получает.

Используется:

```bash
docker run -p 8080:80 nginx
```

Где:

```text
8080 → порт хоста
80   → порт контейнера
```

Схема:

```text
Browser
   ↓
localhost:8080
   ↓
Host port 8080
   ↓
Docker NAT
   ↓
Container port 80
```

---

## EXPOSE != -p

```Dockerfile
EXPOSE 80
```

только документирует порт.

А:

```bash
docker run -p 8080:80 nginx
```

реально публикует его.

---

# 34. Docker Storage

Файлы, записанные в writable layer контейнера, связаны с жизненным циклом контейнера.

Если данные нужно сохранять независимо от контейнера — используется persistent storage.

Основные варианты:

```text
volume
bind mount
tmpfs
```

---

# 35. Volume

Docker-managed storage.

Создать:

```bash
docker volume create postgres-data
```

Посмотреть:

```bash
docker volume ls
```

Использовать:

```bash
docker run \
  -v postgres-data:/var/lib/postgresql/data \
  postgres
```

Условно:

```text
Docker Host
│
└── Docker Volume
      ↓
Container
/var/lib/postgresql/data
```

---

# 36. Bind mount

Папка хоста напрямую подключается внутрь контейнера.

```bash
docker run \
  -v /home/user/site:/usr/share/nginx/html \
  nginx
```

Современный синтаксис:

```bash
docker run \
  --mount type=bind,source=/home/user/site,target=/usr/share/nginx/html \
  nginx
```

Полезно для:

- разработки;
- конфигов;
- исходников;
- локальных файлов.

---

# 37. Volume vs Bind Mount

| Volume | Bind mount |
|---|---|
| Управляется Docker | Управляется пользователем |
| Хорошо для данных приложения | Хорошо для файлов хоста |
| Путь хоста обычно неважен | Путь известен явно |
| Удобен для БД | Удобен для config/source |

---

# 38. tmpfs

Хранит данные в RAM:

```bash
docker run \
  --tmpfs /tmp \
  myapp
```

После остановки данные исчезают.

---

# 39. Docker Registry

**Registry** — хранилище Docker images.

Примеры:

```text
Docker Hub
GitHub Container Registry
GitLab Container Registry
Harbor
Private Registry
```

Типичный DevOps workflow:

```text
Developer
   ↓
Git
   ↓
CI
   ↓
docker build
   ↓
docker push
   ↓
Registry
   ↓
Server / Kubernetes
   ↓
docker pull
```

Registry связывает:

```text
build → delivery → deployment
```

---

# 40. docker pull

Скачать image:

```bash
docker pull nginx
```

Конкретную версию:

```bash
docker pull nginx:1.28
```

---

# 41. docker push

Отправить image:

```bash
docker push username/myapp:1.0
```

---

# 42. Docker tags

Образ обычно имеет имя:

```text
repository:tag
```

Например:

```text
nginx:1.28
postgres:17
myapp:1.4.2
```

Создать дополнительный tag:

```bash
docker tag myapp:1.0 vadim/myapp:1.0
```

---

# 43. latest

`latest` — обычный tag.

Он **не означает автоматически самую новую версию**.

Например:

```bash
docker build -t myapp:latest .
```

Тегом `latest` управляет тот, кто публикует image.

> [!warning]
> Для production лучше использовать конкретные версии, а ещё лучше immutable digest.

---

# 44. Image digest

Image может идентифицироваться digest'ом:

```text
sha256:...
```

Например:

```text
nginx@sha256:abcd...
```

Tag может быть изменён:

```text
myapp:latest
```

Digest — нет.

Это делает deployment более воспроизводимым.

---

# 45. Multi-stage build

Multi-stage build позволяет использовать несколько этапов сборки.

Особенно полезно для компилируемых языков.

Пример Go:

```Dockerfile
FROM golang:1.25 AS builder

WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 go build -o app ./cmd/app


FROM alpine:3.22

WORKDIR /app

COPY --from=builder /src/app ./app

CMD ["./app"]
```

На первом этапе находятся:

```text
Go compiler
source code
build tools
dependencies
```

В финальный image копируется только:

```text
binary
```

---

## Зачем multi-stage build

Без него:

```text
final image =
compiler +
source +
dependencies +
binary
```

С ним:

```text
final image =
runtime +
binary
```

Плюсы:

- меньше размер;
- меньше attack surface;
- быстрее передача;
- меньше лишних файлов.

---

# 46. Dockerfile Best Practices

Основные правила:

1. Использовать минимальный подходящий base image.
2. Не запускать приложение от root без необходимости.
3. Использовать `.dockerignore`.
4. Правильно располагать инструкции для эффективного cache.
5. Объединять связанные `RUN`.
6. Удалять временные файлы в том же `RUN`.
7. Не хранить секреты внутри image.
8. Использовать multi-stage build.
9. Использовать конкретные версии image.
10. Стараться иметь один основной процесс на контейнер.
11. Писать логи в stdout/stderr.
12. Хранить постоянные данные в volumes.
13. Делать контейнер максимально disposable.

---

# 47. Почему нельзя хранить секреты в Dockerfile

Плохо:

```Dockerfile
ENV DB_PASSWORD=supersecret
```

или:

```Dockerfile
COPY .env .
```

Image может попасть:

```text
Registry
CI cache
другому разработчику
production node
backup
```

Секреты должны передаваться отдельно.

Например:

```bash
docker run \
  -e DB_PASSWORD="$DB_PASSWORD" \
  myapp
```

А в production часто используются secret-management системы, например [[Vault]] или секреты оркестратора.

---

# 48. Docker Compose

**Docker Compose** позволяет декларативно описать несколько контейнеров в YAML.

Современный стандартный файл:

```text
compose.yaml
```

Также часто встречаются:

```text
docker-compose.yml
docker-compose.yaml
```

---

# 49. Зачем Compose

Реальное приложение редко состоит из одного контейнера.

Например:

```text
nginx
  ↓
backend
  ↓
postgres
  ↓
redis
```

Без Compose нужно писать много команд:

```bash
docker network create app
docker volume create db-data
docker run ...
docker run ...
docker run ...
```

С Compose:

```bash
docker compose up -d
```

---

# 50. Пример Compose

```yaml
services:
  backend:
    build: .
    ports:
      - "8080:8080"
    environment:
      DB_HOST: db
      DB_PORT: 5432
    depends_on:
      - db

  db:
    image: postgres:17
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: password
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

---

# 51. services

Главный блок:

```yaml
services:
```

Каждый service обычно соответствует контейнеру.

Например:

```yaml
services:
  frontend:
  backend:
  db:
  redis:
```

---

# 52. image

Использовать готовый image:

```yaml
services:
  db:
    image: postgres:17
```

---

# 53. build

Собрать свой image:

```yaml
services:
  backend:
    build: .
```

Можно указать Dockerfile:

```yaml
services:
  backend:
    build:
      context: .
      dockerfile: Dockerfile.prod
```

---

# 54. ports

```yaml
ports:
  - "8080:80"
```

То есть:

```text
host:container
```

---

# 55. environment

```yaml
environment:
  APP_ENV: production
  DB_HOST: db
```

---

# 56. env_file

```yaml
env_file:
  - .env
```

Но `.env` с секретами не следует коммитить.

---

# 57. volumes

```yaml
volumes:
  - db-data:/var/lib/postgresql/data
```

Bind mount:

```yaml
volumes:
  - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

---

# 58. networks

```yaml
services:
  backend:
    networks:
      - internal

  db:
    networks:
      - internal

networks:
  internal:
```

---

# 59. depends_on

```yaml
depends_on:
  - db
```

Означает зависимость порядка запуска.

Но это **не гарантирует**, что БД уже полностью готова принимать соединения.

Для проверки готовности используют:

- retry в приложении;
- healthcheck;
- отдельную orchestration logic.

---

# 60. healthcheck

Пример:

```yaml
services:
  db:
    image: postgres:17
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5
```

---

# 61. Docker Compose Commands

Запустить:

```bash
docker compose up
```

В фоне:

```bash
docker compose up -d
```

Остановить:

```bash
docker compose stop
```

Удалить контейнеры и сеть:

```bash
docker compose down
```

Также удалить volumes:

```bash
docker compose down -v
```

Собрать:

```bash
docker compose build
```

Посмотреть контейнеры:

```bash
docker compose ps
```

Логи:

```bash
docker compose logs
```

Follow:

```bash
docker compose logs -f
```

Выполнить команду:

```bash
docker compose exec backend sh
```

---

# 62. docker compose down vs stop

```text
docker compose stop
```

останавливает контейнеры, но не удаляет их.

```text
docker compose down
```

обычно удаляет:

- контейнеры;
- Compose network.

Volumes по умолчанию остаются.

---

# 63. restart policy

Для обычного Docker:

```bash
docker run \
  --restart unless-stopped \
  nginx
```

В Compose:

```yaml
services:
  backend:
    restart: unless-stopped
```

Популярные варианты:

```text
no
always
on-failure
unless-stopped
```

---

# 64. Docker и логи

Хорошее контейнерное приложение пишет:

```text
stdout
stderr
```

Тогда Docker может собирать вывод:

```bash
docker logs container
```

Не стоит без необходимости складывать основные application logs только в локальный файл внутри контейнера.

---

# 65. Docker и конфигурация

Image должен быть максимально одинаковым между окружениями:

```text
dev
staging
production
```

Меняться должна конфигурация:

```text
environment variables
mounted config
secrets
```

Условно:

```text
same image
   ├── dev config
   ├── staging config
   └── production config
```

---

# 66. Immutable infrastructure

Хорошая контейнерная модель:

```text
не заходить в production container и не чинить его руками
```

Вместо:

```text
docker exec
apt install
nano config
```

лучше:

```text
изменить Dockerfile/config
      ↓
build new image
      ↓
push
      ↓
redeploy
```

Это делает систему воспроизводимой.

---

# 67. Docker и CI/CD

Docker особенно важен вместе с [CI/CD](../CI%20CD.md).

Подробно о средах jobs, service-контейнерах, публикации образов и ограничениях изоляции: [Docker в CI/CD](../CI%20CD/Docker%20в%20CI%20CD.md). Образ для выполнения CI-job и собранный образ приложения — разные объекты; наличие `image` в YAML не выполняет `docker build` автоматически.

Типичный pipeline:

```text
git push
   ↓
test
   ↓
docker build
   ↓
docker tag
   ↓
docker push
   ↓
registry
   ↓
deploy
```

Например:

```text
GitLab CI
   ↓
GitLab Container Registry
   ↓
Server / Kubernetes
```

---

# 68. Docker и Kubernetes

Docker и [[Kubernetes]] — разные уровни.

Docker:

```text
создать и запустить контейнер
```

Kubernetes:

```text
управлять большим количеством контейнеров
на нескольких серверах
```

Kubernetes решает:

- scheduling;
- replicas;
- rolling updates;
- service discovery;
- load balancing;
- self-healing;
- secrets/configs;
- storage;
- networking.

> [!note]
> Современный Kubernetes не требует Docker Engine как runtime. Он может работать напрямую с containerd и другими CRI-compatible runtimes.

---

# 69. Docker и Podman

Кроме Docker существуют:

```text
Podman
Buildah
containerd
CRI-O
```

### Podman

Похож на Docker CLI:

```bash
podman run nginx
```

Особенности:

- daemonless;
- хорошо поддерживает rootless;
- OCI-compatible.

### Buildah

Фокусируется на сборке OCI images.

### containerd

Контейнерный runtime более низкого уровня.

---

# 70. OCI

**Open Container Initiative** определяет стандарты контейнеров.

Основные спецификации:

```text
OCI Image Specification
OCI Runtime Specification
```

Благодаря OCI image не обязательно жёстко привязан именно к Docker.

---

# 71. Полезные команды

## Images

```bash
docker images
docker pull nginx
docker build -t myapp:1.0 .
docker tag myapp:1.0 user/myapp:1.0
docker push user/myapp:1.0
docker rmi myapp:1.0
docker history myapp:1.0
```

## Containers

```bash
docker run
docker ps
docker ps -a
docker stop
docker start
docker restart
docker kill
docker rm
docker logs
docker exec
docker inspect
```

## Networks

```bash
docker network ls
docker network create app
docker network inspect app
docker network rm app
```

## Volumes

```bash
docker volume ls
docker volume create data
docker volume inspect data
docker volume rm data
```

## Compose

```bash
docker compose up -d
docker compose down
docker compose ps
docker compose logs -f
docker compose build
docker compose exec
```

---

# 72. Очистка Docker

Посмотреть использование диска:

```bash
docker system df
```

Удалить неиспользуемые данные:

```bash
docker system prune
```

Более агрессивно:

```bash
docker system prune -a
```

> [!warning]
> Перед `prune` нужно понимать, какие images, containers и networks будут удалены.

Удалить неиспользуемые volumes:

```bash
docker volume prune
```

---

# 73. Диагностика контейнера

Если контейнер не работает, полезный порядок:

```text
1. docker ps -a
2. docker logs <container>
3. docker inspect <container>
4. docker exec -it <container> sh
5. docker network inspect ...
6. проверить ports
7. проверить volumes
8. проверить environment
```

---

# 74. Контейнер сразу завершается

Пример:

```bash
docker run ubuntu
```

Контейнер почти сразу завершится.

Почему?

Потому что основной процесс завершился.

А:

```bash
docker run -it ubuntu bash
```

будет работать, пока работает `bash`.

---

# 75. Проверка сети

Посмотреть IP:

```bash
docker inspect container
```

Проверить DNS между контейнерами:

```bash
docker exec backend getent hosts db
```

Проверить порт:

```bash
docker exec backend curl http://api:8080
```

---

# 76. Практический пример: Nginx

Создать:

```text
site/
├── Dockerfile
└── index.html
```

Dockerfile:

```Dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html
```

Build:

```bash
docker build -t my-nginx:1.0 .
```

Run:

```bash
docker run \
  --name web \
  -d \
  -p 8080:80 \
  my-nginx:1.0
```

Открыть:

```text
http://localhost:8080
```

---

# 77. Практический пример: Nginx + bind mount

Без собственного image:

```bash
docker run \
  --name web \
  -d \
  -p 8080:80 \
  -v "$PWD/index.html:/usr/share/nginx/html/index.html:ro" \
  nginx:alpine
```

Здесь:

```text
host file
    ↓
bind mount
    ↓
container file
```

---

# 78. Практический пример Compose: Backend + PostgreSQL

```yaml
services:
  backend:
    build: .
    ports:
      - "8080:8080"
    environment:
      DB_HOST: db
      DB_PORT: 5432
      DB_NAME: app
      DB_USER: app
      DB_PASSWORD: password
    depends_on:
      - db

  db:
    image: postgres:17
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: password
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

Связь:

```text
localhost:8080
      ↓
backend container
      ↓
db:5432
      ↓
PostgreSQL container
      ↓
db-data volume
```

---

# 79. Практика из roadmap

Хорошая последовательность:

## Этап 1 — один контейнер

Взять уже существующее приложение и написать:

```text
Dockerfile
.dockerignore
```

Далее:

```bash
docker build
docker run
docker logs
docker exec
```

---

## Этап 2 — приложение + БД

Например:

```text
Go backend
PostgreSQL
```

Настроить:

- отдельную network;
- environment variables;
- volume для PostgreSQL;
- port mapping.

---

## Этап 3 — Docker Compose

Перенести всё в:

```text
compose.yaml
```

Добиться запуска через одну команду:

```bash
docker compose up -d
```

---

## Этап 4 — Nginx

Добавить:

```text
Nginx
```

Не просто использовать:

```bash
docker run nginx
```

а:

- создать свой `nginx.conf`;
- подключить его bind mount;
- настроить reverse proxy;
- настроить port mapping;
- отдать свою static page.

---

## Этап 5 — мониторинг

Compose stack:

```text
Grafana
    ↑
Prometheus
    ↑
Node Exporter
```

Дополнительно:

```text
cAdvisor
```

для container metrics.

---

# 80. Что нужно знать для roadmap

Минимум:

- что такое container;
- отличие container от VM;
- namespaces;
- cgroups;
- Docker architecture;
- image;
- container;
- Dockerfile;
- build context;
- `.dockerignore`;
- layers;
- cache;
- registry;
- tags;
- `docker build`;
- `docker run`;
- `docker ps`;
- `docker logs`;
- `docker exec`;
- ports;
- volumes;
- Docker networks;
- Docker Compose.

Хороший уровень:

- multi-stage build;
- image optimization;
- Dockerfile best practices;
- healthchecks;
- restart policies;
- user-defined networks;
- Docker DNS;
- bind mounts vs volumes;
- immutable images;
- container security basics;
- Registry + CI/CD.

---

# 81. Вопросы для собеседования

## 1. Чем контейнеризация отличается от виртуализации?

Контейнер использует ядро хоста и изолирует процессы.

VM содержит полноценную гостевую ОС и собственное ядро.

---

## 2. За счёт чего работает изоляция контейнеров?

В первую очередь:

```text
Linux namespaces
cgroups
```

Namespaces дают изоляцию, cgroups — ограничение ресурсов.

---

## 3. Image и container — в чём разница?

```text
Image     → immutable template
Container → instance image + writable layer + runtime state
```

---

## 4. Что делает Dockerfile?

Описывает процесс создания Docker image.

---

## 5. COPY и ADD — разница?

`COPY` просто копирует файлы.

`ADD` имеет дополнительные возможности, например автоматическую распаковку локальных архивов.

Обычно лучше использовать `COPY`.

---

## 6. CMD и ENTRYPOINT — разница?

`ENTRYPOINT` определяет основной executable.

`CMD` задаёт команду или аргументы по умолчанию.

---

## 7. EXPOSE публикует порт?

Нет.

Публикация выполняется через:

```bash
-p host:container
```

---

## 8. Зачем нужен multi-stage build?

Чтобы отделить build environment от runtime environment.

Это уменьшает:

- размер image;
- количество лишних инструментов;
- attack surface.

---

## 9. Что такое Docker layer?

Неизменяемая часть image filesystem.

Images состоят из набора слоёв.

---

## 10. Что такое build context?

Набор файлов, доступных Docker во время build.

---

## 11. Зачем `.dockerignore`?

Чтобы исключать ненужные и чувствительные файлы из build context.

---

## 12. Volume и bind mount — разница?

Volume управляется Docker.

Bind mount подключает конкретный путь хоста.

---

## 13. Почему данные БД нельзя оставлять только внутри контейнера?

После удаления контейнера его writable layer удаляется.

Persistent data нужно хранить в volume или другом внешнем storage.

---

## 14. Как контейнеры находят друг друга?

В user-defined Docker network работает встроенный DNS.

Можно обращаться по имени контейнера/service:

```text
db:5432
```

---

## 15. Что такое Registry?

Хранилище Docker images.

---

## 16. Что значит `nginx:1.28`?

```text
nginx → repository
1.28  → tag
```

---

## 17. Что такое `latest`?

Обычный tag с именем `latest`.

Он сам по себе не гарантирует, что image действительно самый новый.

---

## 18. Почему контейнер останавливается?

Потому что завершился его основной процесс.

---

## 19. Что делает `docker exec`?

Запускает новый процесс внутри уже работающего контейнера.

---

## 20. Что делает Docker Compose?

Декларативно описывает и запускает многоконтейнерные приложения.

---

# 82. Ментальная модель Docker

Главная цепочка:

```text
Dockerfile
    ↓
docker build
    ↓
Image
    ↓
docker run
    ↓
Container
```

Для production:

```text
Source Code
    ↓
Dockerfile
    ↓
CI
    ↓
docker build
    ↓
Image
    ↓
docker push
    ↓
Registry
    ↓
Deployment
    ↓
Container
```

Для нескольких сервисов:

```text
compose.yaml
    ↓
Docker Compose
    ↓
┌───────────┬────────────┬─────────────┐
│ backend   │ postgres   │ nginx       │
│ container │ container  │ container   │
└───────────┴────────────┴─────────────┘
       │           │
       └── network ┘
             │
           volume
```

---

# 83. Главное, что нужно запомнить

> [!summary]
> **Docker Image** — шаблон.
>
> **Container** — экземпляр image.
>
> **Dockerfile** — рецепт сборки image.
>
> **Registry** — хранилище images.
>
> **Volume** — постоянные данные.
>
> **Network** — связь контейнеров.
>
> **Docker Compose** — описание нескольких сервисов.
>
> **Namespaces** — изоляция.
>
> **cgroups** — лимиты ресурсов.
>
> **Multi-stage build** — разделение build и runtime для уменьшения image.

---

# 84. Связи с другими темами DevOps

```text
[[Linux]]
    ↓
namespaces + cgroups + processes
    ↓
Docker
    ↓
Container Registry
    ↓
[[CI-CD]]
    ↓
[[Kubernetes]]
```

И отдельно:

```text
[[Сети]]
  ↓
ports
DNS
routing
NAT
bridge
  ↓
Docker Networking
```

```text
[[Linux]]
  ↓
mount
filesystem
permissions
  ↓
Docker Volumes
```

```text
[[Git]]
  ↓
source code
  ↓
[[CI-CD]]
  ↓
Docker build
  ↓
Registry
  ↓
Deploy
```
