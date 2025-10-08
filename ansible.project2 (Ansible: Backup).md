# Ansible poject: Backup

Управляющая машина `192.168.87.8`, на которую установлен semaphore.
<br/> Управляемая машина `192.168.87.70`, на которую уже установлена БД PostgresQL 15.
<br/> Ansible конфигурация хранится в GitLab.
<br/> Редактирование кода идёт на `192.168.87.136`.

На управляемых машинах требуется провести действия:
  - создать dump базы данных и сжать в формате .gz;
  - переслать полученный .gz на `192.168.87.136`, а после - на SFTP;
-----------------------------------------------------------------------

## 1. Для начала зайдём в БД и выясним имя самой БД

### 1. Убедимся, что есть доступ через `psql`.
```c
└─ # su - postgres 
postgres@dev70:~$ whoami 
postgres
postgres@dev70:~$ psql 
psql (15.13 (Debian 15.13-0+deb12u1))
Введите "help", чтобы получить справку.

psql -U username -d dbname
```

Находясь будучи в psql, используй команды:
- `\l` or `\list` , `\l+`
- `\db` , `\du`
- `\?`, `\help`, [Шпаргалка по основным командам PostgreSQL](https://www.oslogic.ru/knowledge/598/shpargalka-po-osnovnym-komandam-postgresql/)

### 2. Убедимся, что PostgreSQL слушает подключения (если знаем пароль)
```bash
psql -h 192.168.87.70 -U postgres -W
```
-----------------------------------------------------------------------

## 2. Начнём писать проект Ansible

### 1. Создаём структуру проекта на `192.168.87.136`/GitLab.
  ```ini
  └─ $ treel -f
  [drwxr-xr-x]  .
  ├── [-rw-r--r--]  ./ansible.cfg
  ├── [drwxr-xr-x]  ./group_vars
  ├── [drwxr-xr-x]  ./inventory
  │   └── [-rw-r--r--]  ./inventory/hosts.ini
  ├── [drwxr-xr-x]  ./playbooks
  │   ├── [-rw-r--r--]  ./playbooks/check_user.yml
  │   ├── [-rw-r--r--]  ./playbooks/del_sql.yaml
  │   ├── [-rw-r--r--]  ./playbooks/dump_play.yml
  │   └── [-rw-r--r--]  ./playbooks/playbook.yml
  └── [drwxr-xr-x]  ./roles

  ```

#### 2. `ansible.cfg`
```ini
[defaults]
inventory = ./inventory/hosts.ini
remote_user = postgres
log_path = /var/log/ansible.log
forks = 1
gathering = smart

# Caching of facts
fact_caching = jsonfile
fact_caching_connection = ./facts_cache
fact_caching_timeout = 0

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```

#### 3. `inventory/hosts.ini`
```ini
[pg_db]
192.168.87.70 ansible_user=root 
#ansible_ssh_private_key_file=~/.ssh/ansible_key

[pg_db:vars]
ansible_python_interpreter=/usr/bin/python3.11
# reason is https://docs.ansible.com/ansible-core/2.18/reference_appendices/interpreter_discovery.html

[targets]
192.168.87.99 ansible_user=root
```

#### 4. Сделаем ручной опрос баз данных
```bash
sudo ansible -i ~/GIT-projects/backup/inventory/hosts.ini postgres -m postgresql_info;
sudo ansible -i ~/GIT-projects/backup/inventory/hosts.ini pg_db -m postgresql_info -a 'filter=dat*,rol*';
sudo ansible -i ~/GIT-projects/backup/inventory/hosts.ini pg_db -m postgresql_info -a 'filter=dat*,rol* login_user=postgres login_password=ваш_пароль login_host=192.168.87.70'
```
Но скорее всего возникнет ошибка аутентификации. 
<br/> Можно попробовать отредактировать `/etc/postgresql/15/main/pg_hba.conf`, заменив строку:
```text
local   all             postgres                                peer
```
на строку
```text
local   all             postgres                                md5
# Или для сети:
host    all             postgres        192.168.87.0/24        md5
```


#### 5. `playbooks/dump_play.yml`
Установим `ansible.*` модули:
```bash
ansible-galaxy collection install ansible.posix {--force};
ansible-galaxy collection install community.general;
ansible-galaxy collection install ansible.utils;
ansible-galaxy collection install community.postgresql;

ansible-galaxy collection list | grep posix {postgresql|utils|<etc>};
```
Проверим путь до переменной COLLECTIONS_PATHS:
```bash
ansible-config dump | grep COLLECTIONS_PATHS
```
Проверьте, что rsync установлен на управляющем узле:
```bash
which rsync || sudo apt install rsync
```

Учитывая секцию *`[targets]`* в **`inventory/hosts.ini`**, мы будем перекидывать бэкап с `*.87.70` машины на `*.87.99`, а не на ноут (`*.87.136`).
<br/> Желательно проверить настройки **`sshd_config`** на обоих серверах и проверить командами `rsync`:
```bash
ssh root@192.168.87.99  # from *.87.70;
```
```bash
# На pg_db (192.168.87.70):
which rsync || apt install rsync -y
# На backup-сервере, он же target (192.168.87.99):
which rsync || apt install rsync -y
```
```bash
На pg_db (192.168.87.70) выполните:
# Создаем тестовый файл
echo "This is a test file" > /tmp/test_rsync.txt
chmod 644 /tmp/test_rsync.txt
ls -la /tmp/test_rsync.txt

# На 192.168.87.99 выполнить:
mkdir -p "/usr/local/runtel/storage_files/telecoms/runtel.org/dev70/$(date +%Y-%m-%d)"
ls -alF "/usr/local/runtel/storage_files/telecoms/runtel.org/dev70/"

# Пробуем отправить его на backup-сервер
rsync -avz --progress /tmp/test_rsync.txt root@192.168.87.99:/usr/local/runtel/storage_files/telecoms/runtel.org/dev70/$(date +%Y-%m-%d)/
rsync -avz --progress -e "ssh -i ~/.ssh/ansible_key" /tmp/test_rsync.txt root@192.168.87.99:/usr/local/runtel/storage_files/telecoms/runtel.org/$(hostname)/$(date +%Y-%m-%d)/
rsync -avz --progress /tmp/ваша_база-$(date +%Y-%m-%d).sql.gz root@192.168.87.99:/usr/local/runtel/storage_files/telecoms/runtel.org/$(hostname)/$(date +%Y-%m-%d)/
```
```c
ssh root@192.168.87.99 "ls -la /usr/local/runtel/storage_files/telecoms/runtel.org/dev70/$(date +%Y-%m-%d)/"
ssh root@192.168.87.99 "cat /usr/local/runtel/storage_files/telecoms/runtel.org/dev70/$(date +%Y-%m-%d)/test_rsync.txt"
```


(Читай [Playbooks Delegation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_delegation.html) )
```yaml
---
- name: To make a backup and dump from pg_db
  hosts: all
  gather_facts: true

  # vars_files:
  #   - ../../group_vars/secret.yml

  vars:
    # Количество бекапов для хранения (можно переопределить через переменную окружения)
    backup_retention_days: "{{ (lookup('env', 'BACKUP_RETENTION_DAYS') | default('14', true)) | int }}"
    #postgresql_password: "{{ lookup('file', playbook_dir + '/../group_vars/db_password.txt') }}"

  pre_tasks:
  # задаём время для папки на backup машине
    - name: Get current timestamp from backup machine
      set_fact:
        current_timestamp: "{{ lookup('pipe', 'date +%Y-%m-%d_%H-%M-%S') }}"

    # py installation
    - name: Install python-psycopg2
      ansible.builtin.apt:
        name: python-psycopg2
        state: present
      tags:
        - install
      when: >
        (ansible_distribution == "Debian" and ansible_distribution_major_version == "10") or
        ansible_distribution == "Astra Linux" or
        ansible_distribution == "redos"

    - name: Install python3-psycopg2
      ansible.builtin.apt:
        name: python3-psycopg2
        state: present
      tags:
        - install
      when: >
        ansible_distribution == "Debian" and
        ansible_distribution_major_version in ["8", "11", "12"]

    - name: Install rsync
      ansible.builtin.apt:
        name: rsync
        state: present

  tasks:
    - name: System variables
      debug:
        msg:
        - "{{ lookup('env', 'PWD') }}"
        - "{{ lookup('env', 'HOME') }}"
        - "{{ lookup('env', 'PATH') }}"
        - "{{ lookup('env', 'LC_CTYPE') }}"
        - "{{ lookup('env', 'USER') }}"
        - "{{ ansible_play_hosts }}"
        - "{{ inventory_dir }}"
        - "{{ playbook_dir }}"

    # Create storage directory
    - name: Create storage directory using current date
      ansible.builtin.file:
        path: "/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}"
        state: directory
        mode: '0755'


# Если postgresql_host не определен в inventory:vars хоста, то будет выполнено подключение через pgsql.sock
# пример inventory без postgresql_host
#
# [somehost]
# 177.177.177.177 ansible_user=root
#

    - name: Gather information about PostgreSQL databases only from servers (local socket)
      become: true
      become_user: postgres
      community.postgresql.postgresql_info:
        filter: databases, settings
      register: db_info
      when: postgresql_host is not defined      # Локальное подключение через socket


# Если postgresql_host определен в inventory:vars хоста, то будет выполнено подключение к нему
# пример inventory c postgresql_host
#
# [somehost]
# 177.177.177.177 ansible_user=root
# [somehost:vars]
# postgresql_host = "192.168.1.10"
# postgresql_port = 5432
# postgresql_user = postgres
#
# запись идет в перемменную с другим именем, db_info_with_host, а затем значение этой переменной идет в fact db_info,
# чтобы дальше в рамках playbook можно было выполнять проверки и крутить их в loop


    - name: Gather information about PostgreSQL databases only from servers (remote host)
      become: true
      become_user: postgres
      community.postgresql.postgresql_info:
        filter: databases,settings
        login_host: "{{ postgresql_host }}"
        login_port: "{{ postgresql_port | default(5432) }}"
        login_user: "{{ postgresql_user | default('postgres') }}"
        login_password: "{{ postgresql_password | default(omit) }}"
      register: db_info_with_host
      when: postgresql_host is defined          # Сетевое подключение

    - name: Debug db_info structure
      ansible.builtin.debug:
        var: db_info
      when: postgresql_host is not defined

    - name: Debug db_info_with_host structure
      ansible.builtin.debug:
        var: db_info_with_host
      when: postgresql_host is defined

    - name: Extract PostgreSQL major version from db_info_with_host
      set_fact:
        pg_major_version: "{{ db_info_with_host.postgresql.server_version | regex_search('\\d+') | int }}"
      when: 
        - postgresql_host is defined
        - db_info_with_host.postgresql.server_version is defined

    - name: Extract PostgreSQL major version from db_info
      set_fact:
        pg_major_version: "{{ db_info.postgresql.server_version | regex_search('\\d+') | int }}"
      when: 
        - postgresql_host is not defined
        - db_info.postgresql.server_version is defined

    - name: Set safe default PostgreSQL version
      set_fact:
        pg_major_version: 15
      when: pg_major_version is not defined

    - name: set db_info fact
      set_fact:
        db_info: "{{ db_info_with_host }}"
      when: postgresql_host is defined

    - name: Display/Debug info (db_info)
      ansible.builtin.debug:
        var: db_info
      when: db_info is defined

    - name: Display/Debug info (db_info.databases.postgres.extensions.plpgsql)
      ansible.builtin.debug:
        var: db_info.databases.postgres.extensions.plpgsql
      when: db_info is defined

    - name: Display PostgreSQL version on remote host
      ansible.builtin.debug:
        msg: "PostgreSQL major version: {{ pg_major_version }}"
      when: pg_major_version is defined



    # Create DB dump
    - name: Dump an existing database to a file (with compression) [remote connection]
      become: true
      become_method: su
      become_user: postgres
      community.postgresql.postgresql_db:
        login_host: "{{ postgresql_host }}"
        login_port: "{{ postgresql_port | default(5432) }}"
        login_user: "{{ postgresql_user | default('postgres') }}"
        login_password: "{{ postgresql_password | default(omit) }}"
        name: "{{ item }}"                                    # Просто используем сам элемент, который является именем БД
        state: dump
        #target: "/tmp/{{ item }}-{{ ansible_date_time.date }}.sql.gz"
        target: "/tmp/{{ item }}-{{ current_timestamp }}.sql.gz"
        dump_extra_args: "{{ '--no-synchronized-snapshots' if pg_major_version is defined and pg_major_version < 15 else '' }}"
      loop: "{{ db_info.databases.keys() }}"                            # Проходим по ключам словаря
      when:
        - db_info.databases | length > 0                          # Только когда db_info.databases существует
        - item not in ['postgres','template1','template2']        # Фильтрация баз данных
        - postgresql_host is defined
      #ignore_errors: yes
      register: dump_state1
      loop_control:
        label: "{{ item }}"


    - name: Dump an existing database to a file (with compression) [local socket]
      become: true
      become_method: su
      become_user: postgres
      community.postgresql.postgresql_db:
#        login_host: "{{ postgresql_host }}"
#        login_port: "{{ postgresql_port }}"
#        login_user: "{{ postgresql_user }}"
        #login_password: "{{ postgresql_password }}"
        name: "{{ item }}"                                    # Просто используем сам элемент, который является именем БД
        state: dump
        target: "/tmp/{{ item }}-{{ current_timestamp }}.sql.gz"
        dump_extra_args: "{{ '--no-synchronized-snapshots' if pg_major_version is defined and pg_major_version < 15 else '' }}"
      loop: "{{ db_info.databases.keys() }}"                            # Проходим по ключам словаря
      when:
        - db_info.databases | length > 0                          # Только когда db_info.databases существует
        - item not in ['postgres','template1','template2']        # Фильтрация баз данных
        - postgresql_host is not defined
      #ignore_errors: yes
      register: dump_state2
      loop_control:
        label: "{{ item }}"


    # Check created dump files
            #Модуль find возвращает список файлов, где каждый item содержит:
            #item.path - полный путь к файлу (например /tmp/mydb-2024-09-28_14-30-25.sql.gz)
            #item.size - размер файла
            #item.mtime - время модификации  и другие атрибуты
    - name: Find PostgreSQL dump files in /tmp/
      ansible.builtin.find:
        paths: /tmp/
        patterns: "*.sql.gz"
        #patterns: "*-{{ current_timestamp }}.sql.gz"
        use_regex: no
      tags: fdump
      register: tmp_dumps

    - name: Display number of found dump files
      ansible.builtin.debug:
        msg: "File path: {{ item.path }}, Size: {{ item.size }} bytes, Cred: {{ item.mode }}"
      tags: fdump
      loop: "{{ tmp_dumps.files }}"
      when: tmp_dumps.matched > 0  # Выполнять только если файлы найдены

    - name: Display number of found dump files
      ansible.builtin.debug:
        msg: "кол-во выводимых в /tmp/ *.sql.gz: {{ tmp_dumps.matched }}"
        #var: tmp_dumps.matched        # кол-во выводимых *.sql.gz
      tags: fdump


    - name: Ensure destination directory exists on backup server
      delegate_to: 192.168.87.99
      ansible.builtin.file:
        #path: "/usr/local/runtel/storage_files/telecoms/{{ inventory_hostname }}/"
        #path: "/usr/local/runtel/storage_files/telecoms/{{ inventory_hostname }}/{{ ansible_date_time.date }}"
        path: "/usr/local/runtel/storage_files/telecoms/{{ inventory_hostname }}/{{ current_timestamp.split('_')[0] }}"  # Берем только дату из метки
        state: directory
        mode: '0755'

    - name: Sync dumps to backup server
      delegate_to: 192.168.87.99
      ansible.posix.synchronize:
        mode: pull
        src: "{{ item.path }}"
        dest: "/usr/local/runtel/storage_files/telecoms/{{ inventory_hostname }}/{{ current_timestamp.split('_')[0] }}/"  # Только дата для папки
        #dest: "/usr/local/runtel/storage_files/telecoms/{{ inventory_hostname }}/{{ ansible_date_time.date }}-{{ ansible_date_time.time | replace(':', '-') }}/"
        rsync_opts:
          - "--perms"
          - "--verbose"
        archive: no
        checksum: yes
        compress: yes
      loop: "{{ tmp_dumps.files }}"
      when: tmp_dumps.matched > 0
      register: sync_results
      changed_when: sync_results.rc == 0 or sync_results.rc == 23
      failed_when: sync_results.rc not in [0, 23, 24, 30]
      tags: backup

     # Current step uses 'Find PostgreSQL dump files in /tmp/' step
     # Just delete dump files from 192.168.87.70
    - name: Remove PostgreSQL dump files from /tmp/
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: absent
      tags: fdump
      loop: "{{ tmp_dumps.files }}"
      when: tmp_dumps.matched > 0
      loop_control:
        label: "{{ item.path }}"
      register: cleanup_result

    - name: Display cleanup results
      ansible.builtin.debug:
        msg: "Successfully removed {{ cleanup_result.results | length }} dump files"
      tags: fdump
      when: tmp_dumps.matched > 0

    # Ротация бекапов на сервере 192.168.87.99
    - name: Find all backup directories for current host
      delegate_to: 192.168.87.99
      ansible.builtin.find:
        paths: "/usr/local/runtel/storage_files/telecoms/{{ inventory_hostname }}/"
        patterns: "*"
        file_type: directory
        use_regex: no
      register: backup_dirs
      tags: rotation

    - name: Sort backup directories by modification time (newest first)
      delegate_to: 192.168.87.99
      ansible.builtin.set_fact:
        sorted_backup_dirs: "{{ backup_dirs.files | sort(attribute='mtime', reverse=true) }}"
      tags: rotation

    - name: Display backup retention policy
      ansible.builtin.debug:
        msg: "Save {{ backup_retention_days }} backups. found {{ sorted_backup_dirs | length }} directories"
      tags: rotation

    - name: Remove old backup directories
      delegate_to: 192.168.87.99
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: absent
      loop: "{{ sorted_backup_dirs[backup_retention_days|int:] }}"
      when:
        - sorted_backup_dirs | length > (backup_retention_days | int)
        - backup_retention_days | int > 0
      loop_control:
        label: "{{ item.path }}"
      register: rotation_result
      tags: rotation

    - name: Display rotation results
      ansible.builtin.debug:
        msg: "Removed {{ rotation_result.results | length }} old backup directories"
      when: rotation_result is defined and rotation_result.results is defined
      tags: rotation
```
Предварителньо проверим:
```bash
ansible -i ~/GIT-projects/backup/inventory/hosts.ini --list-hosts targets;
ansible -i ~/GIT-projects/backup/inventory/hosts.ini targets -m ping;
```

Запускаем:
```bash
ansible-playbook -C -i ~/GIT-projects/backup/inventory/hosts.ini dump_play.yml
ansible-playbook -i ~/GIT-projects/backup/inventory/hosts.ini dump_play.yml --tags install
ansible-playbook -i ~/GIT-projects/backup/inventory/hosts.ini dump_play.yml --tags fdump
```

#### 6. `playbooks/del_sql.yaml` (Опционально)

Данный плей нужен для удаления дампов в полуавтоматическом режиме, чтобы они не оставались на машине `192.168.87.70` именно в том случае, если что-то в основном плее пошло не так.
```yaml
---
- name: Cleanup PostgreSQL dump files from /tmp/
  hosts: pg_db
  gather_facts: true

  tasks:
    - name: Find PostgreSQL dump files in /tmp/
      ansible.builtin.find:
        paths: /tmp/
        patterns: "*.sql.gz"
        use_regex: no
      register: tmp_dumps

    - name: Display number of found dump files
      ansible.builtin.debug:
        msg: "Обнаружено {{ tmp_dumps.matched }} dump файлов на удаление"

    - name: Remove PostgreSQL dump files from /tmp/
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: absent
      loop: "{{ tmp_dumps.files }}"
      when: tmp_dumps.matched > 0
      loop_control:
        label: "{{ item.path }}"
      register: cleanup_result

    - name: Display cleanup results
      ansible.builtin.debug:
        msg: "Successfully removed {{ cleanup_result.results | length }} dump files"
      when: tmp_dumps.matched > 0
```
