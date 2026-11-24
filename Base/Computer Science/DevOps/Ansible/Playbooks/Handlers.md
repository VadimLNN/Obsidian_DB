# Handlers

[[Base/Computer Science/DevOps/Ansible/Ansible|Карта тем]] · [[Base/Computer Science/DevOps/Ansible/Playbooks/Переменные и шаблоны|Переменные и шаблоны]] · [[Base/Computer Science/DevOps/Ansible/Практика/LEMP на Ubuntu|LEMP на Ubuntu]] · [[Base/Computer Science/DevOps/Ansible/Playbooks/Управление выполнением|Управление выполнением]]

**Handler** — задача, которую Ansible выполняет **по уведомлению об изменении** (`notify`). Типичный случай: конфиг Nginx изменился → нужно проверить и перечитать/перезапустить службу.

```yaml
---
- name: Configure Nginx
  hosts: web
  become: true

  tasks:
    - name: Deploy Nginx configuration
      ansible.builtin.template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: '0644'
      notify: Reload Nginx

  handlers:
    - name: Reload Nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

Handler обычно запускается в конце соответствующего этапа обработки tasks, если был уведомлён; несколько уведомлений одного handler обычно сворачиваются в один запуск. Это избавляет от ненужных рестартов.

При необходимости вызвать ожидающие handlers раньше:

```yaml
- name: Apply pending handlers
  ansible.builtin.meta: flush_handlers
```

Для изменения конфигов полезно предварительно валидировать шаблон через параметр `validate` у `template`/`copy`, если приложение поддерживает проверку файла.
