---
tags:
  - devops/docker
  - reference/cheatsheet
---

# Большая шпаргалка по Docker

> Справочник по основным командам [[Docker]], Dockerfile, Docker Compose, сетям, томам, логам и очистке ресурсов.

## 1. Контейнеры

### Запуск

```bash
docker run [ОПЦИИ] IMAGE [КОМАНДА]
docker run -d -p 8080:80 --name my-container nginx
```

- `-d` — фоновый режим.
- `-p 8080:80` — порт `8080` хоста → порт `80` контейнера.
- `--name` — имя контейнера.
- `nginx` — образ.

### Просмотр

```bash
docker ps       # запущенные
docker ps -a    # все
docker ps -l    # последний созданный
```

### Управление

```bash
docker start CONTAINER
docker stop CONTAINER
docker restart CONTAINER
docker pause CONTAINER
docker unpause CONTAINER
```

`CONTAINER` — имя или ID контейнера.

### Удаление

```bash
docker rm CONTAINER
docker rm -f CONTAINER
docker container prune
```

`docker container prune` удаляет все остановленные контейнеры.

---

## 2. Образы

```bash
docker search TERM
docker pull IMAGE[:TAG]
docker images
docker image ls
docker rmi IMAGE
docker image prune
docker image prune -a
```

Примеры:

```bash
docker pull nginx
docker pull nginx:latest
docker pull postgres:13
```

- `docker image prune` — удаление неиспользуемых dangling-образов.
- `docker image prune -a` — удаление всех образов, не используемых существующими контейнерами.

---

## 3. Команды внутри контейнера

```bash
docker exec [ОПЦИИ] CONTAINER КОМАНДА
```

Интерактивная оболочка:

```bash
docker exec -it my-container bash
```

Если Bash отсутствует:

```bash
docker exec -it my-container sh
```

Одиночная команда:

```bash
docker exec my-container ls -la
```

> `docker exec` работает с уже запущенным контейнером.

---

## 4. Проброс портов

```bash
docker run -p HOST_PORT:CONTAINER_PORT IMAGE
docker run -p 8080:80 nginx
```

Схема:

```text
localhost:8080
      │
      ▼
 Docker Host
      │
      ▼
 container:80
```

Другие варианты:

```bash
docker run -p 80 nginx
docker run -p 3000-4000:3000-4000 nginx
```

Посмотреть порты:

```bash
docker ps
docker port CONTAINER
```

---

# Dockerfile

## 5. Что такое Dockerfile

`Dockerfile` — текстовый файл с инструкциями, по которым Docker собирает образ.

```text
Dockerfile
    │ docker build
    ▼
Docker Image
    │ docker run
    ▼
Container
```

## 6. Основные инструкции

### FROM

Базовый образ:

```dockerfile
FROM ubuntu:20.04
```

### LABEL

Метаданные:

```dockerfile
LABEL maintainer="your@email.com"
```

### ENV

Переменная окружения:

```dockerfile
ENV NODE_ENV=production
```

### COPY

Копирование файлов из контекста сборки:

```dockerfile
COPY . /app
```

### ADD

Добавление файлов с дополнительными возможностями:

```dockerfile
ADD https://example.com/file.tar.gz /tmp/
```

Для обычного копирования локальных файлов обычно используют `COPY`.

### WORKDIR

Рабочая директория:

```dockerfile
WORKDIR /app
```

### RUN

Команда, выполняемая при сборке образа:

```dockerfile
RUN apt-get update && apt-get install -y nginx
```

### EXPOSE

Документирует используемый приложением порт:

```dockerfile
EXPOSE 80
```

> `EXPOSE` сам по себе не публикует порт на хосте. Для этого нужен `-p` или `ports` в Compose.

### CMD

Команда по умолчанию при запуске контейнера:

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

### ENTRYPOINT

Основной исполняемый файл контейнера:

```dockerfile
ENTRYPOINT ["/app/start.sh"]
```

### RUN vs CMD vs ENTRYPOINT

