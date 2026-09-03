---
tags:
  - devops
  - rabbitmq
  - messaging
status: todo
---

# RabbitMQ

RabbitMQ — классический брокер сообщений: producer публикует сообщение, broker маршрутизирует его через exchange в queue, consumer подтверждает обработку.

## Что нужно знать

- producer, consumer и queue;
- exchange и routing key;
- acknowledgements;
- durability и persistence;
- retry и dead-letter queue;
- отличие очереди сообщений от Kafka log.

## Практика

- [ ] Поднять RabbitMQ локально.
- [ ] Настроить exchange и несколько queues.
- [ ] Проверить retry и dead-letter flow.

## Связанные темы

- [[Kafka]]
- [[Docker]]

