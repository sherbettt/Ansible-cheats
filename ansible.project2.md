Управляющая машина `192.168.87.8`, на которую установлен semaphore.
<br/> Управляемая машина `192.168.87.70`, на которую уже установлена БД PostgresQL 15.
<br/> Ansible конфигурация хранится в GitLab.
<br/> Редактирование кода идёт на `192.168.87.136`.

На управляемых машинах требуется провести действия:
  - создать dump базы данных и сжать в формате .gz;
  - переслать полученный .gz на `192.168.87.136`, а после - на SFTP;
-----------------------------------------------------------------------

### 1. Для начала зайдём в БД и выясним имя самой БД
```
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
```






