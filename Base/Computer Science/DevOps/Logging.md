---
tags:
  - devops/logging
  - devops/monitoring
status: todo
---

# Logging

Централизованное логирование собирает события с серверов и приложений, позволяет искать причины ошибок и связывать их с метриками.

## Источники

- log-файлы серверов;
- stdout/stderr контейнеров Kubernetes;
- системные журналы.

## Стеки

### ELK

- Elasticsearch;
- Logstash;
- Kibana.

### Loki

- Loki + Grafana;
- labels и LogQL.

## Collectors

- Fluentd;
- Filebeat;
- Fluent Bit.

## Практика

- [ ] Собрать stdout приложения.
- [ ] Добавить структурированные JSON-логи.
- [ ] Найти ошибку по request/correlation ID.
- [ ] Настроить хранение и ротацию.

## Связанные темы

- [[Grafana]]
- [[Prometheus]]
- [[Kubernetes]]