| Инструкция | Когда | Назначение |
|---|---|---|
| `RUN` | `docker build` | Выполнить команду и создать слой образа |
| `CMD` | `docker run` | Команда/аргументы по умолчанию |
| `ENTRYPOINT` | `docker run` | Основной исполняемый файл |

---

## 7. Сборка образа

```bash
docker build -t my-image:latest .
docker build -f Dockerfile.dev -t my-app:dev .
```

Где в первой команде:

- `build` — сборка;
- `-t` — имя и тег;
- `my-image` — имя;
- `latest` — тег;
- `.` — контекст сборки.

После сборки:

```bash
docker image ls
docker run my-image:latest
```

---

# Docker Compose

## 8. Назначение

Docker Compose позволяет описать несколько связанных сервисов в YAML-файле и управлять ими как одним приложением.

```text
Compose project
├── web
│   └── nginx
└── db
    └── postgres
```

Современное распространённое имя файла:

```text
compose.yaml
```

Также встречаются `compose.yml`, `docker-compose.yml` и `docker-compose.yaml`.

## 9. Пример

```yaml
services:
  web:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html
    depends_on:
      - db

  db:
    image: postgres:13
    environment:
      POSTGRES_PASSWORD: example
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

Сервисы:

- `web` — Nginx;
- `db` — PostgreSQL;
- `db_data` — именованный том.

## 10. Команды Compose

```bash
docker compose up
docker compose up -d
docker compose down
docker compose ps
docker compose logs
docker compose logs -f
docker compose build
docker compose up -d --build
```

> В старых материалах часто используется `docker-compose`. Современный синтаксис — `docker compose`.

---

# Диагностика

## 11. Логи

```bash
docker logs CONTAINER
docker logs -f CONTAINER
docker logs --tail N CONTAINER
```

Пример:

```bash
docker logs --tail 100 nginx
```

## 12. Информация и мониторинг

```bash
docker inspect CONTAINER
docker stats
docker top CONTAINER
docker version
docker info
```

- `inspect` — подробная конфигурация объекта Docker.
- `stats` — использование CPU, RAM, сети и других ресурсов.
- `top` — процессы контейнера.

---

# Сети Docker

## 13. Управление

```bash
docker network ls
docker network create my-network
docker network inspect my-network
docker network rm my-network
docker network connect my-network CONTAINER
docker network disconnect my-network CONTAINER
```

## 14. Типы сетей

### bridge

Стандартный драйвер для обычных контейнеров на одном Docker-хосте.

```text
Host
└── bridge network
    ├── container A
    └── container B
```

### host

Контейнер использует сетевое пространство хоста:

```bash
docker run --network host IMAGE
```

### none

Контейнер без обычного сетевого подключения:

```bash
docker run --network none IMAGE
```

### overlay

Используется для связи между Docker-демонами, в частности в Docker Swarm.

---

# Volumes

## 15. Зачем нужны тома

Постоянные данные приложения лучше хранить отдельно от writable layer контейнера.

Основные варианты:

- **named volume** — хранилищем управляет Docker;
- **bind mount** — в контейнер монтируется конкретный путь хоста.

## 16. Управление volumes

```bash
docker volume ls
docker volume create my-volume
docker volume inspect my-volume
docker volume rm my-volume
docker volume prune
```

## 17. Named volume

```bash
docker run -v my-volume:/data nginx
```

```text
Docker volume
my-volume
    │
    ▼
container:/data
```

## 18. Bind mount

```bash
docker run -v /host/path:/container/path nginx
```

```text
Host
/host/path
    │
    ▼
Container
/container/path
```

---

# Переменные окружения

## 19. Передача переменных

```bash
docker run -e VAR=value nginx
```

Несколько:

```bash
docker run \
  -e DB_HOST=database \
  -e DB_PORT=5432 \
  IMAGE
