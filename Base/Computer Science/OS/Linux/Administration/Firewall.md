---
tags:
  - linux/networking
  - linux/security
---

# Firewall

[[OS/Linux/Linux|Карта Linux]]

## Firewall

Для старта достаточно понимать концепцию.

На разных системах можно встретить:

- nftables;
- iptables;
- ufw;
- firewalld.

### Нужно уметь

- разрешить [[SSH и передача файлов|SSH]];
- разрешить HTTP/HTTPS;
- закрыть ненужный порт;
- проверить результат с другой машины.

**Важно:** на удалённом сервере сначала разрешай SSH и только потом ужесточай firewall, иначе можно потерять доступ.

## Связанные темы

- [[OS/Linux/Administration/Сеть Linux]]
- [[OS/Linux/Administration/Безопасность сервера]]
