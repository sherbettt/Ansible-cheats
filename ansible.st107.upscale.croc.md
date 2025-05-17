
### Структура директорий проекта:
```
/root/ansible
├── group_vars              # Групповые переменные
│   ├── all.yml             # Переменные общие для всех хостов
│   ├── routers.yml         # Переменные группы роутеров
│   ├── datacenters.yml     # Переменные группы дата-центров
│   └── storages.yml        # Переменные группы хранилищ
├── inventory               # Инвентарь хоста
│   ├── hosts.ini           # Список управляемых машин
│   └── group_vars          # Группы инвентаря
│       ├── routers.yml     
│       ├── datacenters.yml  
│       └── storages.yml    
├── playbooks               # Плейбуки
│   ├── main.yml            # Основной плейбук
│   ├── configure_routers.yml
│   ├── setup_datacenter.yml
│   └── install_storage.yml
└── roles                   # Роли (опционально)
    ├── common
    │   ├── tasks
    │   │   └── main.yml
    │   ├── handlers
    │   │   └── main.yml
    │   ├── templates
    │   ├── vars
    │   │   └── main.yml
    │   ├── defaults
    │   │   └── main.yml
    │   └── meta
    │       └── main.yml
    ├── webserver
    └── database
```

---

### Шаги по созданию структуры проекта:

#### 1. Создаем файл `inventory/hosts.ini`:
Файл инвентаря описывает наши серверы и группирует их по ролям.
```ini
# inventory/hosts.ini
[routers]
router1 st107=router1.st107.upscale.croc ansible_host=192.168.107.254

[domaincontrol]
dc1 st107=dc1.st107.upscale.croc ansible_host=192.168.107.10

[storages]
storage st107=storage.st107.upscale.croc ansible_host=192.168.107.11
```

#### 2. Создаем файл общих переменных (`group_vars/all.yml`):
Общие настройки для всех узлов сети.
```yaml
# group_vars/all.yml
ansible_user: admin  #st107admin
ansible_become: true
ansible_python_interpreter: /usr/bin/python3
```

#### 3. Создаем групповые переменные:
Создаем отдельные файлы для каждой группы устройств, содержащие специфичные для них переменные.
Например, `group_vars/routers.yml`, `group_vars/datacenters.yml`, `group_vars/storages.yml`.

Пример файла для роутера:
```yaml
# group_vars/routers.yml
network_device_type: cisco_ios
snmp_community: public
```

#### 4. Создание плейбоков (`playbooks/main.yml`)
Основной плейбук запускает необходимые задачи на каждом узле.
```yaml
# playbooks/main.yml
---
- name: Configure Routers
  hosts: routers
  become: yes
  gather_facts: no
  tasks:
    - name: Install required packages on routers
      yum:
        name: "{{ item }}"
        state: present
      loop:
        - iptables
        - net-tools

- name: Setup Data Center Server
  hosts: datacenters
  become: yes
  gather_facts: yes
  tasks:
    - name: Update system
      dnf:
        update_cache: yes
        upgrade: yes

- name: Install Storage Software
  hosts: storages
  become: yes
  gather_facts: yes
  tasks:
    - name: Install NFS server package
      apt:
        name: nfs-kernel-server
        state: latest
```

---

### Запуск проекта:
Для запуска плейбука используется команда:
```bash
ansible-playbook -i inventory/hosts.ini playbooks/main.yml
```
-------
Чтобы избежать ручного создания сложной структуры каталогов внутри директории ролей (например, `tasks`, `handlers`, `templates`, etc.) каждый раз при создании новой роли, удобно воспользоваться командой **`ansible-galaxy init`** — встроенным инструментом Ansible Galaxy, который автоматически создает шаблон структуры для новой роли.

### Как создать роль с помощью команды `ansible-galaxy init`
Допустим, вам нужно добавить новую роль под названием `my_role`. Выполните следующую команду:

```bash
ansible-galaxy init my_role
```

Эта команда создаст в вашем проекте директорию с именем `my_role`, содержащую базовую структуру файлов и каталогов, необходимых для полноценной роли Ansible:

```bash
roles/my_role
├── defaults                # Файлы с настройками по умолчанию
│   └── main.yml            
├── files                  # Каталог статичных файлов (шаблоны конфигов, скриптов и др.)
├── handlers               # Обработчики событий (handler'ы)
│   └── main.yml           
├── meta                   # Информация о зависимости от других ролей и метаданные
│   └── main.yml          
├── tasks                  # Основные задачи роли
│   └── main.yml          
├── templates              # Шаблоны Jinja2 для динамической генерации файлов конфигурации
├── tests                  # Директория для тестов (необязательно)
│   ├── inventory          
│   └── test.yml           
└── vars                   # Дополнительные переменные для роли
    └── main.yml
```

Таким образом, создание новых ролей значительно упрощается и стандартизируется. Если вы планируете много ролей, рекомендую этот способ. Он избавляет вас от ручной работы и гарантирует правильную организацию структуры.

---

Теперь вы можете использовать эту роль в ваших плейбуках следующим образом:

```yaml
---
- name: Apply My Role
  hosts: all
  become: yes
  roles:
    - role: my_role
```
