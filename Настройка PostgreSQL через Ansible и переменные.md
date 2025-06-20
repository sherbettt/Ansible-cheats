Для файла `postgresql.conf.j2` можно добавить несколько важных параметров PostgreSQL, которые часто настраиваются в зависимости от нагрузки и характеристик сервера.  

### **Рекомендуемые переменные для добавления:**
1. **Основные настройки памяти и процессов:**
   ```jinja2
   max_connections = {{ postgresql_max_connections }}
   listen_addresses = '{{ postgresql_listen_addresses }}'
   port = {{ postgresql_port }}
   ```

2. **Настройки Write-Ahead Log (WAL):**
   ```jinja2
   wal_level = '{{ postgresql_wal_level }}'
   synchronous_commit = {{ postgresql_synchronous_commit }}
   checkpoint_timeout = '{{ postgresql_checkpoint_timeout }}'
   checkpoint_completion_target = {{ postgresql_checkpoint_completion_target }}
   ```

3. **Настройки производительности:**
   ```jinja2
   random_page_cost = {{ postgresql_random_page_cost }}
   effective_io_concurrency = {{ postgresql_effective_io_concurrency }}
   max_worker_processes = {{ postgresql_max_worker_processes }}
   max_parallel_workers_per_gather = {{ postgresql_max_parallel_workers_per_gather }}
   ```

4. **Логирование:**
   ```jinja2
   log_destination = '{{ postgresql_log_destination }}'
   logging_collector = {{ postgresql_logging_collector }}
   log_directory = '{{ postgresql_log_directory }}'
   log_filename = '{{ postgresql_log_filename }}'
   log_rotation_age = '{{ postgresql_log_rotation_age }}'
   log_rotation_size = '{{ postgresql_log_rotation_size }}'
   ```

5. **Аутентификация:**
   ```jinja2
   password_encryption = '{{ postgresql_password_encryption }}'
   ```

---

### **Где брать значения переменных?**
1. **Официальная документация PostgreSQL:**  
   - [PostgreSQL Tuning](https://www.postgresql.org/docs/current/runtime-config-resource.html)  
   - [PostgreSQL Configuration](https://www.postgresql.org/docs/current/runtime-config.html)  

2. **Рекомендации по настройке в зависимости от ОЗУ и диска:**  
   - **`shared_buffers`** – обычно 25% от RAM (но не более 8-16 ГБ).  
   - **`effective_cache_size`** – ~75% от RAM.  
   - **`work_mem`** – `(RAM - shared_buffers) / (max_connections * 2)`.  
   - **`maintenance_work_mem`** – до 2 ГБ для серверов с большими таблицами.  

3. **Пример значений для сервера с 16 ГБ RAM:**
   ```yaml
   postgresql_ram_shared: "4GB"
   postgresql_ram_cache: "12GB"
   postgresql_work_mem: "32MB"
   postgresql_maintenance_work_mem: "1GB"
   postgresql_max_connections: 100
   postgresql_wal_level: "replica"
   postgresql_checkpoint_timeout: "15min"
   postgresql_checkpoint_completion_target: 0.9
   postgresql_random_page_cost: 1.1  # для SSD
   postgresql_effective_io_concurrency: 200  # для NVMe/SSD
   ```

4. **Автоматические калькуляторы:**  
   - [PGTune](https://pgtune.leopard.in.ua/) – помогает рассчитать параметры на основе характеристик сервера.  
   - [PGConfig](https://pgconfig.org/) – генерирует конфиг под заданные параметры.  

---

### **Где хранить переменные?**
- В `group_vars/` для разных групп серверов (например, `group_vars/postgresql_servers.yml`).  
- В `host_vars/` для специфичных настроек под конкретный сервер.  
- В роли (если используется Ansible Role) – `defaults/main.yml` или `vars/main.yml`.  

Пример структуры:
```yaml
# group_vars/postgresql_servers.yml
postgresql_port: 5432
postgresql_listen_addresses: "0.0.0.0"
postgresql_max_connections: 200
postgresql_wal_level: "replica"
postgresql_password_encryption: "scram-sha-256"
```


