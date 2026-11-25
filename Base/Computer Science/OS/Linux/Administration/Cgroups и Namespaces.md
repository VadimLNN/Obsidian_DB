---
tags:
  - linux/administration
  - devops/docker
---

# Cgroups и Namespaces

[[OS/Linux/Linux|Карта Linux]]

## Cgroups

Cgroups — механизм Linux для контроля ресурсов группы процессов.

Можно ограничивать:

- CPU;
- RAM.

Roadmap подчёркивает связь:

```text
cgroups
   ↓
контейнеризация
   ↓
Docker / Kubernetes
```

## Namespaces

Namespaces обеспечивают изоляцию:

- процессов;
- сети;
- файловой системы.

В roadmap:

```text
cgroups + namespaces
```

названы базовыми механизмами контейнеризации.

## Связанные темы

- [[OS/Linux/Administration/Процессы и сигналы]]
- [[OS/Linux/Administration/Ресурсы и диагностика]]
- [[OS/Linux/Administration/systemd и сервисы]]
- [[Docker]]
- [[Requests, Limits и QoS]]
