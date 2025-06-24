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
192.168.87.70
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
```yaml
---
- name: To make a backup and dump from pg_db
  hosts: pg_db
  gather_facts: true

  # vars_files:
  #   - ../../group_vars/secret.yml

  pre_tasks:
    # Pre-tasks section, installation
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
    # Main tasks section
    - name: Check if PostgreSQL user exists
      ansible.builtin.getent:
        database: passwd
        key: postgres
      register: postgres_user_check
      ignore_errors: yes
      changed_when: false

    - name: Debug PostgreSQL user info
      ansible.builtin.debug:
        msg: "PostgreSQL user check result: {{ postgres_user_check }}"

    - name: Verify PostgreSQL user exists
      ansible.builtin.debug:
        msg: "PostgreSQL user exists: {{ postgres_user_check is not failed and postgres_user_check.getent_passwd is defined }}"
      when: postgres_user_check is not failed

    # Create storage directory
    - name: Create storage directory using current date
      ansible.builtin.file:
        path: "/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}"
        state: directory
        mode: '0755'

    # Get database info
    - name: Collect PostgreSQL databases
      become: true
      become_user: postgres
      community.postgresql.postgresql_info:
        filter: databases
      register: db_info
      changed_when: false

    - name: Debug database info
      ansible.builtin.debug:
        var: db_info.databases

    # Create DB dump
    - name: Dump an existing database to a file (with compression)
      become: true
      become_method: su
      become_user: postgres
      community.postgresql.postgresql_db:
        name: "{{ item.name }}"
        state: dump
        target: "/tmp/{{ item.name }}-{{ ansible_date_time.date }}.sql.gz"
      loop: "{{ db_info.databases | dict2items }}"
      when: 
        - postgres_user_check is not failed
        - postgres_user_check.getent_passwd is defined
        - db_info.databases | length > 0
      loop_control:
        label: "{{ item.key }}"

    # Check created dump files
    - name: Find PostgreSQL dump files in /tmp/
      ansible.builtin.find:
        paths: /tmp/
        patterns: "*.sql.gz"
        use_regex: no
      register: tmp_dumps

    - name: Display found dump files
      ansible.builtin.debug:
        var: tmp_dumps.files


```






