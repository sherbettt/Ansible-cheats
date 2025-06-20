Следующий плейбук для PSQL идёт вместе с шаблоном j2 и переменными.

```yaml
# ~/.ansible/project1/playbooks/05_psql-conf.yml
####

---
- name: Configure PostgreSQL using template
  hosts: clients
  become: true
  
  # vars:
  #   - pg_config_cmd: "pg_config --version"
  #   - pg_etc_path: "/etc/postgresql"
  vars_files:
    - /root/.ansible/project1/group_vars/clients/psql_var.yml

  tasks:

    - name: Verify required variables are set
      ansible.builtin.assert:
        that:
          - pg_config_cmd is defined
          - pg_etc_path is defined
          - postgresql_ram_shared is defined
          - postgresql_ram_cache is defined

    - name: Check Postgresql version after update
      ansible.builtin.command: "{{ pg_config_cmd }}"
      register: pg_version
      changed_when: false

    - name: Display Postgresql version
      ansible.builtin.debug:
        var: pg_version.stdout

    - name: Get PostgreSQL version (short form)
      ansible.builtin.set_fact:
        postgresql_version: "{{ pg_version.stdout.split('.')[0] | regex_replace('[^0-9]', '') }}"

    - name: Calculate PostgreSQL memory settings
      ansible.builtin.set_fact:
        postgresql_ram_shared: "{{ (ansible_memtotal_mb * 0.25) | int }}MB"

    - name: Copy postgresql.conf with templating
      ansible.builtin.template:
        src: /root/.ansible/project1/templates/postgresql.conf.j2
        dest: "{{ pg_etc_path }}/{{ postgresql_version }}/main/postgresql.conf"
        owner: postgres
        group: postgres
        mode: '0644'
      notify: restart postgresql

    - name: Ensure PSQL is running
      ansible.builtin.service:
        name: postgresql
        state: started
        enabled: true

  handlers:
    - name: restart postgresql
      ansible.builtin.service:
        name: postgresql
        state: restarted
```

Template:
```yaml
# ~/.ansible/project1/templates/postgresql.conf.j2
####

# postgresql.conf.j2
shared_buffers = {{ postgresql_ram_shared }}
effective_cache_size = {{ postgresql_ram_cache }}
work_mem = {{ postgresql_work_mem }}
maintenance_work_mem = {{ postgresql_maintenance_work_mem }}
```

переменные:
```yaml
# ~/.ansible/project1/group_vars/clients/psql_var.yml
#####

# /root/.ansible/project1/group_vars/clients/psql_var.yml
pg_config_cmd: "pg_config --version"
pg_etc_path: "/etc/postgresql"

# Memory settings (25% of total RAM for shared_buffers, 15% for cache)
postgresql_ram_shared: "{{ (ansible_memtotal_mb * 0.25) | int }}MB"
postgresql_ram_cache: "{{ (ansible_memtotal_mb * 0.15) | int }}MB"

# ÐÑÑÐ³Ð¸Ðµ ÑÐµÐºÐ¾Ð¼ÐµÐ½Ð´ÑÐµÐ¼ÑÐµ Ð¿Ð°ÑÐ°Ð¼ÐµÑÑÑ
postgresql_work_mem: "64MB"
postgresql_maintenance_work_mem: "256MB"
postgresql_effective_cache_size: "{{ (ansible_memtotal_mb * 0.6) | int }}MB"
```


### Причина ошибок:
1. В одном из файлов (скорее всего в `psql_var.yml` или шаблоне) содержатся символы, которые не могут быть корректно обработаны в UTF-8
2. Возможно, файл был сохранён в неправильной кодировке или содержит специальные символы

### Решение:

1. **Проверьте файл переменных**:
```bash
# Проверим файл на наличие нестандартных символов
cat -v /root/.ansible/project1/group_vars/clients/psql_var.yml

# Или с помощью hexdump для бинарного просмотра
hexdump -C /root/.ansible/project1/group_vars/clients/psql_var.yml | head -20
```

2. **Пересохраните файл в правильной кодировке**:
```bash
# Создадим резервную копию
cp /root/.ansible/project1/group_vars/clients/psql_var.yml /root/.ansible/project1/group_vars/clients/psql_var.yml.bak

# Пересохраним файл в UTF-8
iconv -f ISO-8859-1 -t UTF-8 /root/.ansible/project1/group_vars/clients/psql_var.yml -o /root/.ansible/project1/group_vars/clients/psql_var.yml.utf8
mv /root/.ansible/project1/group_vars/clients/psql_var.yml.utf8 /root/.ansible/project1/group_vars/clients/psql_var.yml
```

3. **Проверьте шаблон postgresql.conf.j2**:
```bash
file -i /root/.ansible/project1/templates/postgresql.conf.j2
```

4. **Временное решение для диагностики**:
Создайте минимальный тестовый vars-файл:
```bash
echo -e "pg_config_cmd: \"pg_config --version\"\npg_etc_path: \"/etc/postgresql\"" > /root/.ansible/project1/group_vars/clients/test_vars.yml
```
И измените playbook на использование этого файла:
```yaml
vars_files:
  - /root/.ansible/project1/group_vars/clients/test_vars.yml
```

5. **Если проблема в шаблоне**, попробуйте:
```bash
# Удалите BOM (Byte Order Mark) если есть
sed -i '1s/^\xEF\xBB\xBF//' /root/.ansible/project1/templates/postgresql.conf.j2

# Или пересоздайте шаблон
cp /root/.ansible/project1/templates/postgresql.conf.j2 /root/.ansible/project1/templates/postgresql.conf.j2.bak
echo "# Minimal template" > /root/.ansible/project1/templates/postgresql.conf.j2
```

6. **Дополнительные проверки**:
```bash
# Проверьте кодировку всех файлов
find /root/.ansible/project1 -type f -name "*.yml" -o -name "*.j2" | xargs file -i

# Проверьте на наличие не-ASCII символов
grep -nRP "[^\x00-\x7F]" /root/.ansible/project1/{group_vars,playbooks,templates}
```

После выполнения этих шагов попробуйте снова запустить playbook. Если ошибка сохраняется, попробуйте:
```bash
LC_ALL=C.UTF-8 ansible-playbook 05_psql-conf.yml -vvv
```

Эта ошибка обычно возникает при:
- Наличии специальных символов в файлах
- Неправильной кодировке файлов
- Повреждённых символах в файлах
- Проблемах с локалью системы


