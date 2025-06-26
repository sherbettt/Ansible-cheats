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





## Разбор Ansible Playbook для резервного копирования PostgreSQL

Этот playbook выполняет резервное копирование баз данных PostgreSQL на удаленный сервер. Давайте разберем его построчно:

### Основные директивы playbook

```yaml
- name: To make a backup and dump from pg_db
  hosts: pg_db
  gather_facts: true
```
- **name** - Описание задачи/playbook
- **hosts: pg_db** - Определяет группу хостов, на которых будет выполняться playbook
- **gather_facts: true** - Включает сбор фактов о системе перед выполнением задач

### Pre-tasks (Предварительные задачи)

```yaml
pre_tasks:
  - name: Install python-psycopg2
    ansible.builtin.apt:
      name: python-psycopg2
      state: present
    tags:
      - install
    when: ((ansible_distribution == "Debian" and ansible_distribution_major_version == "10") or ansible_distribution == "Astra Linux")
```
- Устанавливает python-psycopg2 для Debian 10 или Astra Linux
- **when** - Условное выполнение задачи
- **tags** - Позволяет запускать только помеченные задачи

```yaml
  - name: Install python3-psycopg2
    ansible.builtin.apt:
      name: python3-psycopg2
      state: present
    tags:
      - install
    when: ansible_distribution == "Debian" and (ansible_distribution_major_version == "11" or ansible_distribution_major_version == "12")
```
- Аналогично, но для Debian 11/12 устанавливает python3-версию

### Основные задачи (Tasks)

#### Создание директории для хранения

```yaml
- name: Create storage directory using current date
  ansible.builtin.file:
    path: "/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}"
    state: directory
    mode: '0755'
```
- Создает директорию с именем текущей даты для хранения резервных копий
- **mode** - Устанавливает права доступа 0755

#### Получение информации о базах данных

```yaml
- name: Gather information about PostgreSQL databases only from servers
  become: true
  become_user: postgres
  community.postgresql.postgresql_info:
    filter: databases
  register: db_info
  changed_when: false
```
- **become/become_user** - Выполняет задачу от имени пользователя postgres
- **filter: databases** - Собирает только информацию о базах данных
- **register** - Сохраняет результат в переменную db_info
- **changed_when: false** - Всегда отмечает задачу как неизмененную

#### Отладка информации

```yaml
- name: Display/Debug info (db_info.databases.postgres.extensions.plpgsql)
  ansible.builtin.debug:
    var: db_info.databases.postgres.extensions.plpgsql
```
- Выводит отладочную информацию о расширении plpgsql

```yaml
- name: Display/Debug all database info (db_info.databases)
  ansible.builtin.debug:
    var: db_info.databases
```
- Выводит полную информацию о всех базах данных

#### Создание дампов баз данных

```yaml
- name: Dump an existing database to a file (with compression)
  become: true
  become_method: su
  become_user: postgres
  community.postgresql.postgresql_db:
    name: "{{ item }}"
    state: dump
    target: "/tmp/{{ item }}-{{ ansible_date_time.date }}.sql.gz"
  loop: "{{ db_info.databases.keys() }}"
  when: 
    - db_info.databases | length > 0
  loop_control:
    label: "{{ item }}"
```
- Создает сжатые дампы всех баз данных
- **loop** - Итерируется по всем именам баз данных
- **loop_control.label** - Упрощает вывод имени текущей БД в логах
- **when** - Выполняет только если есть базы данных

#### Поиск созданных дампов

```yaml
- name: Find PostgreSQL dump files in /tmp/
  ansible.builtin.find:
    paths: /tmp/
    patterns: "*.sql.gz"
    use_regex: no
  register: tmp_dumps
```
- Ищет созданные файлы дампов в /tmp/
- **register** - Сохраняет результат в переменную tmp_dumps

#### Отладка информации о найденных дампах

```yaml
- name: Display found dump files (paths)
  ansible.builtin.debug:
    msg: "File path: {{ item.path }}, Size: {{ item.size }} bytes, Cred: {{ item.mode }}"
  loop: "{{ tmp_dumps.files }}"
  when: tmp_dumps.matched > 0
```
- Выводит информацию о каждом найденном файле дампа

```yaml
- name: Display number of found dump files
  ansible.builtin.debug:
    msg: "кол-во выводимых в /tmp/ *.sql.gz: {{ tmp_dumps.matched }}"
```
- Выводит количество найденных файлов дампов

#### Подготовка к копированию на backup сервер

```yaml
- name: Ensure destination directory exists on backup server
  delegate_to: 192.168.87.99
  ansible.builtin.file:
    path: "/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}"
    state: directory
    mode: '0755'
```
- **delegate_to** - Выполняет задачу на backup сервере
- Создает целевую директорию на backup сервере

```yaml
- name: Check SSH connection to backup server
  ansible.builtin.ping:
  delegate_to: "{{ groups['targets'][0] }}"
  register: ssh_check
```
- Проверяет SSH соединение с backup сервером
- **register** - Сохраняет результат проверки

```yaml
- name: Fail if SSH check failed
  fail:
    msg: "Cannot connect to backup server via SSH"
  when: ssh_check is failed
```
- Прерывает выполнение при неудачной проверке SSH

#### Копирование дампов на backup сервер (маленькие файлы)

```yaml
- name: Sync dumps to backup server (small dumps)
  ansible.posix.synchronize:
    mode: push
    src: "{{ item.path }}"
    dest: "/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}/"
    rsync_opts:
      - "--inplace"
      - "--perms"
      - "--verbose"
      - "--progress"
      - "--timeout=150"
      - "--no-delay-updates"
    private_key: "~/.ssh/id_rsa"
    archive: no
    checksum: yes
    compress: yes
  delegate_to: "{{ groups['targets'][0] }}"
  loop: "{{ tmp_dumps.files | selectattr('size', 'lt', 1048576) | list }}"
  when: tmp_dumps.matched > 0 and ssh_check is success
  register: sync_results
  ignore_errors: yes
  changed_when: sync_results.rc == 0 or sync_results.rc == 23
  failed_when: sync_results.rc not in [0, 23, 24, 30]
```
- **selectattr('size', 'lt', 1048576)** - Фильтрует файлы меньше 1MB
- **rsync_opts** - Опции для rsync
- **ignore_errors** - Продолжает выполнение при ошибках
- **changed_when/failed_when** - Кастомные условия для определения состояния задачи

#### Копирование дампов на backup сервер (большие файлы)

Аналогично предыдущей задаче, но для файлов >=1MB (`selectattr('size', 'ge', 1048576)`).
