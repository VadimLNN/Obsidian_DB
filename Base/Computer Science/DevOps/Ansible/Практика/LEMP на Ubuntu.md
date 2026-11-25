---
tags:
  - devops/ansible
  - web/backend
  - databases
---

# LEMP на Ubuntu

[[Base/Computer Science/DevOps/Ansible/Ansible|Карта тем]] · [[Base/Computer Science/DevOps/Ansible/Практика/Развёртывание приложений|Развёртывание приложений]] · [[Base/Computer Science/DevOps/Ansible/Roles/Роли|Роли]] · [[Base/Computer Science/DevOps/Ansible/Безопасность/Ansible Vault|Ansible Vault]] · [[Base/Computer Science/DevOps/Ansible/Playbooks/Handlers|Handlers]]

**Цель:** одним playbook на Ubuntu 24.04 LTS настроить MariaDB, PHP-FPM, [[Nginx]] и маленькое PHP-приложение, проверяющее соединение с БД. Этот пример специально упрощён: он показывает **архитектуру автоматизации**, а не является готовым production-шаблоном Laravel.

**Предусловия:** пустая Ubuntu 24.04 LTS, SSH-доступ пользователем `deployer` с sudo, Python 3, разрешённые сетевые подключения. В примере [[SSH и передача файлов|SSH]] работает на порту 22. Изменить имена, IP и параметры для собственного стенда.

## Структура проекта

```text
lemp-deploy/
├── ansible.cfg
├── inventory.yml
├── site.yml
├── vars.yml
├── secrets.yml                 # создаётся через ansible-vault
└── roles/
    ├── database/tasks/main.yml
    ├── php/tasks/main.yml
    ├── nginx/tasks/main.yml
    ├── nginx/handlers/main.yml
    ├── nginx/templates/demo.conf.j2
    ├── application/tasks/main.yml
    ├── application/templates/config.json.j2
    ├── application/templates/index.php.j2
    └── firewall/tasks/main.yml
```

## Конфигурация и inventory

`ansible.cfg`:

```ini
[defaults]
inventory = ./inventory.yml
roles_path = ./roles
host_key_checking = True
```

`inventory.yml`:

```yaml
---
all:
  children:
    lemp:
      hosts:
        app01:
          ansible_host: 192.0.2.10
          ansible_user: deployer
```

`vars.yml`:

```yaml
---
app_db_name: demoapp
app_db_user: demoapp
app_directory: /var/www/demo/public
php_fpm_socket: /run/php/php8.3-fpm.sock
ssh_port: 22
```

`php_fpm_socket` соответствует типичному PHP 8.3 из Ubuntu 24.04; в другой ОС или версии PHP проверь фактический сокет: `ls /run/php/`.

`secrets.yml` — **создать зашифрованным**, а не записывать пароль открытым текстом в Git:

```bash
ansible-vault create secrets.yml
```

Его содержимое **до шифрования**, вводимое в редакторе Vault:

```yaml
---
app_db_password: REPLACE_WITH_LONG_RANDOM_SECRET
```

## Главный playbook

`site.yml`:

```yaml
---
- name: Deploy demo LEMP application
  hosts: lemp
  become: true
  gather_facts: true
  vars_files:
    - vars.yml
    - secrets.yml
  roles:
    - database
    - php
    - nginx
    - application
    - firewall
```

Порядок ролей здесь осознанный: сначала БД и PHP-FPM, затем веб-сервер, файлы приложения и [[Firewall|firewall]].

## Роль `database`

`roles/database/tasks/main.yml`:

```yaml
---
- name: Install MariaDB and Python driver
  ansible.builtin.apt:
    name:
      - mariadb-server
      - python3-pymysql
    state: present
    update_cache: true

- name: Ensure MariaDB is enabled and started
  ansible.builtin.service:
    name: mariadb
    enabled: true
    state: started

- name: Create application database
  ansible.mariadb.mariadb_db:
    name: "{{ app_db_name }}"
    state: present
    login_unix_socket: /run/mysqld/mysqld.sock

- name: Create database user with limited permissions
  ansible.mariadb.mariadb_user:
    name: "{{ app_db_user }}"
    password: "{{ app_db_password }}"
    host: 127.0.0.1
    priv: "{{ app_db_name }}.*:SELECT,INSERT,UPDATE,DELETE"
    state: present
    login_unix_socket: /run/mysqld/mysqld.sock
  no_log: true
```

