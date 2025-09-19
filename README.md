

---------------
## Ansible Variables Documentation

**Официальная документация Ansible:**
- 📖 [Ansible Variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html) - полное руководство по переменным
- 📖 [Ansible Facts](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html) - системные переменные (facts)
- 📖 [Special Variables](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html) - специальные переменные

## Semaphore Variables Documentation

**Для Semaphore переменных окружения:**
- 📖 [Semaphore Environment Variables](https://docs.ansible-semaphore.com/administration-guide/environment-variables) - официальная документация
- 📖 [Semaphore CI/CD Variables](https://docs.semaphoreui.com/user-guide/environment/) - переменные в проектах

## Практическая шпаргалка по переменным:

### **1. Ansible Facts (системные переменные)**
```yaml
# Системная информация
{{ ansible_distribution }}          # "Debian", "Ubuntu", "CentOS"
{{ ansible_distribution_version }}  # "11", "20.04", "7"
{{ ansible_os_family }}             # "Debian", "RedHat"

# Сетевая информация  
{{ ansible_default_ipv4.address }}  # основной IP адрес
{{ ansible_hostname }}              # имя хоста
{{ ansible_fqdn }}                  # полное доменное имя

# Аппаратная информация
{{ ansible_memory_mb.real.total }}  # общая память в MB
{{ ansible_processor_count }}       # количество процессоров
{{ ansible_processor_cores }}       # количество ядер

# Дата и время
{{ ansible_date_time.date }}        # "2024-01-15"
{{ ansible_date_time.time }}        # "14:30:25"
{{ ansible_date_time.iso8601 }}     # "2024-01-15T14:30:25Z"
```

### **2. Inventory переменные**
```yaml
# Из inventory файла
{{ inventory_hostname }}            # имя хоста из inventory
{{ groups }}                        # все группы
{{ groups['pg_db'] }}               # хосты в группе pg_db

# Переменные хоста
{{ hostvars[inventory_hostname].postgresql_host }}
{{ hostvars[inventory_hostname].postgresql_port }}
```

### **3. Semaphore CI/CD переменные**
```yaml
# Информация о задаче
{{ lookup('env', 'SEMAPHORE_JOB_ID') }}
{{ lookup('env', 'SEMAPHORE_JOB_NAME') }}

# Информация о проекте  
{{ lookup('env', 'SEMAPHORE_PROJECT_ID') }}
{{ lookup('env', 'SEMAPHORE_PROJECT_NAME') }}

# Git информация
{{ lookup('env', 'SEMAPHORE_GIT_BRANCH') }}
{{ lookup('env', 'SEMAPHORE_GIT_COMMIT') }}
{{ lookup('env', 'SEMAPHORE_GIT_REPO_SLUG') }}

# Workflow информация
{{ lookup('env', 'SEMAPHORE_WORKFLOW_ID') }}
{{ lookup('env', 'SEMAPHORE_WORKFLOW_NAME') }}
```

### **4. Переменные окружения**
```yaml
# Пользовательские переменные
{{ lookup('env', 'BACKUP_RETENTION_DAYS') }}
{{ lookup('env', 'DB_PASSWORD') }}

# Системные переменные
  tasks:
    - name: System variables
      debug:
        msg: 
        - "{{ lookup('env', 'PWD') }}"
        - "{{ lookup('env', 'HOME') }}"
        - "{{ lookup('env', 'PATH') }}"
        - "{{ lookup('env', 'LC_CTYPE') }}"
        - "{{ lookup('env', 'USER') }}"
```

### **5. Специальные переменные Ansible**
```yaml
{{ play_hosts }}                    # все хосты в текущем play
{{ inventory_dir }}                 # путь к директории inventory
{{ playbook_dir }}                  # путь к директории playbook

# Информация о текущей задаче
{{ ansible_play_name }}             # имя текущего play
{{ ansible_play_hosts }}            # хосты в текущем play
```

## Как исследовать доступные переменные:

**В самом playbook:**
```yaml
- name: Display all known variables
  debug:
    var: hostvars[inventory_hostname]
  when: false  # поменяйте на true для отладки

- name: Display only variable names
  debug:
    msg: "{{ item }}"
  loop: "{{ hostvars[inventory_hostname].keys() | list }}"
  when: false
```

**Через командную строку:**
```bash
# Все facts для хоста
ansible -i inventory/hosts.ini your_host -m setup

# Конкретная переменная
ansible -i inventory/hosts.ini your_host -m debug -a "var=ansible_distribution"

# Все inventory переменные
ansible-inventory -i inventory/hosts.ini --list --yaml
```
---------------
Сбор фактов об управляемой машине: [Discovering variables: facts and magic variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html)

Вместе с модулями смотри [Common Return Values](https://docs.ansible.com/ansible/latest/reference_appendices/common_return_values.html)

Базовые страницы для поиска модулей в [ansible-doc](https://docs.ansible.com/ansible/latest/cli/ansible-doc.html):
<br/> [Playbooks Delegation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_delegation.html)
<br/> [Indexes of all modules and plugins](https://docs.ansible.com/ansible/latest/collections/all_plugins.html)
<br/> [Collection Index](https://docs.ansible.com/ansible/latest/collections/):
- [ansible.builtin modules](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/)
- [Ansible.Netcommon modules](https://docs.ansible.com/ansible/latest/collections/ansible/netcommon/index.html)
- [Ansible.Posix modules](https://docs.ansible.com/ansible/latest/collections/ansible/posix/index.html)
- [Community.Postgresql modules](https://docs.ansible.com/ansible/latest/collections/community/postgresql/index.html)

<br/> [Playbook Keywords: Play](https://docs.ansible.com/ansible/latest/reference_appendices/playbooks_keywords.html#play) ,
<br/> [Playbook Keywords: Task](https://docs.ansible.com/ansible/latest/reference_appendices/playbooks_keywords.html#task) ,
<br/> [Using Ansible playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/) ,

<br/> [Templating (Jinja2)](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html) -> [Jinja2 Docs](https://jinja.palletsprojects.com/en/latest/templates/)
 
 [YAMLSyntax](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)
<br/> [Using Ansible playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/index.html)
<br/> [Building Ansible inventories](https://docs.ansible.com/ansible/latest/inventory_guide/index.html)
<br/> [Ansible CLI cheatsheet](https://docs.ansible.com/ansible/latest/command_guide/cheatsheet.html)

---------------
[Примеры работы с Ansible](https://www.dmosk.ru/miniinstruktions.php?mini=ansible-examples)
<br/> ещё одна [Шпаргалка по Ansible](https://github.com/horv1tz/useful/blob/main/DevOps/Ansible.md)

Данный проект на GIT можно использовать как роль: [palkan-ansible/postgresql/](https://github.com/palkan-ansible/postgresql/tree/master)
