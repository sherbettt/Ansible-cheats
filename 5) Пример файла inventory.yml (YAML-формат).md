Приведу пример сложного inventory-файла Ansible с комментариями и пояснениями, а затем подробно разберу его структуру.

### Пример файла `inventory.yml` (YAML-формат):
```yaml
---
all:
  children:
    production:
      children:
        webservers:
          hosts:
            web1.prod.example.com:
              ansible_host: 192.168.1.10
              server_role: frontend
            web2.prod.example.com:
              ansible_host: 192.168.1.11
              server_role: backend
          vars:
            http_port: 80
            ansible_user: prod-admin
            package_version: "1.2.3"

        databases:
          hosts:
            db1.prod.example.com:
              ansible_host: 192.168.1.20
              ansible_connection: docker
              db_engine: postgresql
            db2.prod.example.com:
              ansible_host: 192.168.1.21
              ansible_connection: ssh
              db_engine: mysql

      vars:
        env: production
        ansible_ssh_private_key_file: ~/.ssh/prod_key.pem

    staging:
      hosts:
        staging.example.com:
          ansible_host: 192.168.2.100
          ansible_connection: local
          env_specific_var: "test-config"

    network_devices:
      hosts:
        cisco_switch:
          ansible_host: 192.168.3.1
          ansible_network_os: ios
          ansible_connection: network_cli
          ansible_user: network-admin

    _hypervisors:
      hosts:
        hypervisor01:
          ansible_host: 192.168.4.10
          ansible_connection: libvirt_lxc
          vm_capacity: high
```

---

### Построчное описание с пояснениями:

1. **`all:`**  
   Корневая группа, содержащая все хосты и группы.

2. **`children:`**  
   Определяет дочерние группы. Здесь начинается иерархия.

3. **`production:`**  
   Группа для production-окружения. Наследует параметры от `all`.

4. **`children:`** внутри production:  
   Вложенные группы внутри production.

5. **`webservers:`**  
   Группа веб-серверов. Содержит два хоста:
   - **`web1.prod.example.com`** и **`web2.prod.example.com`**  
     Указаны явные IP через `ansible_host` (модуль `ansible.builtin.setup`), так как DNS может быть недоступен.
   - **`server_role`**  
     Кастомная переменная для роли сервера (используется в playbook).
   - **`vars:`**  
     Групповые переменные:
     - `http_port: 80` — параметр для конфигурации веб-сервера (например, в `nginx`).
     - `ansible_user: prod-admin` — модуль `ansible.builtin.user` для подключения от имени этого пользователя.
     - `package_version: "1.2.3"` — версия пакета для установки через `ansible.builtin.apt`.

6. **`databases:`**  
   Группа баз данных:
   - **`db1.prod.example.com`** использует `docker`-соединение (модуль `community.docker.docker_container`), что позволяет управлять контейнером напрямую.
   - **`db2.prod.example.com`** использует стандартное SSH-соединение.
   - **`db_engine`** — переменная для выбора конфигурации СУБД в playbook.

7. **`vars:`** в production:  
   Глобальные переменные для всего production-окружения:
   - `env: production` — метка окружения.
   - `ansible_ssh_private_key_file` — путь к SSH-ключу (модуль `ansible.builtin.file`), используется для аутентификации.

8. **`staging:`**  
   Отдельная группа для staging:
   - **`ansible_connection: local` — выполнение задач на локальной машине через модуль `ansible.builtin.command`.
   - **`env_specific_var`** — пример переменной для специфичных настроек.

9. **`network_devices:`**  
   Группа сетевых устройств:
   - **`ansible_network_os: ios`** — модуль `cisco.ios.ios_config` для работы с Cisco IOS.
   - **`ansible_connection: network_cli`** — специализированное соединение для сетевого оборудования.

10. **`_hypervisors:`**  
    Группа с префиксом `_` (часто используется для служебных групп):
    - **`ansible_connection: libvirt_lxc`** — управление LXC-контейнерами через libvirt (модуль `community.libvirt.virt`).
    - **`vm_capacity: high`** — кастомная переменная для выделения ресурсов.

---

### Ключевые модули и причины их использования:
1. **`ansible_connection`**  
   - `ssh` (по умолчанию) — для Linux/Unix-хостов.
   - `docker` — управление контейнерами без SSH.
   - `network_cli` — взаимодействие с сетевым оборудованием через CLI.
   - `local` — выполнение задач на control-node.

2. **`ansible_user`**  
   Определяет пользователя для подключения. Используется модулем `ansible.builtin.user`.

3. **`ansible_ssh_private_key_file`**  
   Указание SSH-ключа (альтернатива парольной аутентификации).

4. **`ansible_network_os`**  
   Требуется для сетевых модулей (например, `cisco.ios.ios_command`).

---

### Иерархия переменных (от высшего к низшему):
1. Глобальные переменные в `all`.
2. Переменные группы `production`.
3. Переменные подгруппы (например, `webservers`).
4. Переменные хоста (например, `web1.prod.example.com`).

---

### Динамический инвентарь (дополнение):
Для сложных сценариев можно добавить плагины, например:
```yaml
plugin: aws_ec2
regions:
  - us-east-1
filters:
  tag:env: production
```
Это автоматически добавит хосты из AWS EC2 с тегом `env: production`.

---

Такой inventory-файл позволяет:
- Четко разделять окружения.
- Использовать разные способы подключения.
- Централизованно управлять переменными.
- Масштабироваться за счет групп и подгрупп.