Для MariaDB используется отдельная коллекция `ansible.mariadb`; установить её на **control node**. `ansible.mysql` предназначена для MySQL: её документация объявляет прекращение поддержки MariaDB с версии 6.0.0. Параметры приведены по документации [mariadb_db](https://docs.ansible.com/projects/ansible/latest/collections/ansible/mariadb/mariadb_db_module.html) и [mariadb_user](https://docs.ansible.com/projects/ansible/latest/collections/ansible/mariadb/mariadb_user_module.html); [объявление ansible.mysql](https://docs.ansible.com/projects/ansible/latest/collections/ansible/mysql/mysql_user_module.html).

```bash
ansible-galaxy collection install ansible.mariadb
```

Роль MariaDB использует локальную unix-сокет-аутентификацию администратора; она подходит для типичного свежего Ubuntu-стенда при выполнении задач с `become: true`. Для уже настроенной БД или другого дистрибутива параметры подключения могут отличаться.

## Роль `php`

`roles/php/tasks/main.yml`:

```yaml
---
- name: Install PHP-FPM and MariaDB driver
  ansible.builtin.apt:
    name:
      - php-fpm
      - php-mysql
    state: present

- name: Ensure PHP-FPM is enabled and running
  ansible.builtin.service:
    name: php8.3-fpm
    enabled: true
    state: started
```

Для Ubuntu 24.04 обычно используется PHP-FPM 8.3; на другой версии Ubuntu адаптировать имя systemd-unit и путь к сокету.

## Роль `nginx`

`roles/nginx/tasks/main.yml`:

```yaml
---
- name: Install Nginx
  ansible.builtin.apt:
    name: nginx
    state: present

- name: Disable default Nginx site
  ansible.builtin.file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  notify: Reload Nginx

- name: Deploy application virtual host
  ansible.builtin.template:
    src: demo.conf.j2
    dest: /etc/nginx/sites-available/demo.conf
    owner: root
    group: root
    mode: '0644'
  notify: Reload Nginx

- name: Enable application virtual host
  ansible.builtin.file:
    src: /etc/nginx/sites-available/demo.conf
    dest: /etc/nginx/sites-enabled/demo.conf
    state: link
  notify: Reload Nginx

- name: Validate Nginx configuration
  ansible.builtin.command:
    cmd: nginx -t
  changed_when: false

- name: Ensure Nginx is enabled and running
  ansible.builtin.service:
    name: nginx
    enabled: true
    state: started
```

`roles/nginx/templates/demo.conf.j2`:

```nginx
server {
    listen 80 default_server;
    server_name _;
    root {{ app_directory }};
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:{{ php_fpm_socket }};
    }

    location ~ /\. {
        deny all;
    }
}
```

`roles/nginx/handlers/main.yml`:

```yaml
---
- name: Reload Nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded
```

Роль проверяет конфигурацию через `nginx -t` перед запуском службы и отложенным reload. Ошибка проверки останавливает задачи на хосте; автоматического отката уже записанного файла этот пример не делает.

## Роль `application`

`roles/application/tasks/main.yml`:

```yaml
---
- name: Create application document root
  ansible.builtin.file:
    path: "{{ app_directory }}"
    state: directory
    owner: root
    group: www-data
    mode: '0755'

- name: Create private application configuration directory
  ansible.builtin.file:
    path: /etc/demo-app
    state: directory
    owner: root
    group: www-data
    mode: '0750'

- name: Deploy private database configuration
  ansible.builtin.template:
    src: config.json.j2
    dest: /etc/demo-app/config.json
    owner: root
    group: www-data
    mode: '0640'
  no_log: true
  diff: false

- name: Deploy demo PHP application
  ansible.builtin.template:
    src: index.php.j2
    dest: "{{ app_directory }}/index.php"
    owner: root
    group: www-data
    mode: '0644'
```

`roles/application/templates/config.json.j2`:

```jinja2
{{ {
    'host': '127.0.0.1',
    'database': app_db_name,
    'username': app_db_user,
    'password': app_db_password
} | ansible.builtin.to_nice_json }}
```

Конфигурация сохраняется как JSON вне webroot и читается как данные: `$`, кавычки и обратные слеши в пароле не становятся [PHP-подстановками](https://www.php.net/manual/en/language.types.string.php#language.types.string.syntax.double). Сериализацию выполняет [to_nice_json](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/to_nice_json_filter.html), чтение — [json_decode](https://www.php.net/manual/en/function.json-decode.php).

`roles/application/templates/index.php.j2`:

```php
<?php
try {
    $configJson = file_get_contents('/etc/demo-app/config.json');
    if ($configJson === false) {
        throw new RuntimeException('Cannot read application configuration');
    }
    $config = json_decode($configJson, true, 512, JSON_THROW_ON_ERROR);
    $dsn = sprintf('mysql:host=%s;dbname=%s;charset=utf8mb4',
        $config['host'], $config['database']);
    $db = new PDO($dsn, $config['username'], $config['password'], [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    ]);
    $db->query('SELECT 1');
    header('Content-Type: text/plain; charset=UTF-8');
    echo "LEMP application OK; database connection OK\n";
} catch (Throwable $error) {
    http_response_code(503);
    header('Content-Type: text/plain; charset=UTF-8');
    echo "Application is not ready\n";
}
```

Пароль находится вне публичного webroot. Для production лучше подключать секреты через специализированные secret-management решения и защищённые переменные окружения / файловые монтирования. **Нельзя** оставлять необработанные секреты в HTML, логах или stdout задачи.

## Роль `firewall`

Ниже пример **только для Ubuntu с UFW**. В исходном курсе используется `firewalld`, что зависит от ОС. Задача firewall — оставить доступным SSH и открыть HTTP.

`roles/firewall/tasks/main.yml`:

```yaml
---
- name: Install UFW
  ansible.builtin.apt:
    name: ufw
    state: present

- name: Allow SSH before enabling firewall
  community.general.ufw:
    rule: allow
    port: "{{ ssh_port }}"
    proto: tcp

- name: Allow HTTP
  community.general.ufw:
    rule: allow
    port: '80'
    proto: tcp

- name: Enable UFW
  community.general.ufw:
    state: enabled
```

Зависимость:

```bash
ansible-galaxy collection install community.general
```

**Предупреждение:** если SSH использует другой порт, есть IPv6-особенности, отдельные правила доступа или облачный firewall, предварительно адаптировать политику. Ошибка в firewall может оборвать доступ к серверу. В production безопаснее проверять правила и доступ через независимый канал управления.

## Запуск и проверка

```bash
ansible-galaxy collection install ansible.mariadb community.general
ansible-inventory --graph
ansible lemp -m ansible.builtin.ping
ansible-playbook site.yml --syntax-check --ask-vault-pass
ansible-playbook site.yml --ask-vault-pass
```

После запуска:

```bash
curl -fsS http://192.0.2.10/
# Ожидаемый ответ:
# LEMP application OK; database connection OK
```

На целевой машине:

```bash
sudo systemctl status mariadb
sudo systemctl status nginx
sudo systemctl status php8.3-fpm
sudo nginx -t
sudo ss -ltnp | grep ':80'
```

Запусти playbook **повторно**: большинство задач должны показать `ok`, а не `changed`, если конфигурация не изменялась. Изменение Nginx-шаблона должно уведомить handler.

При использовании PHP-FPM отключите `display_errors` в рабочем окружении: ошибки чтения файлов и другие PHP warnings не должны попадать в HTTP-ответ. Ошибки соединения и разбора JSON в примере возвращают общее сообщение с HTTP 503.
