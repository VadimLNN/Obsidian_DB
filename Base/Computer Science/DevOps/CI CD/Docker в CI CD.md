---
tags:
  - devops
  - ci-cd
  - docker
---

# Docker в CI/CD

[Навигация](README.md) · Источник: [оригинал курса](../CI%20CD%20курс%20ориг.md), разделы «Зачем Docker в CI/CD?» и «Использование Docker образов в пайплайнах».

Основы image/container, Dockerfile, networks и volumes уже описаны в [конспекте Docker](../Docker/Docker.md). Здесь рассматривается их роль в pipeline.

## Три разные роли контейнеров

| Роль | Пример | Для чего нужен |
|---|---|---|
| Среда выполнения job | `image: node:22-bookworm-slim` | Дать командам согласованные Node, npm и системные библиотеки |
| Вспомогательный сервис | `postgres:15`, `redis:7` | Поднять временную БД/кэш для интеграционных тестов |
| Результат сборки приложения | `registry.example.com/team/app:<version>` | Доставить приложение вместе с runtime в staging и production |

Образ Node для тестов **не становится образом приложения автоматически**. GitLab `image` запускает job в готовом образе, а `docker build` создаёт новый образ из Dockerfile и контекста. Docker CLI внутри контейнера не означает наличие доступного Docker daemon.

Docker уменьшает расхождения окружений, фиксируя пользовательскую среду. Он не фиксирует ядро хоста, архитектуру CPU, внешние сервисы, DNS, секреты, данные и ограничения памяти. Поэтому «работает в контейнере CI» не гарантирует «работает в production».

## Образы для jobs

