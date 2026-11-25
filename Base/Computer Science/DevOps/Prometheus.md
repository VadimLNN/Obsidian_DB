---
tags:
  - devops/monitoring/prometheus
status: todo
---

# Prometheus

Prometheus собирает временные ряды метрик по pull-модели и позволяет запрашивать их через PromQL.

## Что нужно знать

- pull-модель и scrape targets;
- типы метрик: Counter, Gauge, Histogram, Summary;
- labels и cardinality;
- PromQL;
- exporters, включая Node Exporter;
- recording и alerting rules;
- Alertmanager;
- service discovery.

## Практика

- [ ] Собрать метрики хоста через Node Exporter.
- [ ] Написать несколько PromQL-запросов.
- [ ] Настроить alert и маршрут в Alertmanager.
- [ ] Построить дашборд в Grafana.

## Подходы к наблюдаемости

- Golden Signals;
- USE;
- RED.

## Связанные темы

- [[Grafana]]
- [[Logging]]
- [[Kubernetes]]

