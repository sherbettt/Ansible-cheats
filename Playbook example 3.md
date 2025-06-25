# Подробный анализ Ansible плейбука для резервного копирования PostgreSQL

Этот плейбук выполняет резервное копирование баз данных PostgreSQL на целевых хостах. Разберём его структуру и директивы подробно.

## 1. Основные директивы плейбука

```yaml
- name: To make a backup and dump from pg_db
  hosts: pg_db
  gather_facts: true
```

- **name** - Описание назначения плейбука (создание резервных копий PostgreSQL)
- **hosts: pg_db** - Плейбук выполняется только на хостах из группы pg_db
- **gather_facts: true** - Включение сбора фактов о системе, что необходимо для условных операций (when) в задачах

## 2. Pre-tasks (Предварительные задачи)

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

- **pre_tasks** - Задачи, выполняемые до основных задач
- Первая задача устанавливает python-psycopg2 (Python 2) только для:
  - Debian 10 или
  - Astra Linux
- **tags: install** - Позволяет запускать только эти задачи с тегом `--tags install`

```yaml
  - name: Install python3-psycopg2
    ansible.builtin.apt:
      name: python3-psycopg2
      state: present
    tags:
      - install
    when: ansible_distribution == "Debian" and (ansible_distribution_major_version == "11" or ansible_distribution_major_version == "12")
```

- Вторая задача устанавливает python3-psycopg2 (Python 3) для:
  - Debian 11 или 12
- psycopg2 необходим для работы модулей Ansible с PostgreSQL

## 3. Основные задачи (tasks)

### Проверка пользователя PostgreSQL

```yaml
- name: Check if PostgreSQL user exists
  ansible.builtin.getent:
    database: passwd
    key: postgres
  register: postgres_user_check
  ignore_errors: yes
  changed_when: false
```

- Проверяет существование пользователя 'postgres' через getent
- **register** - сохраняет результат в переменную postgres_user_check
- **ignore_errors: yes** - игнорирует ошибки если пользователя нет
- **changed_when: false** - указывает что задача никогда не меняет состояние системы

```yaml
- name: Debug PostgreSQL user info
  ansible.builtin.debug:
    msg: "PostgreSQL user check result: {{ postgres_user_check }}"
```

- Отладочный вывод результатов проверки пользователя

```yaml
- name: Verify PostgreSQL user exists
  ansible.builtin.debug:
    msg: "PostgreSQL user exists: {{ postgres_user_check is not failed and postgres_user_check.getent_passwd is defined }}"
  when: postgres_user_check is not failed
```

- Проверяет и выводит существует ли пользователь postgres
- Условие **when** выполняется только если предыдущая задача не завершилась ошибкой

### Создание директории для хранения

```yaml
    # ansible -i ~/GIT-projects/backup/inventory/hosts.ini pg_db -m debug -a 'var=inventory_hostname'
    # ansible -i ~/GIT-projects/backup/inventory/hosts.ini pg_db -m debug -a 'var=ansible_date_time.date'
    # check https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html
- name: Create storage directory using current date
  ansible.builtin.file:
    path: "/usr/local/runtel/storage_files/telecoms/runtel.org/{{ inventory_hostname }}/{{ ansible_date_time.date }}"
    state: directory
    mode: '0755'
```

- Создаёт директорию для хранения резервных копий с путём, включающим:
  - Фиксированную часть пути
  - Имя хоста из инвентаря
  - Текущую дату (используется факт ansible_date_time)
- Устанавливает права 0755 (rwxr-xr-x)
    ```bash
    ┌─ root /etc/postgresql/15/main 
    ─ dev70 
    └─ # ll /usr/local/runtel/storage_files/telecoms/runtel.org/192.168.87.70/
    итого 16
    drwxr-xr-x 4 root root 4096 июн 25 07:07 ./
    drwxr-xr-x 3 root root 4096 июн 24 10:04 ../
    drwxr-xr-x 2 root root 4096 июн 24 10:04 2025-06-24/
    drwxr-xr-x 2 root root 4096 июн 25 07:07 2025-06-25/
    ```


### Получение информации о базах данных

```yaml
    # Get database info
        # check https://docs.ansible.com/ansible/latest/collections/community/postgresql/postgresql_info_module.html
        # field filter
- name: Collect PostgreSQL databases
  become: true
  become_user: postgres
  community.postgresql.postgresql_info:
    filter: databases
  register: db_info
  changed_when: false
```

- **become: true** - включает повышение привилегий
- **become_user: postgres** - выполняет задачу от имени пользователя postgres
- Использует модуль postgresql_info для получения списка БД
- **filter: databases** - получает только информацию о базах данных
- Результат сохраняется в переменную db_info

```yaml
- name: Debug database info
  ansible.builtin.debug:
    var: db_info.databases
```

- Отладочный вывод информации о базах данных

### Создание дампов баз данных

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

- Ключевая задача создания дампов:
  - **become_method: su** - использует su для смены пользователя (вместо sudo по умолчанию)
  - Для каждой БД (loop) создаётся сжатый дамп в /tmp/
  - Имя файла включает имя БД и текущую дату
  - Условия выполнения (**when**):
    - Пользователь postgres существует
    - Получена информация о БД
    - Список БД не пуст
  - **loop_control.label** - улучшает вывод, показывая только имя БД вместо всего объекта

### Проверка созданных дампов

```yaml
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

- Поиск созданных файлов дампов в /tmp/
- Вывод списка найденных файлов для проверки

## Почему такие сочетания директив?

1. **Разделение pre_tasks и tasks**:
   - Предварительная установка зависимостей (psycopg2) выполняется до основных задач
   - Это гарантирует что все зависимости будут на месте перед работой с PostgreSQL

2. **Условные выполнения (when)**:
   - Разные пакеты для разных ОС и версий
   - Проверка существования пользователя postgres перед выполнением задач
   - Проверка что есть БД для резервного копирования

3. **Работа с привилегиями**:
   - Задачи связанные с PostgreSQL выполняются от имени пользователя postgres
   - Использование su (become_method) вместо sudo может быть требованием безопасности

4. **Организация путей**:
   - Директории создаются с иерархией включающей имя хоста и дату
   - Временные файлы создаются в /tmp/, что является стандартной практикой

5. **Отладочная информация**:
   - Многочисленные debug задачи помогают в диагностике проблем
   - Регистрация результатов ключевых проверок (register)

Такой подход обеспечивает:
- Идемпотентность (многократное выполнение даёт одинаковый результат)
- Безопасность (правильные права и пользователи)
- Переносимость (работа на разных ОС и версиях)
- Надёжность (проверки перед критическими операциями)