GitLab Docker executor подготавливает контейнер из `image`, делает исходники доступными в рабочем каталоге и исполняет команды. Shell executor игнорирует этот способ выбора среды: команды работают на хосте. Jenkins использует Docker Pipeline plugin и Docker-capable agent; workspace монтируется в контейнер. [GitLab Docker executor](https://docs.gitlab.com/runner/executors/docker/), [Docker agent в Jenkins](https://www.jenkins.io/doc/book/pipeline/docker/).

В примерах курса встречаются Node 16/18. Для нового учебного проекта выбран Node 22, который на дату подготовки указан как LTS; старые EOL-ветки не используются как рекомендуемая основа. [Сроки поддержки Node.js](https://nodejs.org/en/about/previous-releases).

`node:22-bookworm-slim` фиксирует major-версию и семейство ОС, но **не точное содержимое**. Тег может указывать на обновлённый образ. Для строгой повторяемости фиксируют проверенный digest `image@sha256:…` и обновляют его осознанно. Не вставляй выдуманный digest. Даже patch-тег — имя, а не криптографическая гарантия неизменности.

Образ должен содержать необходимые shell и инструменты. В slim/alpine может не быть `git`, `curl`, компилятора или Bash. Alpine использует musl, поэтому размер меньше не означает совместимость с любым нативным модулем. Выбирай образ после проверки зависимостей проекта.

## Services для интеграционных проверок

Service — соседний контейнер, доступный job по сети. Это не установка PostgreSQL внутрь контейнера Node/Python и не production-база.

```text
Job: Python / Node / psql
  ├── postgres:5432 → временный PostgreSQL
  └── redis:6379    → временный Redis
```

Для GitLab Docker executor имя/alias сервиса используется как DNS-имя. `localhost` в job обычно указывает на сам контейнер job, а не на PostgreSQL. Сетевые детали других executors могут отличаться. Service не устанавливает клиентские библиотеки в job; Python-драйвер PostgreSQL нужно установить отдельно. [GitLab services](https://docs.gitlab.com/ci/services/).

Ниже самостоятельный `.gitlab-ci.yml` для проверки временной PostgreSQL. Это дополнение к примеру курса: вместо отсутствующих `requirements.txt` и тестов используются клиент `psql` из того же официального образа и явная проверка SQL.

```yaml
workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "push" && $CI_COMMIT_BRANCH'
    - when: never

database_check:
  image: postgres:15
  services:
    - name: postgres:15
      alias: postgres
  variables:
    POSTGRES_DB: ci_test
    POSTGRES_USER: ci_test
    POSTGRES_PASSWORD: disposable-ci-only
    PGHOST: postgres
    PGDATABASE: ci_test
    PGUSER: ci_test
  script:
    - |
      for attempt in $(seq 1 30); do
        if pg_isready -h "$PGHOST" -U "$PGUSER" -d "$PGDATABASE"; then
          break
        fi
        sleep 1
      done
      export PGPASSWORD="$POSTGRES_PASSWORD"
      result=$(psql -v ON_ERROR_STOP=1 -Atc 'SELECT 1 + 1')
      test "$result" = 2
```

**Условия:** runner с Docker executor и поддержкой services, доступ к образу PostgreSQL. Запуск — push ветки; tags, MR и расписание этим `workflow` не разрешены. Создаются контейнер job и service, команда ожидает готовность, затем выполняет запрос. Ожидается успешная job; при недоступной БД или неверных credentials проверка завершается ошибкой. Это smoke-тест подключения, не полноценные интеграционные тесты приложения.

`disposable-ci-only` — намеренно открытый пароль одноразовой тестовой БД без внешних данных. Его нельзя переносить на постоянные или доступные извне БД. Порт сервиса не нужно публиковать на хост. Готовность TCP-порта не означает, что применены миграции и загружены fixtures: эти действия при необходимости выполняются отдельно.

## Из собранного приложения в Docker-образ

Учебный [Dockerfile](examples/ci-demo/Dockerfile) использует результат `npm run build`:

```dockerfile
FROM node:22-bookworm-slim
WORKDIR /app
COPY --chown=node:node dist/app.cjs ./app.cjs
USER node
EXPOSE 3000
CMD ["node", "app.cjs"]
```

Он не пересобирает исходники: копирует уже подготовленный файл `dist/app.cjs`. Runtime запускается от `node`, не root. `EXPOSE` документирует порт, но не публикует его. `.dockerignore` разрешает передавать только нужный файл из `dist/`, исключая `.env`, `.git` и другие материалы из контекста.

Для этого приложения нет сторонних runtime-зависимостей. У настоящего npm-проекта с зависимостями копирования одного файла недостаточно: нужен bundle либо установка production-зависимостей по lock-файлу. Multi-stage build отделяет компиляторы и тестовые инструменты от финального runtime; подробности уже есть в Docker-конспекте. [Рекомендации Docker по сборке](https://docs.docker.com/build/building/best-practices/).

Команды из корня **учебного проекта**, при работающем локальном Docker:

```bash
npm ci --no-audit --no-fund
npm test
npm run build
docker build -t ci-demo:local .
docker run --rm -d --name ci-demo-local -p 127.0.0.1:8080:3000 \
  -e APP_ENV=development ci-demo:local
curl --fail http://127.0.0.1:8080/health
curl --fail http://127.0.0.1:8080/
docker logs ci-demo-local
docker stop ci-demo-local
```

Ожидается JSON `{"status":"ok"}` на `/health` и сообщение с `environment: development` на `/`. Первому запросу может понадобиться повтор после запуска процесса. Порт 8080 и имя контейнера должны быть свободны. `--rm` удаляет именно этот контейнер после остановки, не пользовательские volumes. Если сборка не была выполнена, `COPY dist/app.cjs` завершится ошибкой — это полезный сигнал о нарушенном порядке.

## Registry, теги и публикация

Registry хранит версии образов. В курсе названы Docker Hub, GitLab Container Registry, Harbor, Nexus, Quay.io и реестры облачных провайдеров. Конкретные доступность и тарифы проверяются отдельно.

Полное имя обычно выглядит как `registry/namespace/image:tag`. `myapp:abc` и `registry.example.com/myapp:abc` — разные ссылки: нельзя собрать первое и ожидать, что push второго автоматически найдёт этот образ.

Фрагмент `script` для GitLab job на **выделенном Shell runner с Docker CLI и daemon**; переменные `CI_REGISTRY*` предоставляет GitLab при включённом Container Registry:

```bash
export DOCKER_CONFIG
DOCKER_CONFIG=$(mktemp -d)
trap 'docker logout "$CI_REGISTRY" >/dev/null 2>&1' EXIT
printf '%s' "$CI_REGISTRY_PASSWORD" | docker login "$CI_REGISTRY" \
  --username "$CI_REGISTRY_USER" --password-stdin
image_ref="$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA-$CI_PIPELINE_ID"
docker build -t "$image_ref" .
docker push "$image_ref"
```

Запуск и порядок этого фрагмента определяет job; полный пример с rules есть в [деплое](Деплой%20и%20безопасность.md). Перед ним нужно скачать `dist/` из build-job. Пароль не передаётся аргументом `-p` и не выводится в лог. Отдельный `DOCKER_CONFIG` отделяет авторизацию job от постоянной конфигурации runner; `logout` удаляет credentials из этого конфигурационного файла. [Docker login](https://docs.docker.com/reference/cli/docker/login/).

SHA связывает образ с исходниками, pipeline ID различает повторные сборки одного commit. Для релиза сохраняют digest опубликованного образа и запрещают перезапись release-тегов; для production предпочтительна доставка по digest. Registry retention должен сохранять используемые версии и версии для rollback.

## Почему Docker-in-Docker не универсальное решение

| Вариант | Где daemon | Особенности |
|---|---|---|
| Shell runner с Docker | На выделенном хосте runner | Простой учебный вариант; доступ к Docker даёт большие права на хосте |
| Docker socket mount | На хосте, socket доступен job | Job фактически управляет хостовым daemon; контейнерная граница сильно ослаблена |
| Docker-in-Docker (`docker:dind`) | В service-контейнере | Нужны согласованные клиент/daemon, TLS/сертификаты, сеть и разрешённый режим runner |

Обычной job с `image: node:…` Docker-in-Docker не нужен. Он появляется, когда job сама должна обращаться к Docker daemon. Ошибка `Cannot connect to Docker daemon` может означать неверный endpoint, права на socket, TLS или неработающий daemon. Добавление одной строки `services: [docker:dind]` не исправляет все эти случаи. [Варианты Docker build в GitLab](https://docs.gitlab.com/ci/docker/using_docker_build/).

Контейнер не является абсолютной песочницей для чужого кода. Не давай недоверенным MR доступ к Docker socket, privileged runner, production-ключам и внутренней сети. Build runners следует отделять от production-хостов.

## Оптимизация и диагностика

Сначала измерь время pull, установки зависимостей и build. Затем уменьши build context через `.dockerignore`, используй подходящий базовый образ, устойчивые слои и при необходимости registry mirror. Docker layer cache, кэш скачанных образов и GitLab `cache` — разные механизмы.

Наличие образа на диске не означает, что runner никогда не обращается к registry: поведение зависит от pull policy. Фиксированные теги/digests помогают контролировать изменения, но требуют регулярного обновления ради исправлений безопасности.

| Симптом | Что проверить |
|---|---|
| `manifest unknown` | Имя, tag/digest, факт push и права чтения |
| `unauthorized` / `denied` | Registry endpoint, scope токена, namespace |
| `exec format error` | Архитектуру образа и хоста |
| `command not found` | Содержимое выбранного образа и доступный shell |
| PostgreSQL connection refused | Alias, порт, готовность service; не использован ли `localhost` |
| `No space left on device` | `df -h`, `docker system df`, политика хранения слоёв/artifacts |
| Контейнер сразу завершился | Exit code, `docker logs`, команда запуска и конфигурация |

`docker system prune` из курса — операция удаления, а не диагностика. На общем runner сначала выясни, что используется другими jobs; очистку выполняют по согласованной политике. Не добавляй `--volumes` в общий «рецепт починки».

## Что важно запомнить

Среда job, service и образ приложения выполняют разные роли. Container registry хранит результат сборки; CI artifacts и cache не заменяют его. Один проверенный образ продвигается через окружения с разной runtime-конфигурацией. Изоляция зависит от прав и настройки хоста.

## Проверка понимания

**Зачем `image: node:…`, если собираем свой Docker-образ?** Это среда выполнения команд job. Образ приложения — отдельный результат `docker build`.

**Почему PostgreSQL в service не находится по localhost?** Для Docker executor это другой контейнер; нужен service alias, например `postgres`.

**Точный patch-тег гарантирует неизменный образ?** Нет. Теги могут перемещаться; содержимое идентифицируется digest.

**Почему после успешного push deploy может получить unauthorized?** Авторизация публикации в CI и чтения на сервере — разные credentials и права.

**Почему нельзя лечить любой Docker-сбой включением privileged?** Это расширяет права, но не исправляет ошибочное имя, TLS, сеть или архитектуру; одновременно увеличивает риск для хоста.
