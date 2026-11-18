# Bash-скрипты

[[OS/Linux/Linux|Карта Linux]]

## Bash scripting

Roadmap рекомендует знать:

- условия;
- циклы;
- функции;
- exit codes.

### Простейший Bash-скрипт

```bash
#!/bin/bash

echo "Hello"
```

### Условия

```bash
if [ -f "/etc/nginx/nginx.conf" ]; then
    echo "config exists"
else
    echo "config not found"
fi
```

### Цикл

```bash
for file in *.log; do
    echo "$file"
done
```

### Функции

```bash
check_service() {
    systemctl status "$1"
}

check_service nginx
```

### Exit code

Каждая команда завершается кодом.

```text
0       успешно
не 0    ошибка
```

Посмотреть код предыдущей команды:

```bash
echo $?
```

### Навыки и практика

Не нужно становиться Bash-разработчиком. Нужно уметь автоматизировать рутину.

#### Освоить

- shebang;
- переменные;
- аргументы `$1`, `$2`, `$@`;
- `$?`;
- exit code;
- `if`;
- `case`;
- `for`;
- `while`;
- functions;
- command substitution `$(...)`;
- `set -euo pipefail`.

#### Минимальный шаблон

```bash
#!/usr/bin/env bash
set -euo pipefail

main() {
    echo "hello"
}

main "$@"
```

#### Практические скрипты

##### Скрипт 1 — health check

Проверяет URL:

```bash
./check_url.sh http://localhost:8080
```

Возвращает:

```text
OK 200
```

или ненулевой exit code при ошибке.

##### Скрипт 2 — disk warning

Проверяет заполнение `/` и выводит предупреждение при превышении, например, 80%.

##### Скрипт 3 — backup

Создаёт timestamped архив директории:

```text
backup-2026-09-03-2100.tar.gz
```

#### Чеклист

- [ ] Умею читать аргументы.
- [ ] Проверяю exit code команды.
- [ ] Использую функции.
- [ ] Обрабатываю ошибочные входные данные.
- [ ] Скрипты исполняемые через `chmod +x`.

## Связанные темы

- [[OS/Linux/Terminal/Bash и Fish]]
- [[OS/Linux/Terminal/Конвейеры и операторы]]
- [[OS/Linux/Практика Linux]]
