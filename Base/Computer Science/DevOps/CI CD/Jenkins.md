---
tags:
  - devops
  - ci-cd
  - jenkins
---

# Jenkins

[Навигация](README.md) · Источник: [оригинал](../CI%20CD%20курс%20ориг.md), разделы «Первый Jenkinsfile», «Использование образов в Jenkins» и «Секреты в CI/CD».

## Controller, agent, executor и workspace

**Jenkins** — сервер автоматизации, в котором Pipeline описывает сборку, проверки и доставку. Интеграции с Git, контейнерами и внешними системами во многом предоставляют плагины.

**Controller** хранит конфигурацию, управляет очередью и координирует pipeline. **Agent** выполняет назначенную работу на подключённом узле. Связь может устанавливаться через SSH либо входящее подключение agent, в том числе через WebSocket; конкретный способ настраивается администратором. Controller выбирает узел по label и доступности, резервирует исполнительный слот и передаёт действия agent через канал связи.

**Executor в Jenkins — слот выполнения на узле**, а не механизм изоляции GitLab Runner. Два executor на одном agent могут одновременно выполнять две задачи, деля CPU, RAM и другие ресурсы. Контейнеризация и виртуализация настраиваются отдельно. При недоступном agent или занятых слотах выполнение ждёт в очереди. [Архитектура agents](https://www.jenkins.io/doc/book/using/using-agents/).

**Workspace** — рабочий каталог checkout и сборки на agent. Он может переживать несколько запусков. Один верхнеуровневый `agent` обычно позволяет stages работать в одном workspace. При смене agent каталог и файлы могут быть другими; нельзя предполагать, что `dist/` появится там автоматически. Не запускай недоверенные сборки на controller или рядом с production-секретами.

## Jenkinsfile и два синтаксиса

`Jenkinsfile` хранится с исходниками. Это Groovy-based DSL Jenkins, не YAML и не произвольная обычная Groovy-программа: шаги и выполнение зависят от Jenkins и плагинов.

| Declarative Pipeline | Scripted Pipeline |
|---|---|
| `pipeline { agent … stages { stage { steps { … } } } }` | Обычно `node { stage('…') { … } }` |
| Проверяемая структура, директивы `when`, `post`, `options` | Управление через код и шаги Pipeline |
| Удобный старт; основной синтаксис курса | Полезен для сложной программной логики |

Scripted не является «следующим уровнем, на который все обязаны перейти». Используй достаточную выразительность без ненужных циклов и обёрток. [Синтаксис Jenkins Pipeline](https://www.jenkins.io/doc/book/pipeline/syntax/).

## Рабочий Jenkinsfile для того же приложения

Полный файл: [Jenkinsfile](examples/ci-demo/Jenkinsfile). Предпосылки: Linux agent с label `docker`, установленный Docker и доступ к daemon; плагины Pipeline, Declarative Pipeline, Git и Docker Pipeline с совместимыми версиями. Для SCM-задачи также нужен доступ на чтение репозитория. Label — договорённость с администратором, а не встроенное имя.

```groovy
pipeline {
    agent {
        docker {
            image 'node:22-bookworm-slim'
            label 'docker'
        }
    }
    options {
        timeout(time: 10, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    environment {
        APP_ENV = 'test'
        NPM_CONFIG_CACHE = "${WORKSPACE}/.npm"
    }
    stages {
        stage('Test') {
            steps {
                sh 'npm ci --no-audit --no-fund'
                sh 'npm run check'
                sh 'npm test'
            }
        }
        stage('Build') {
            steps {
                sh 'npm run build'
                sh 'node --check dist/app.cjs'
                sh 'tar -czf app.tar.gz -C dist app.cjs'
                archiveArtifacts artifacts: 'app.tar.gz', fingerprint: true
            }
        }
    }
    post {
        success {
            echo 'Tests passed; application artifact archived.'
        }
        failure {
            echo 'Build failed; inspect the first failing step.'
        }
    }
}
```

**Что запускает pipeline:** создание файла само по себе не подключает Jenkins к Git. Создай Pipeline job с **Pipeline script from SCM**, Git URL и путём `Jenkinsfile`, затем выполни **Build Now**. Для реакции на push настрой webhook и соответствующий Git/branch-source plugin; альтернативы — polling или расписание ниже. Declarative Pipeline из SCM по умолчанию выполняет checkout на выделенном agent, если его не отключить.

**Порядок:** выделение Docker-capable agent → контейнер Node с workspace → Test → Build → архивирование → `post`. При ошибке обязательного `sh` обычный следующий stage не выполняется. Результат успешного запуска — архив `app.tar.gz` в Jenkins; сервис ещё не развёрнут. В отличие от GitLab-примера syntax и HTTP-тест здесь выполняются последовательно.

`archiveArtifacts` сохраняет результат сборки, а fingerprint помогает отслеживать его использование. Срок хранения определяется политиками Jenkins/build discarder; настрой хранение build records и artifacts так, чтобы ручной релиз и rollback не потеряли нужную версию. `post` в примере только пишет статус, не отправляет уведомления и не делает откат. [Работа с Jenkinsfile и artifacts](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/).

## Основные директивы курса

| Директива | Как применяется | Ошибка, которую стоит предотвратить |
|---|---|---|
| `agent any` | Любой подходящий доступный узел | «Any» не устанавливает Node/Docker и не означает любую ОС |
| `agent none` | Не выделять общий agent, назначать его stages | Для `sh` и работы с workspace нужен подходящий node context |
| `agent { label 'linux' }` | Выбор по label | Несуществующий label оставляет pipeline в ожидании |
| `agent { docker { image '…' } }` | Контейнер на Docker-capable agent | Нужны плагин, daemon и права |
| `environment` | Переменные на pipeline или stage | `NODE_ENV=production` до установки может исключить devDependencies, нужные тестам |
| `options` | Timeout, concurrency, retries и другие политики | Повтор всего pipeline может повторить уже выполненный deploy/миграцию |
| `parameters` | Контролируемые входные данные ручного запуска | Подстановка свободного текста в `sh` без проверки может привести к выполнению команд |
| `triggers` | Cron или SCM polling | Не путать периодический build и поиск новых изменений |
| `post` | `always`, `success`, `failure`, cleanup и другие условия | Само наличие `failure` не откатывает приложение |

`timestamps()` из курса требует соответствующего плагина Timestamper. Стандартные имена вроде `BUILD_NUMBER`/`BUILD_ID` лучше не переопределять ради пользовательского номера релиза.

### События запуска: два варианта

Следующие блоки вставляются **внутрь `pipeline`** рабочего Jenkinsfile. Для простоты выбери один из них:

```groovy
triggers {
    pollSCM('H/5 * * * *')
}
```

Jenkins периодически проверяет SCM и запускает build при обнаружении изменений. `H` — стабильное распределение времени по хешу имени job, чтобы множество jobs не опрашивали Git одновременно. Это не точный cron «каждые пять минут, начиная с нуля».

```groovy
triggers {
    cron('H 4 * * 1-5')
}
```

Build запускается по будням в выбранную Jenkins минуту часа 04 в применимой временной зоне, даже если исходники не изменились. При изменении Jenkinsfile триггеры должны быть прочитаны Jenkins при выполнении/настройке job; не рассчитывай, что сервер узнает о новом файле сам без подключения SCM.

Для Multibranch Pipeline Jenkins обнаруживает подходящие ветки с Jenkinsfile и создаёт задачи для них. Discovery и webhook настраиваются в конкретном branch-source plugin. Условие `when { branch 'main' }` рассчитано на Multibranch, а не на любой Pipeline job.

### Параметры и ручное подтверждение

Пример безопасного **фрагмента** внутри `pipeline`:

```groovy
parameters {
    choice(name: 'TARGET_ENV', choices: ['staging', 'production'],
           description: 'Целевое окружение')
}
```

Он добавляет параметр, но сам не выполняет deploy. Для production-stage существует шаг `input` или директива stage `input`; круг подтверждающих пользователей и timeout нужно задавать осознанно. Параметр `RUN_TESTS=false` из учебного обзора не должен позволять обойти обязательные релизные проверки.

## Docker agents и обмен файлами

Docker Pipeline монтирует workspace в контейнер, поэтому команды видят checkout. Если у stage свой Docker agent, `reuseNode true` позволяет использовать текущий узел и workspace. Это не передача файлов между произвольными серверами. Пользователь контейнера должен иметь права на каталог; после root-контейнеров могут остаться недоступные агенту файлы. [Docker в Jenkins Pipeline](https://www.jenkins.io/doc/book/pipeline/docker/).

**Дополнительно:** для небольших файлов между agents одного запуска применяются `stash`/`unstash`. Для сохраняемых результатов — `archiveArtifacts` и подходящее artifact-хранилище. Для большого Docker-образа — registry. Остатки workspace нельзя считать гарантированным артефактом или надёжным кэшем.

## Credentials и секреты

В Jenkins сохраняют credential в хранилище с ID, а pipeline обращается к ID. Сам пароль в Jenkinsfile не записывают. Тип credential важен: secret text даёт одно значение; username/password дополнительно предоставляет компоненты имени и пароля.

Фрагмент **внутри stage**, где действительно нужен секрет:

```groovy
environment {
    API_TOKEN = credentials('demo-api-token')
}
```

Credential с этим ID должен быть создан администратором и доступен job. Для его использования shell-командой предпочитай одинарные/triple-single Groovy-строки: переменную раскроет shell, а не Groovy-интерполяция в controller. Не выводи `env`, `printenv` или `echo "$API_TOKEN"`. Маскирование логов — вспомогательная защита; pipeline-код с доступом к секрету способен его отправить наружу. Ограничивай scope credentials и доверие к SCM-коду. [Jenkins: безопасная работа с credentials](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/#handling-credentials).

## Проверка и типичные сбои

| Симптом | Проверка |
|---|---|
| `Waiting for next available executor` | Online agent, label, занятые executor slots |
| `No such DSL method` | Плагин, версия, правильный шаг и его контекст |
| `npm: not found` | Какой agent/container реально выполняет `sh` |
| Docker permission denied | Доступ пользователя agent к daemon; не раздавать права всем подряд |
| `dist/` пропал в следующем stage | Сменился agent/workspace; настроить передачу |
| Build прошёл локально, упал в Jenkins | Commit, версия Node, environment, права, сеть и оставшиеся файлы |

Для проверки Declarative Pipeline используй Jenkins Declarative Linter на целевом сервере, а для шагов — Snippet Generator. Обычный Groovy-парсер не проверяет весь DSL и доступность plugins. [Инструменты разработки Pipeline](https://www.jenkins.io/doc/book/pipeline/development/).

В курсе показан Blue Ocean. По [текущему статусу Jenkins](https://www.jenkins.io/doc/book/blueocean/) с июля 2026 он deprecated и больше не получает security fixes; для нового стенда документация указывает Pipeline Graph View / Pipeline: Stage View. Это актуализация интерфейса, не замена Jenkins другой CI-платформой.

## Что важно запомнить

Controller координирует, agent исполняет, executor задаёт доступный слот. Jenkinsfile хранит процесс в Git, но Jenkins job и доступ к SCM надо настроить. Workspace, stash и архив artifacts решают разные задачи. Выбор container agent не отменяет обслуживание Docker-хоста и плагинов.

## Проверка понимания

**Почему GitLab job и Jenkins job не равнозначны?** GitLab job — отдельная задача pipeline; Jenkins Pipeline job обычно описывает весь процесс со stages и steps.

**`agent any` гарантирует чистую контейнерную среду?** Нет, только выбор узла. Изоляция определяется его настройками и используемым agent.

**Почему retry всего pipeline опасен для доставки?** Он может повторить побочные эффекты — миграцию, публикацию, внешнее действие. Повторяемость нужно обеспечивать отдельно.

**Почему архивирование не равно deploy?** Архив сохранён в Jenkins, но ещё не доставлен и не запущен в целевом окружении.

**Можно ли доверять маскированию credentials, выполняя любой Jenkinsfile?** Нет. Доступ к секрету получает выполняемый код, поэтому важны review, scope и ограничения для недоверенных веток.