```

Из файла:

```bash
docker run --env-file .env IMAGE
```

Пример `.env`:

```dotenv
DB_HOST=database
DB_PORT=5432
DB_NAME=app
```

---

# Ресурсы и безопасность

## 20. Ограничение ресурсов

```bash
docker run --memory=512m IMAGE
docker run --cpus=1.5 IMAGE
docker run --memory=512m --cpus=1.5 IMAGE
docker run --memory=512m --memory-swap=1g IMAGE
```

## 21. Запуск от пользователя

```bash
docker run -u 1000:1000 IMAGE
```

Формат:

```text
UID:GID
```

---

# Очистка Docker

## 22. Использование диска

```bash
docker system df
docker system df -v
```

## 23. События

```bash
docker system events
```

## 24. Очистка

```bash
docker container prune
docker image prune
docker volume prune
docker network prune
docker system prune
docker system prune -a
```

`prune` удаляет неиспользуемые ресурсы, поэтому перед выполнением стоит проверить, нужны ли они.

---

# Практические сценарии

## 25. Обычный жизненный цикл контейнера

```bash
# Скачать образ
docker pull nginx

# Создать и запустить контейнер
docker run -d --name web -p 8080:80 nginx

# Проверить
docker ps

# Логи
docker logs web

# Зайти внутрь
docker exec -it web sh

# Остановить
docker stop web

# Удалить
docker rm web

# При необходимости удалить образ
docker rmi nginx
```

## 26. Dockerfile: от исходников до контейнера

```text
Исходный код
     │
     ▼
Dockerfile
     │ docker build
     ▼
    Image
     │ docker run
     ▼
  Container
```

```bash
docker build -t my-app:1.0 .
docker run -d --name my-app -p 8080:80 my-app:1.0
docker ps
docker logs my-app
```

## 27. Docker Compose: типичный цикл

```bash
docker compose up -d
docker compose ps
docker compose logs -f
docker compose restart web
docker compose down
```

---

# 28. Команды, которые стоит знать

```bash
# Контейнеры
docker run
docker ps
docker ps -a
docker start
docker stop
docker rm
docker exec
docker logs
docker inspect

# Образы
docker pull
docker image ls
docker build
docker rmi

# Сети
docker network ls
docker network inspect

# Тома
docker volume ls
docker volume inspect

# Compose
docker compose up -d
docker compose ps
docker compose logs
docker compose down

# Диагностика
docker stats
docker info
docker system df

# Очистка
docker container prune
docker image prune
docker volume prune
docker system prune
```

---

# 29. Ментальная модель Docker

```text
Dockerfile
    │
    │ build
    ▼
┌─────────────┐
│    Image    │
└──────┬──────┘
       │ run
       ▼
┌─────────────┐
│  Container  │
└──────┬──────┘
       │
       ├──── Network
       ├──── Port mapping
       └──── Volume
```

- **Dockerfile** — рецепт сборки образа.
- **Image** — шаблон, из которого создаются контейнеры.
- **Container** — запущенный экземпляр образа.
- **Volume** — хранение данных вне writable layer контейнера.
- **Network** — сетевое взаимодействие контейнеров.
- **Port mapping** — публикация порта контейнера на хосте.
- **Docker Compose** — декларативное описание нескольких связанных сервисов.

---

# 30. Быстрая памятка

| Задача | Команда |
|---|---|
| Запустить контейнер | `docker run IMAGE` |
| Запустить в фоне | `docker run -d IMAGE` |
| Запущенные контейнеры | `docker ps` |
| Все контейнеры | `docker ps -a` |
| Остановить | `docker stop NAME` |
| Удалить | `docker rm NAME` |
| Зайти внутрь | `docker exec -it NAME sh` |
| Посмотреть логи | `docker logs NAME` |
| Скачать образ | `docker pull IMAGE` |
| Посмотреть образы | `docker image ls` |
| Собрать образ | `docker build -t NAME .` |
| Пробросить порт | `-p HOST:CONTAINER` |
| Подключить volume | `-v VOLUME:PATH` |
| Запустить Compose | `docker compose up -d` |
| Остановить Compose | `docker compose down` |
| Мониторинг | `docker stats` |
| Использование диска | `docker system df` |
| Очистка | `docker system prune` |

## Связанные заметки

- [[Docker_Best_Practices]]
