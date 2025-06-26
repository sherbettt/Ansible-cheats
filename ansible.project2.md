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
  ┌─ kirill ~/GIT-projects/backup 
  └─ $ treel -f
  [kirill   kirill           118 Jun 24 17:07]  .
  ├── [kirill   kirill           503 Jun 24 13:15]  ./ansible.cfg
  ├── [kirill   kirill             0 Jun 24 11:44]  ./facts_cache
  ├── [kirill   kirill             0 Jun 24 11:30]  ./group_vars
  ├── [kirill   kirill            18 Jun 24 11:35]  ./inventory
  │   └── [kirill   kirill            21 Jun 24 13:13]  ./inventory/hosts.ini
  ├── [kirill   kirill            50 Jun 24 12:02]  ./playbooks
  │   ├── [kirill   kirill          2016 Jun 24 16:07]  ./playbooks/dump_play.yml
  │   └── [kirill   kirill          3389 Jun 24 12:02]  ./playbooks/playbook.yml
  └── [kirill   kirill             0 Jun 24 11:30]  ./roles
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
  hosts: pg_db
  gather_facts: true

  # vars_files:
  #   - ../../group_vars/secret.yml

  pre_tasks:
    # py installation
    - name: Install python-psycopg2
      ansible.builtin.apt:
        name: python-psycopg2
        state: present
      tags:
        - install
      when: ((ansible_distribution == "Debian" and ansible_distribution_major_version == "10") or ansible_distribution == "Astra Linux")

    - name: Install python3-psycopg2
      ansible.builtin.apt:
        name: python3-psycopg2
        state: present
      tags:
        - install
      when: ansible_distribution == "Debian" and (ansible_distribution_major_version == "11" or ansible_distribution_major_version == "12")


    
  tasks:
    # Create storage directory
       # ansible -i ~/GIT-projects/backup/inventory/hosts.ini pg_db -m debug -a 'var=inventory_hostname'
       # ansible -i ~/GIT-projects/backup/inventory/hosts.ini pg_db -m debug -a 'var=ansible_date_time.date'
         # check https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html
    - name: Create storage directory using current date
      ansible.builtin.file:
        path: "/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}"
        state: directory
        mode: '0755'

    # Get database info
        # check https://docs.ansible.com/ansible/latest/collections/community/postgresql/postgresql_info_module.html
        # field filter
    - name: Gather information about PostgreSQL databases only from servers
      become: true
      become_user: postgres
      community.postgresql.postgresql_info:
        filter: databases
      register: db_info
      changed_when: false

    - name: Display/Debug info (db_info.databases.postgres.extensions.plpgsql)
      ansible.builtin.debug:
        var: db_info.databases.postgres.extensions.plpgsql

    - name: Display/Debug all database info (db_info.databases)
      ansible.builtin.debug:
        var: db_info.databases

    # Create DB dump
        # Check https://docs.ansible.com/ansible/latest/collections/ansible/builtin/find_module.html#return-values
    - name: Dump an existing database to a file (with compression)
      become: true
      become_method: su
      become_user: postgres
      community.postgresql.postgresql_db:
        name: "{{ item }}"  # Просто используем сам элемент, который является именем БД
        state: dump
        target: "/tmp/{{ item }}-{{ ansible_date_time.date }}.sql.gz"
      loop: "{{ db_info.databases.keys() }}"  # Проходим по ключам словаря
      when: 
        - db_info.databases | length > 0    # Только когда db_info.databases существует
      loop_control:
        label: "{{ item }}"

    # Check created dump files
    - name: Find PostgreSQL dump files in /tmp/
      ansible.builtin.find:
        paths: /tmp/
        patterns: "*.sql.gz"
        use_regex: no
      register: tmp_dumps

    - name: Display found dump files (paths)
      ansible.builtin.debug:
        msg: "File path: {{ item.path }}, Size: {{ item.size }} bytes, Cred: {{ item.mode }}"
      loop: "{{ tmp_dumps.files }}"
      when: tmp_dumps.matched > 0    # Выполнять только если файлы найдены

    - name: Display number of found dump files
      ansible.builtin.debug:
        msg: "кол-во выводимых в /tmp/ *.sql.gz: {{ tmp_dumps.matched }}"
#        var: tmp_dumps.matched        # кол-во выводимых *.sql.gz


    - name: Ensure destination directory exists on backup server
      delegate_to: 192.168.87.99
      ansible.builtin.file:
        path: "/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}"
        state: directory
        mode: '0755'

    - name: Check SSH connection to backup server
      ansible.builtin.ping:
      delegate_to: "{{ groups['targets'][0] }}"
      register: ssh_check
      
    - name: Fail if SSH check failed
      fail:
        msg: "Cannot connect to backup server via SSH"
      when: ssh_check is failed

    - name: Sync dumps to backup server (small dumps)
      ansible.posix.synchronize:
        mode: push
        src: "{{ item.path }}"
#        dest: "{{ groups['targets'][0] }}:/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}/"
        dest: "/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}/"
        rsync_opts:
          - "--inplace"
          - "--perms"
          - "--verbose"
          - "--progress"
          - "--timeout=150"
          - "--no-delay-updates"
        private_key: "~/.ssh/id_rsa"  # Укажите путь к ключу при необходимости
        archive: no                   # Отключаем автоматические опции (-rlptgoD)
        checksum: yes
        compress: yes
      delegate_to: "{{ groups['targets'][0] }}"
#      loop: "{{ tmp_dumps.files }}"
      loop: "{{ tmp_dumps.files | selectattr('size', 'lt', 1048576) | list }}"  # Файлы <1MB
      when: tmp_dumps.matched > 0 and ssh_check is success
      register: sync_results
      ignore_errors: yes
      changed_when: sync_results.rc == 0 or sync_results.rc == 23
      failed_when: sync_results.rc not in [0, 23, 24, 30]


    - name: Sync dumps to backup server (big dumps)
      ansible.posix.synchronize:
        mode: push
        src: "{{ item.path }}"
#        dest: "{{ groups['targets'][0] }}:/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}/"
        dest: "/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}/"
        rsync_opts:
          - "--inplace"
          - "--perms"
          - "--verbose"
          - "--progress"
          - "--timeout=150"
          - "--no-delay-updates"
        private_key: "~/.ssh/id_rsa"  # Укажите путь к ключу при необходимости
        archive: no                   # Отключаем автоматические опции (-rlptgoD)
        checksum: yes
        compress: yes
      delegate_to: "{{ groups['targets'][0] }}"
      loop: "{{ tmp_dumps.files | selectattr('size', 'ge', 1048576) | list }}"  # Файлы >=1MB
      when: tmp_dumps.matched > 0 and ssh_check is success
      register: sync_results
      ignore_errors: yes
      changed_when: sync_results.rc == 0 or sync_results.rc == 23
      failed_when: sync_results.rc not in [0, 23, 24, 30]



```
Предварителньо проверим:
```bash
ansible -i ~/GIT-projects/backup/inventory/hosts.ini --list-hosts targets;
ansible -i ~/GIT-projects/backup/inventory/hosts.ini targets -m ping;
```

Запускаем:
```bash
sudo ansible-playbook -C -i ~/GIT-projects/backup/inventory/hosts.ini dump_play.yml
```



