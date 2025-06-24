# Построчное объяснение плейбука и конфигурации Ansible

## Плейбук: Резервное копирование PostgreSQL

### Общая структура плейбука:
```yaml
- name: To make a backup and dump from pg_db
  hosts: pg_db
  gather_facts: true
```
- **name** - описательное название плейбука
- **hosts: pg_db** - применяется только к хостам из группы pg_db
- **gather_facts: true** - сбор информации о системе перед выполнением задач

### Предварительные задачи (pre_tasks):
```yaml
- name: Install python-psycopg2
  ansible.builtin.apt:
    name: python-psycopg2
    state: present
  tags:
    - install
  when: ((ansible_distribution == "Debian" and ansible_distribution_major_version == "10") or ansible_distribution == "Astra Linux")
```
- Устанавливает python-psycopg2 (Python 2) только для Debian 10 или Astra Linux
- **tags: install** - позволяет запускать только эту задачу с тегом
- **when** - условное выполнение на основе собранных фактов

```yaml
- name: Install python3-psycopg2
  ansible.builtin.apt:
    name: python3-psycopg2
    state: present
  tags:
    - install
  when: ansible_distribution == "Debian" and (ansible_distribution_major_version == "11" or ansible_distribution_major_version == "12")
```
- Устанавливает python3-psycopg2 для Debian 11 и 12
- Аналогично предыдущей задаче, но для Python 3

### Основные задачи (tasks):

#### Проверка пользователя PostgreSQL:
```yaml
- name: Check if PostgreSQL user exists
  ansible.builtin.getent:
    database: passwd
    key: postgres
  register: postgres_user_check
  ignore_errors: yes
  changed_when: false
```
- Проверяет существование пользователя postgres через getent
- **register** сохраняет результат в переменную
- **ignore_errors** - продолжает выполнение при ошибке
- **changed_when: false** - всегда отмечает как неизменяющее систему

```yaml
- name: Debug PostgreSQL user info
  ansible.builtin.debug:
    msg: "PostgreSQL user check result: {{ postgres_user_check }}"
```
- Выводит отладочную информацию о проверке пользователя

```yaml
- name: Verify PostgreSQL user exists
  ansible.builtin.debug:
    msg: "PostgreSQL user exists: {{ postgres_user_check is not failed and postgres_user_check.getent_passwd is defined }}"
  when: postgres_user_check is not failed
```
- Проверяет и выводит, существует ли пользователь postgres

#### Создание директории для хранения:
```yaml
- name: Create storage directory using current date
  ansible.builtin.file:
    path: "/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}"
    state: directory
    mode: '0755'
```
- Создает директорию с путем, включающим имя хоста и текущую дату
- **mode: '0755'** - устанавливает права доступа

#### Работа с PostgreSQL:
```yaml
- name: Collect PostgreSQL databases
  become: true
  become_user: postgres
  community.postgresql.postgresql_info:
    filter: databases
  register: db_info
  changed_when: false
```
- Собирает информацию о базах данных под пользователем postgres
- **become_user: postgres** - выполнение от имени пользователя postgres

```yaml
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
```
- Создает дамп каждой базы данных с именем и датой в /tmp/
- **loop** - перебирает все найденные базы данных
- **when** - выполняется только если есть пользователь postgres и базы данных

#### Проверка созданных дампов:
```yaml
- name: Find PostgreSQL dump files in /tmp/
  ansible.builtin.find:
    paths: /tmp/
    patterns: "*.sql.gz"
    use_regex: no
  register: tmp_dumps
```
- Ищет созданные файлы дампов в /tmp/

```yaml
- name: Display found dump files
  ansible.builtin.debug:
    var: tmp_dumps.files
```
- Выводит информацию о найденных файлах дампов

## Конфигурация Ansible (ansible.cfg)

### Основные настройки:
```ini
[defaults]
inventory = ./inventory/hosts.ini
remote_user = postgres
log_path = /var/log/ansible.log
forks = 1
gathering = smart
```
- **inventory** - путь к файлу инвентаризации
- **remote_user** - пользователь для подключения по умолчанию
- **log_path** - запись логов выполнения
- **forks = 1** - последовательное выполнение (без параллелизма)
- **gathering = smart** - умный сбор фактов (кеширование)

### Кеширование фактов:
```ini
fact_caching = jsonfile
fact_caching_connection = ./facts_cache
fact_caching_timeout = 0
```
- Кеширует факты в JSON-файлы
- **timeout = 0** - кеш никогда не истекает

### Повышение привилегий:
```ini
[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```
- По умолчанию использует sudo для повышения до root
- **become_ask_pass = False** - не запрашивает пароль

### Особенности конфигурации:
1. Удаленный пользователь postgres, но в задачах используется повышение прав до postgres через become
2. Кеширование фактов ускоряет повторные выполнения
3. Последовательное выполнение (forks=1) для избежания перегрузки сервера
4. Подробное логирование в /var/log/ansible.log



