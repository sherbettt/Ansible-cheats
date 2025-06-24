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
┌─ root /etc/postgresql/15/main 
─ dev70 
└─ # su - postgres 
postgres@dev70:~$ whoami 
postgres
postgres@dev70:~$ psql 
psql (15.13 (Debian 15.13-0+deb12u1))
Введите "help", чтобы получить справку.

postgres=# SELECT datname, datdba, encoding, datcollate, datctype FROM pg_database;
      datname      | datdba | encoding | datcollate  |  datctype   
-------------------+--------+----------+-------------+-------------
 postgres          |     10 |        6 | ru_RU.UTF-8 | ru_RU.UTF-8
 rt_pbx_v2         |  16384 |        6 | ru_RU.UTF-8 | ru_RU.UTF-8
 template1         |     10 |        6 | ru_RU.UTF-8 | ru_RU.UTF-8
 template0         |     10 |        6 | ru_RU.UTF-8 | ru_RU.UTF-8
 rt_pbx_v2_media   |  16384 |        6 | ru_RU.UTF-8 | ru_RU.UTF-8
 rt_pbx_v2_stat    |  16384 |        6 | ru_RU.UTF-8 | ru_RU.UTF-8
 rt_pbx_v2_logging |  16384 |        6 | ru_RU.UTF-8 | ru_RU.UTF-8
(7 строк)

postgres=# \l
                                                      Список баз данных
        Имя        | Владелец | Кодировка | LC_COLLATE  |  LC_CTYPE   | локаль ICU | Провайдер локали |     Права доступа     
-------------------+----------+-----------+-------------+-------------+------------+------------------+-----------------------
 postgres          | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | 
 rt_pbx_v2         | rt_pbx   | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | 
 rt_pbx_v2_logging | rt_pbx   | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | 
 rt_pbx_v2_media   | rt_pbx   | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | 
 rt_pbx_v2_stat    | rt_pbx   | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | 
 template0         | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | =c/postgres          +
                   |          |           |             |             |            |                  | postgres=CTc/postgres
 template1         | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | =c/postgres          +
                   |          |           |             |             |            |                  | postgres=CTc/postgres
(7 строк)
```

```c
postgres@dev70:~$ psql -U postgres -d template1
psql (15.13 (Debian 15.13-0+deb12u1))
```

### 2. Список ролей в базе данных PostgreSQL вместе с соответствующими привилегиями.
```c
template1=# \du
                                          Список ролей
 Имя роли |                                Атрибуты                                 | Член ролей 
----------+-------------------------------------------------------------------------+------------
 postgres | Суперпользователь, Создаёт роли, Создаёт БД, Репликация, Пропускать RLS | {}
 rt_pbx   |                                                                         | {}
```
-----------------------------------------------------------------------

## 2. Начнём писать проект Ansible.

### 1. Создаём структуру проекта на `192.168.87.136`/GitLab.
<pre style="color: #00ff00; background-color: #000000;">
┌─ kirill ~/GIT-projects/backup 
└─ $ treel -Cs
[kirill   kirill           118 Jun 24 11:44]  .
├── [kirill   kirill           503 Jun 24 13:15]  ansible.cfg
├── [kirill   kirill             0 Jun 24 11:44]  facts_cache
├── [kirill   kirill             0 Jun 24 11:30]  group_vars
├── [kirill   kirill            18 Jun 24 11:35]  inventory
│   └── [kirill   kirill            21 Jun 24 13:13]  hosts.ini
├── [kirill   kirill            50 Jun 24 12:02]  playbooks
│   ├── [kirill   kirill          2016 Jun 24 16:07]  dump_play.yml
│   └── [kirill   kirill          3389 Jun 24 12:02]  playbook.yml
└── [kirill   kirill             0 Jun 24 11:30]  roles
</pre>

#### `ansible.cfg`
```cfg
[defaults]
inventory = ./inventory/hosts.ini
remote_user = postgres
log_path = /var/log/ansible.log
forks = 1
gathering = smart
#timeout = 10  # SSH timeout

# Caching of facts
fact_caching = jsonfile
fact_caching_connection = ./facts_cache
fact_caching_timeout = 0

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

#[ssh_connection]
#host_key_checking = true
## This condition will be applied for all machines
#ssh_args = -o BindAddress=192.168.56.1
```

