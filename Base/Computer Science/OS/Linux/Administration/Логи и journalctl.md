# Логи и journalctl

[[OS/Linux/Linux|Карта Linux]]

## journalctl

systemd хранит логи сервисов в journal.

```bash
journalctl
```

Для конкретного сервиса:

```bash
journalctl -u nginx
```

Roadmap отдельно рекомендует изучить:

- фильтрацию журналов;
- systemd dependencies;
- systemd timers.

## Логи

Нужно знать:

```text
journalctl
syslog
/var/log/*
logrotate
```

### `/var/log`

Один из основных каталогов логов Linux.

```bash
ls /var/log
```

### `logrotate`

Используется для ротации логов.

Идея:

```text
app.log
app.log.1
app.log.2
...
```

Это не позволяет логам бесконечно расти.

### Навыки и практика

#### Нужно знать

```text
/var/log/
journalctl
syslog
logrotate
```

В современных systemd-системах значительная часть логов доступна через journal.

#### Команды

```bash
journalctl
journalctl -b
journalctl -p err
journalctl -u sshd
journalctl --since "10 minutes ago"
journalctl -f
```

#### Практика

- [ ] Найти логи текущей загрузки системы.
- [ ] Найти сообщения уровня error.
- [ ] Посмотреть live logs своего systemd-сервиса.
- [ ] Намеренно сломать `ExecStart`, увидеть ошибку и диагностировать её по журналу.

## Связанные темы

- [[OS/Linux/Terminal/Поиск и обработка текста]]
- [[OS/Linux/Administration/systemd и сервисы]]
