---
tags:
  - linux/administration
---

# systemd и сервисы

[[OS/Linux/Linux|Карта Linux]]

## systemd

systemd — основной менеджер системы и сервисов во многих Linux-дистрибутивах.

Roadmap называет его:

> главным процессом, запускающим всё остальное.

DevOps должен уметь написать простой unit-файл.

### Основные команды

Проверить сервис:

```bash
systemctl status nginx
```

Запустить:

```bash
systemctl start nginx
```

Остановить:

```bash
systemctl stop nginx
```

Перезапустить:

```bash
systemctl restart nginx
```

Добавить в автозапуск:

```bash
systemctl enable nginx
```

## Пример systemd unit

Roadmap рекомендует взять простой HTTP-сервис и оформить его как systemd-службу.

Пример структуры:

```ini
[Unit]
Description=My HTTP Service
After=network.target

[Service]
ExecStart=/usr/bin/python /opt/app/server.py
Restart=always

[Install]
WantedBy=multi-user.target
```

Идея:

```text
Linux запускается
      ↓
systemd
      ↓
запускает сервис
      ↓
если сервис падает
      ↓
Restart=always
      ↓
сервис поднимается снова
```

## Практика: свой systemd-сервис

Roadmap предлагает:

> взять любой простой HTTP-сервис и оформить его как systemd unit.

Требования:

- автозапуск;
- рестарт при падении;
- лимиты памяти через [[Cgroups и Namespaces|cgroups]];
- после `kill -9` сервис должен подняться снова.

### Проверка

Например:

```bash
systemctl status myapp
```

Получить PID:

```bash
ps aux | grep myapp
```

Убить:

```bash
kill -9 <PID>
```

Проверить:

```bash
systemctl status myapp
```

Если unit настроен правильно — сервис должен снова работать.

## systemd и управление сервисами

### Нужно понимать

- что такое init system;
- PID 1;
- service/unit;
- target;
- dependency;
- restart policy;
- journal.

### Команды

```bash
systemctl status SERVICE
systemctl start SERVICE
systemctl stop SERVICE
systemctl restart SERVICE
systemctl reload SERVICE
systemctl enable SERVICE
systemctl disable SERVICE
systemctl daemon-reload
systemctl list-units
journalctl
journalctl -u SERVICE
journalctl -f
```

### Практика — свой systemd unit

Создай простейший HTTP-сервер.

Например:

```bash
python -m http.server 8080
```

Оформи его как systemd service.

Пример структуры unit-файла:

```ini
[Unit]
Description=DevOps Linux Lab HTTP Server
After=network.target

[Service]
Type=simple
User=YOUR_USER
WorkingDirectory=/home/YOUR_USER/linux-lab
ExecStart=/usr/bin/python -m http.server 8080
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### Проверка

- [ ] Сервис запускается через `systemctl start`.
- [ ] Логи видны через `journalctl -u`.
- [ ] Сервис включён в автозагрузку.
- [ ] После `kill -9 <PID>` systemd поднимает процесс заново.
- [ ] После reboot сервис стартует автоматически.

#### Дополнительно

Разобраться:

- `Restart=on-failure` vs `Restart=always`;
- `After=`;
- `Requires=`;
- `Wants=`;
- systemd timers.

## Связанные темы

- [[OS/Linux/Administration/Процессы и сигналы]]
- [[OS/Linux/Administration/Логи и journalctl]]
- [[OS/Linux/Administration/Диагностика проблем]]
