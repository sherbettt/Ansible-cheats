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



# Подробный разбор Ansible плейбука для резервного копирования PostgreSQL

Этот плейбук предназначен для создания резервных копий (дампа) баз данных PostgreSQL и их передачи на backup-сервер. Разберём его построчно:

## Общая структура

```yaml
- name: To make a backup and dump from pg_db
  hosts: pg_db
  gather_facts: true
```

- **name** - Описание назначения плейбука
- **hosts: pg_db** - Применяется к группе хостов pg_db из inventory
- **gather_facts: true** - Собирает информацию о целевых хостах (ОС, версия и т.д.)

## Предварительные задачи (pre_tasks)

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

- Устанавливает python-psycopg2 (Python 2) для Debian 10 или Astra Linux
- **tags: install** - Позволяет запускать только эту задачу с тегом
- **when** - Условие для выполнение на определённых ОС

```yaml
  - name: Install python3-psycopg2
    ansible.builtin.apt:
      name: python3-psycopg2
      state: present
    tags:
      - install
    when: ansible_distribution == "Debian" and (ansible_distribution_major_version == "11" or ansible_distribution_major_version == "12")
```

- Аналогично устанавливает python3-psycopg2 для Debian 11/12

## Основные задачи (tasks)

### Создание директории для хранения

```yaml
- name: Create storage directory using current date
  ansible.builtin.file:
    path: "/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}"
    state: directory
    mode: '0755'
```

- Создаёт директорию с именем текущей даты на целевом хосте
- Использует переменные:
  - `inventory_hostname` - имя хоста из inventory
  - `ansible_date_time.date` - текущая дата

### Получение информации о БД

```yaml
- name: Gather information about PostgreSQL databases only from servers
  become: true
  become_user: postgres
  community.postgresql.postgresql_info:
    filter: databases
  register: db_info
  changed_when: false
```

- Получает информацию о базах данных PostgreSQL
- **become: true** - Повышение привилегий
- **become_user: postgres** - Выполнение от пользователя postgres
- **filter: databases** - Получает только информацию о БД
- **register: db_info** - Сохраняет результат в переменную db_info
- **changed_when: false** - Всегда отмечает задачу как неизменённую

### Отладка информации о БД

```yaml
- name: Display/Debug info (db_info.databases.postgres.extensions.plpgsql)
  ansible.builtin.debug:
    var: db_info.databases.postgres.extensions.plpgsql

- name: Display/Debug all database info (db_info.databases)
  ansible.builtin.debug:
    var: db_info.databases
```

- Вывод отладочной информации о БД (для проверки)

### Создание дампов БД

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

- Создаёт сжатые дампы всех БД
- **loop** - Итерируется по всем именам БД из db_info
- **target** - Сохраняет в /tmp с именем БД и датой
- **loop_control.label** - Упрощает вывод, показывая только имя БД вместо полных данных

### Поиск созданных дампов

```yaml
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
  when: tmp_dumps.matched > 0

- name: Display number of found dump files
  ansible.builtin.debug:
    msg: "кол-во выводимых в /tmp/ *.sql.gz: {{ tmp_dumps.matched }}"
```

- Находит все созданные дампы в /tmp/
- Выводит информацию о найденных файлах (путь, размер, права)
- Показывает количество найденных файлов

### Подготовка к копированию на backup-сервер

```yaml
- name: Ensure destination directory exists on backup server
  delegate_to: 192.168.87.99
  ansible.builtin.file:
    path: "/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}"
    state: directory
    mode: '0755'
```

- Создаёт целевую директорию на backup-сервере (192.168.87.99)
- **delegate_to** - Выполняет задачу на указанном хосте

```yaml
- name: Check SSH connection to backup server
  ansible.builtin.ping:
  delegate_to: "{{ groups['targets'][0] }}"
  register: ssh_check
  
- name: Fail if SSH check failed
  fail:
    msg: "Cannot connect to backup server via SSH"
  when: ssh_check is failed
```

- Проверяет SSH-соединение с backup-сервером
- В случае ошибки прерывает выполнение

### Копирование дампов на backup-сервер

Копирование разделено на две задачи для файлов разного размера:

#### Для маленьких файлов (<1MB)

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

#### Для больших файлов (>=1MB)

Аналогичная задача, но с фильтром `selectattr('size', 'ge', 1048576)`

Ключевые параметры:
- **mode: push** - Копирование с локального хоста на удалённый
- **rsync_opts** - Опции rsync для контроля процесса копирования
- **private_key** - Использует SSH-ключ для аутентификации
- **loop** с фильтром selectattr - Разделяет файлы по размеру
- **ignore_errors: yes** - Продолжает выполнение при ошибках
- **changed_when/failed_when** - Кастомные условия для определения состояния задачи

## Особенности реализации

1. **Разделение по размеру файлов**: Маленькие и большие файлы копируются отдельно, что позволяет лучше контролировать процесс.

2. **Гибкая обработка ошибок**: Определены допустимые коды возврата rsync (0, 23, 24, 30), которые не считаются ошибками.

3. **Использование переменных**: Широко применяются факты Ansible (информация о хосте, дата) для создания уникальных путей.

4. **Безопасность**: Все операции с БД выполняются от пользователя postgres.

5. **Модульность**: Задачи разделены на логические блоки с проверкой промежуточных результатов.

Этот плейбук обеспечивает надёжное резервное копирование PostgreSQL с детальным контролем процесса и обработкой возможных ошибок.
