---
tags:
  - devops/infrastructure-as-code/terraform
  - devops/cloud
status: todo
---

# Terraform

Terraform описывает инфраструктуру декларативно и создаёт её через API провайдеров.

## Что нужно знать

- Provider;
- Resource и Data Source;
- State и его защита;
- `init`, `plan`, `apply`, `destroy`;
- variables и outputs;
- Module;
- remote Backend и блокировки state;
- drift и импорт существующих ресурсов.

## Практика

- [ ] Создать сеть и виртуальную машину.
- [ ] Вынести повторяемую конфигурацию в module.
- [ ] Настроить remote backend.
- [ ] Просмотреть и объяснить plan перед apply.

## Собеседование

- [ ] Зачем Terraform хранит state?
- [ ] Чем Provider отличается от Backend?
- [ ] Как безопасно работать со state в команде?

## Связанные темы

- [[Clouds]]
- [[Ansible]]
- [[Git]]

