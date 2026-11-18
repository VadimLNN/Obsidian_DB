# Nginx

[[OS/Linux/Linux|Карта Linux]]

## Nginx

Nginx нужен и как часть Linux-практики, и дальше для Docker/Kubernetes.

### Нужно понимать

- web server;
- reverse proxy;
- upstream;
- virtual host/server block;
- HTTP headers;
- access log / error log.

### Практика

Поднять приложение:

```text
client -> nginx :80 -> app :8080
```

#### Требования

- [ ] Nginx слушает 80 порт.
- [ ] Приложение слушает только localhost:8080.
- [ ] Nginx проксирует запросы в приложение.
- [ ] Есть access log.
- [ ] Есть error log.
- [ ] После перезапуска системы всё стартует автоматически.

## Связанные темы

- [[OS/Linux/Administration/Сеть Linux]]
- [[OS/Linux/Administration/systemd и сервисы]]
- [[OS/Linux/Практика Linux]]
