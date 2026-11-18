# SSH и передача файлов

[[OS/Linux/Linux|Карта Linux]]

## SSH

SSH — обязательный инструмент DevOps.

Roadmap рекомендует изучить:

```text
ssh-keygen
ssh-copy-id
~/.ssh/config
port forwarding
scp
rsync
```

### Создание ключа

```bash
ssh-keygen
```

### Копирование ключа

```bash
ssh-copy-id user@server
```

### Подключение

```bash
ssh user@server
```

### SSH config

Файл:

```text
~/.ssh/config
```

Позволяет создавать короткие конфигурации подключения.

Пример:

```text
Host myserver
    HostName 192.168.1.10
    User devops
```

После этого:

```bash
ssh myserver
```

### Навыки и практика

SSH — обязательный навык DevOps.

#### Нужно понимать

- client/server;
- SSH key pair;
- private key;
- public key;
- `authorized_keys`;
- host key;
- SSH agent;
- port forwarding.

#### Команды

```bash
ssh
ssh-keygen
ssh-copy-id
scp
rsync
```

Конфиг:

```text
~/.ssh/config
```

Пример:

```sshconfig
Host dev-server
    HostName 192.168.1.50
    User devops
    IdentityFile ~/.ssh/id_ed25519
```

Теперь:

```bash
ssh dev-server
```

#### Практика

Лучше использовать отдельную VM.

- [ ] Создать SSH key pair.
- [ ] Настроить вход по ключу.
- [ ] Отключить password authentication на тестовом сервере.
- [ ] Отключить root login.
- [ ] Передать файл через `scp`.
- [ ] Синхронизировать директорию через `rsync`.
- [ ] Настроить `~/.ssh/config`.

##### Port forwarding

Разобраться минимум с local forwarding:

```bash
ssh -L 8080:localhost:80 user@server
```

## `scp`

Копирование файлов через SSH.

```bash
scp file.txt user@server:/tmp/
```

## `rsync`

Синхронизация файлов.

```bash
rsync -av ./project/ user@server:/opt/project/
```

## Связанные темы

- [[OS/Linux/Administration/Сеть Linux]]
- [[OS/Linux/Administration/Безопасность сервера]]
