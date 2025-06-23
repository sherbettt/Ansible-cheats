Есть Ubuntu подобный роутер. На нём три интерфейса:
<br/> eth0 - `192.168.87.112` (Главная Управляющая Ansible машина (смотрит в интернет)), шлюз: `192.168.87.1/24`;
<br/> eth1 - `192.168.56.1` (не смотрит в интернет, для связи с локальными машинами);
<br/> eth2 - `192.168.96.113` (смотрит в интернет), шлюз: `192.168.96.1/24`
<br/> Имеется дополнительно управляющая Ansible машина: `192.168.87.136/24`.

Также есть две другие Ubuntu управляемые машины, подключённые к выше указанном роутеру с адресами: `192.168.56.2/24`; `192.168.56.3/24`. Аутентификация на машины по ключу root'а.

На управляемых машинах требуется провести действия:
  - собрать факты об удалённых узлах;
  - установить python (разные версии для теста);
  - установить postgreql;
  - создать template postgresql.j2
  - 
-----------------------------------------------------------------------

## Ansible на роутере.

### 1. Перед созданием структуры требуется проверить соединение по SSH.
Так как аутентификация по ключу уже настроена, убедитесь, что:
- На роутере (`192.168.87.112`) есть приватный ключ (`~/.ssh/id_rsa`).
- Публичный ключ (`~/.ssh/id_rsa.pub`) добавлен в `~/.ssh/authorized_keys` на управляемых машинах (`192.168.56.2` и `192.168.56.3`).

Проверить подключение:
```bash
ssh -i ~/.ssh/id_rsa root@192.168.56.2
ssh -i ~/.ssh/id_rsa root@192.168.56.3
```

На всякий случай роверить `sshd_config` на `test-gw` и других машинах.
```bash
grep -E "PermitRootLogin|PasswordAuthentication" /etc/ssh/sshd_config
```
И выполнить:
 - в иделае поменять параметр на `PermitRootLogin yes` + `systemctl restart sshd`;
 - либо создать отдельного пользователя для Ansible (например, `ansible`) и настроить sudo без пароля.



### 2. Создадим структуру проекта:
Для начала рассмотрим управление с роутера (`192.168.87.112/24`), установив на него предварительно Ansible.
Структура проекта будет выглядеть примерно так на начальном этапе:
```
~/.ansible/project1/
.
|-- ansible.cfg          # Конф. файл
|-- facts_cache          # кеш фактов
|-- group_vars
|   `-- all.yml          # основные переменные
|-- inventory
|   `-- hosts.ini        # инвентарный файл
|-- playbooks
|   `-- site.yml         # основной плейбук
`-- roles                # директория с ролями
```

```bash
mkdir -p ~/.ansible/project1/{inventory,group_vars,roles,playbooks}
cd ~/.ansible/project
```

 **Файл **`~/.ansible/project1/inventory/hosts.ini`**:**
 <br/> [Run tasks using libssh for ssh connection](https://docs.ansible.com/ansible/latest/collections/ansible/netcommon/libssh_connection.html#ansible-collections-ansible-netcommon-libssh-connection)
 <br/> [connect via SSH client binary](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ssh_connection.html#ansible-collections-ansible-builtin-ssh-connection)
 <br/> [Переменная ansible_ssh_common_args](https://github.com/sherbettt/Ansible-cheats/blob/main/Переменная%20ansible_ssh_common_args.md)
```ini
[masters]
# check group_vars/masters/ssh.yml
test-gw ansible_host=192.168.87.112 ansible_user=root  # лучше использовать ansible, создав на клиентских машинах
#test-gw ansible_host=192.168.87.112 ansible_user=root ansible_ssh_common_args=''
kiko0217 ansible_host=192.168.87.136 ansible_user=root

#[clients]
#test-lan ansible_host=192.168.56.2 ansible_user=root
#test-lan2 ansible_host=192.168.56.3 ansible_user=root

# To allow route through eth1 192.168.56.1 
[clients]
test-lan ansible_host=192.168.56.2 ansible_user=root ansible_ssh_common_args='-o BindAddress=192.168.56.1'
test-lan2 ansible_host=192.168.56.3 ansible_user=root ansible_ssh_common_args='-o BindAddress=192.168.56.1'

[all:children]
masters
clients
```
где:
1. Секция `[masters]` содержит хосты управляющих машиг с указанием их IP-адреса и ansible-пользователя
2. Секция `[clients]` содержит два управляемых сервера с их IP-адресами
3. Секция `[all:children]` объединяет группы masters и clients в одну группу all

В вашем inventory-файле для клиентских машин (`test-lan` и `test-lan2`) используется параметр **`ansible_ssh_common_args`** с опцией **`-o BindAddress=192.168.56.1`**. 

Это параметр Ansible, который позволяет передавать дополнительные аргументы в SSH-клиент при подключении к удалённым хостам.  
Формат: `ansible_ssh_common_args='ДОПОЛНИТЕЛЬНЫЕ_SSH_ФЛАГИ' `

В нашем случае:
```ini
ansible_ssh_common_args='-o BindAddress=192.168.56.1'
```
Это означает, что Ansible при подключении к `test-lan` и `test-lan2` будет использовать SSH с опцией **`BindAddress`**, указывая, через какой локальный интерфейс (IP-адрес) устанавливать соединение.

 Роутер (`test-gw`) имеет несколько интерфейсов:
- **`eth0`** (`192.168.87.112`) — смотрит в интернет (управляющая машина Ansible).
- **`eth1`** (`192.168.56.1`) — внутренняя сеть (LAN, где находятся `test-lan` и `test-lan2`).
- **`eth2`** (`192.168.96.113`) — альтернативный интернет-интерфейс.

#### **Проблема:**
- Если Ansible попытается подключиться к `192.168.56.2` (`test-lan`) **через `eth0` (`192.168.87.112`)**, то:
  - Пакеты могут уходить не туда (например, через дефолтный шлюз `192.168.87.1`).
  - Возможно, фаервол или маршрутизация не пропустят такой трафик.
  - Подключение может не работать или идти через неправильный интерфейс.

#### **Решение:**
- **`BindAddress=192.168.56.1`** заставляет SSH использовать **только интерфейс `eth1` (`192.168.56.1`)** для подключения к `test-lan` (`192.168.56.2`).  
  Это гарантирует, что:
  - Подключение идёт через правильную сеть (`192.168.56.0/24`).
  - Нет конфликтов с другими интерфейсами.
  - Если на роутере есть правила фаервола или NAT, они корректно обработают трафик.

### Это требуется когда:
- **Если у роутера несколько интерфейсов**, и нужно явно указать, через какой из них подключаться.
- **Если маршрутизация по умолчанию ведёт не туда** (например, дефолтный шлюз `192.168.87.1`, а нужно идти через `192.168.56.1`).
- **Если фаервол или политики сети требуют строгого контроля исходящих интерфейсов**.

-----

#### **Альтернативные решения с подключением по определённому интерфейсу**
Если не хочется прописывать `ansible_ssh_common_args` для каждого хоста, можно:

  **Вариант 1. Указать в `ansible.cfg`**
```ini
[ssh_connection]
ssh_args = -o BindAddress=192.168.56.1
```
Но это повлияет на **все** SSH-подключения Ansible (может быть нежелательно).

  **Вариант 2. Исправить маршрутизацию на роутере**
Настроить `ip route`, чтобы трафик до `192.168.56.0/24` шёл через `eth1`:
```bash
ip route add 192.168.56.0/24 dev eth1
```
Тогда `BindAddress` может не понадобиться.

----

См.: [Описание и пример ansible.cfg](https://github.com/sherbettt/Ansible-cheats/blob/main/03.%20Описание%20и%20пример%20ansible.cfg.md), 
[Условие when](https://github.com/sherbettt/Ansible-cheats/blob/main/50.%20Условие%20when.md)

**Файл **`ansible.cfg`**:**
```ini
[defaults]
inventory = ./inventory/hosts.ini
remote_user = root  # there is no ansible user
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

[ssh_connection]
host_key_checking = true
# This condition will be applied for all machines
ssh_args = -o BindAddress=192.168.56.1
```
Проверка параметров командой: `ansible-config dump --only-changed`
<br/> Проверка всех машин из инвенторки: `ansible all -m ping -vvv` и `ansible clients -m ping -vv`

----

Проверка соединения локальных машин Ansible'ом и классически по SSH:
```ini
┌─ root ~/.ansible/project1 
─ test-gw 
└─ # ansible test-lan -m ping
test-lan | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

```ini
┌─ root ~/.ansible/project1 
─ test-gw 
└─ # ssh root@192.168.56.2 -o BindAddress=192.168.56.1
Linux test-lan 6.8.12-9-pve #1 SMP PREEMPT_DYNAMIC PMX 6.8.12-9 (2025-03-16T19:18Z) x86_64
....
Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu Jun 19 10:39:09 2025 from 192.168.56.1
```

```ini
┌─ root ~/.ansible/project1 
─ test-gw 
└─ # pcat facts_cache/test-lan
{
    "discovered_interpreter_python": "/usr/bin/python3"
}
```

-----

**Файлы **переменных в `~/.ansible/project1/group_vars/..`**:**
<br/> С учётом специфики оборудования и применения SSH соединения с локальной сетю, в которой управляющие машины, было создано исключение для группы masters в файле `group_vars/masters/ssh.yml`. 
Важно отметить, что публичные ключи ещё не раздавались между 87.112 и 87.136, т.е. будет закномерная ошибка в случае пинга, но пока что оставим эту пробелму, т.к. управлять нжно машинами 56.2 и 56.3.
```ini
┌─ root ~/.ansible/project1 
─ test-gw 
└─ # pcat group_vars/masters/ssh.yml 
---
ansible_ssh_common_args: ''
```

### 3. Создание playbook'а
Итак, сейчас структура изменилась и выглядит по-другому:
```
~/.ansible/project1/
.
|-- ansible.cfg
|-- facts_cache
|   |-- test-lan
|   `-- test-lan2
|-- group_vars
|   |-- all.yml
|   `-- masters
|       `-- ssh.yml
|-- inventory
|   `-- hosts.ini
|-- playbooks
|   |-- install.yml
|   `-- ping.yml
`-- roles
```

#### Создадим простой плейбук для пинга клиентских машин
Не забывайте про двойные отступы.
<br/> См. страницу [ansible.builtin.ping module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ping_module.html#ansible-collections-ansible-builtin-ping-module)
```yaml
# ~/.ansible/project1/playbooks
#####
---
- name: PING clients machines
  hosts:
#    - masters
    - clients
  become: true
  gather_facts: true
  vars_files:
    - /root/.ansible/project1/group_vars/masters/ssh.yml
#    - ../../group_vars/masters/ssh.yml

  vars:
    ansible_user: root
    ansible_ssh_private_key_file: "~/.ssh/id_rsa"  # path to your ssh private key

  tasks:
    - name: Test ping connectivity
      ansible.builtin.ping:

    - name: Fail if ping
      ansible.builtin.ping:
        data: CRASH
      when: false # Condition, not auto start of task
```

Проверим синтаксис плейбука: 
```bash
┌─ root ~/.ansible/project1/playbooks 
─ test-gw 
└─ # ansible-playbook -i /root/.ansible/project1/inventory/hosts.ini --syntax-check ping.yml        

playbook: ping.yml
```

А чтобы запустить данный плейбук, необходимо к нему обратиться напрямую (или поместить в эту же директорию новый инвенторный файл):
```bash
┌─ root ~/.ansible/project1/playbooks 
─ test-gw 
└─ # ansible-playbook -i /root/.ansible/project1/inventory/hosts.ini ping.yml

└─ # ansible-playbook -vv -i /root/.ansible/project1/inventory/hosts.ini ping.yml
```

Чтобы узнать все факты об ОС на клиенсткой машине, можно воспользоваться Ad-Hoc командой и при помощи grep фильтровать данные:
```bash
ansible clients -i ~/.ansible/project1/inventory/hosts.ini -m ansible.builtin.gather_facts --tree /tmp/facts;
ansible clients -i ~/.ansible/project1/inventory/hosts.ini -m ansible.builtin.gather_facts --tree /tmp/facts | grep 'pkg_mgr';

ansible -i ~/.ansible/project1/inventory/hosts.ini test-lan -m ansible.builtin.setup;

 # ansible -i ~/.ansible/project1/inventory/hosts.ini test-lan -m ansible.builtin.setup | grep mem
        "ansible_memfree_mb": 371,
        "ansible_memory_mb": {
        "ansible_memtotal_mb": 2048,
или

ansible -i /root/.ansible/project1/inventory/hosts.ini test-lan -m debug -a 'var=ansible_date_time'
ansible -i /root/.ansible/project1/inventory/hosts.ini clients -m debug -a 'var=ansible_date_time,ansible_default_ipv4
```

#### Создадим более сложный плейбук: установим приложения и соберём инфо о клиентских машинах
См. [ansible.builtin.apt module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html#ansible-collections-ansible-builtin-apt-module)
<br/> См. [Error handling in playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html) -> искать `changed_when:`, 
<br/> См. [Discovering variables: facts and magic variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html) 

```yaml
└─ # pcat playbooks/all_facts.yml 
---
- name: Write hostname
  hosts: clients
  tasks:

    - name: Print all available facts
      ansible.builtin.debug:
        var: ansible_facts

    - name: Print hostname facts
      ansible.builtin.debug:
        var: ansible_facts.hostname
```


```yaml
# root/.ansible/project1/playbooks/apt-install.yml
######

---
- name: Basic APT management
  hosts: clients
  become: true
  gather_facts: true

# The current variables do not need to be specified here.
# You may comment them.
  vars:
    ansible_user: root
    ansible_ssh_private_key_file: "~/.ssh/id_rsa"  # path to your ssh pub key

# https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html#ansible-collections-ansible-builtin-apt-module
  tasks:
    - name: Return motd to registered var
      ansible.builtin.command: cat /etc/os-release
      register: motd

    - name: Display all variables/facts known for a host
      ansible.builtin.debug:
        var: hostvars[inventory_hostname]
        verbosity: 4

    - name: Check OS and version
      ansible.builtin.debug:
        msg: "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"

    - name: Gather and display system information (OS, CPU, RAM, Processes)
      ansible.builtin.debug:
        msg: |
          Ansible version: {{ansible_version}}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }} (Kernel: {{ ansible_kernel }})
          CPU: {{ ansible_processor_vcpus }} vCPUs ({{ ansible_processor[1] if ansible_processor | length > 1 else ansible_processor[0] }})
          Total RAM: {{ (ansible_memtotal_mb / 1024) | round(2) }} GB
          Free RAM: {{ (ansible_memfree_mb / 1024) | round(2) }} GB
          Running processes: {{ ansible_facts['ansible_lsb']['processes'] if 'ansible_lsb' in ansible_facts and 'processes' in ansible_facts['ansible_lsb'] else 'N/A' }}

    - name: Update all packages to their latest version
      ansible.builtin.apt:
        name: "*"
        state: latest

    - name: Update repositories cache and install "python" and "postgreql" packages
      ansible.builtin.apt:
        update_cache: yes
        name: "{{ packages }}"
        state: present
      vars:
        packages:
          - python3.11-full
          - postgresql

    - name: Check Python version after update
      ansible.builtin.command: python3 --version
      register: python_version
      changed_when: false

```

#### Создадим несколько плейбуков под разные задачи.
Снова изменим структуру Ansible проекта, добавиви плейбук для postgresql и изменив базовый apt-install.yml, а таже написать шаблон postgresql.j2.
```
/root/.ansible/project1/
|-- ansible.cfg
|-- facts_cache
|   |-- test-lan
|   `-- test-lan2
|-- group_vars
|   |-- all.yml
|   `-- masters
|       `-- ssh.yml
|-- inventory
|   `-- hosts.ini
|-- playbooks
|   |-- apt-install.yml
|   |-- ping.yml
|   `-- postgresql-setup.yml
|-- roles
`-- templates
    `-- postgresql.j2
```

Перепишем базовый плейбук для установки приложений и сбора инфо.
```yaml
# root/.ansible/project1/playbooks/apt-install.yml
######

---
- name: Basic APT management
  hosts: clients
  become: true
  gather_facts: true

# https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html#ansible-collections-ansible-builtin-apt-module
  tasks:

    - name: Gather and display system information (OS, CPU, RAM)
      ansible.builtin.debug:
        msg: |
          Ansible version: {{ansible_version}}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }} (Kernel: {{ ansible_kernel }})
          Total RAM: {{ (ansible_memtotal_mb / 1024) | round(2) }} GB
          Free RAM: {{ (ansible_memfree_mb / 1024) | round(2) }} GB

    - name: Update all packages to their latest version
      ansible.builtin.apt:
        name: "*"
        state: latest

    - name: Update repositories cache and install "python" and "postgreql" packages
      ansible.builtin.apt:
        update_cache: yes
        name: "{{ packages }}"
        state: present
      vars:
        packages:
          - python3.11-full
          - postgresql

    - name: Check Python version after update
      ansible.builtin.command: python3 --version
      register: py_vers
      changed_when: false

# More Informative stdout
#    - name: Print Py version on a screen
#      ansible.builtin.debug:
#        msg: Version is {{ py_vers }} /*/

    - name: Display Python version
      ansible.builtin.debug:
        var: py_vers.stdout

    - name: Check Postgresql version on a screen
      ansible.builtin.command: pg_config --version
      register: pg_version
      changed_when: false

    - name: Display Postgresql version
      ansible.builtin.debug:
        var: pg_version.stdout
```

Создаём template `templates/postgresql.j2` -> см. [Шаблон postgresql.conf.j2 в Ansible](https://github.com/sherbettt/Ansible-cheats/blob/main/Шаблон%20postgresql.conf.j2%20в%20Ansible.md)



А теперь плейбук для конфигурации postgresql
<br/> См. [postgresql_db_module](https://docs.ansible.com/ansible/latest/collections/community/postgresql/postgresql_db_module.html)
<br/> См. [postgresql_user_module](https://docs.ansible.com/ansible/latest/collections/community/postgresql/postgresql_user_module.html#ansible-collections-community-postgresql-postgresql-user-module) 
<br/> См. [postgresql_pg_hba_module](https://docs.ansible.com/ansible/latest/collections/community/postgresql/postgresql_pg_hba_module.html)

```yaml
## ~/.ansible/project1/playbooks/05_psql-conf.yml
######
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
          # dest: /etc/postgresql/15/main
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

```
# /root/.ansible/project1/group_vars/clients/psql_var.yml
pg_config_cmd: "pg_config --version"
pg_etc_path: "/etc/postgresql"

# Memory settings (25% of total RAM for shared_buffers, 15% for cache)
postgresql_ram_shared: "{{ (ansible_memtotal_mb * 0.25) | int }}MB"
postgresql_ram_cache: "{{ (ansible_memtotal_mb * 0.15) | int }}MB"

postgresql_work_mem: "64MB"
postgresql_maintenance_work_mem: "256MB"
postgresql_effective_cache_size: "{{ (ansible_memtotal_mb * 0.6) | int }}MB"

```

Можно сделать тестовый запуск:
Проверим синтаксис плейбука: 
```bash
ansible-playbook -i /root/.ansible/project1/inventory/hosts.ini --syntax-check 05_psql-conf.yml
ansible-playbook -C -i /root/.ansible/project1/inventory/hosts.ini 05_psql-conf.yml
```


### 4. Создание дополнительных playbook'ов
```yaml
# pcat 00_test_loop.yml 
---
- name: Test loop
  hosts: clients
  become: true
  gather_facts: true

  tasks:
    - name: Check existing of groups
      ansible.builtin.group:
        name: "{{ item }}"
        state: present
      loop:
        - ansible
        - sudo

    - name: Create users
      ansible.builtin.user:
        name: "{{ item.name }}"
        state: present
        groups: "{{ item.groups }}"
        append: true
      loop: 
        - { name: 'alice', groups: 'sudo' }
        - { name: 'tom', groups: 'sudo' }
        - { name: 'bob', groups: 'ansible' }

```

```yaml
# pcat 00_test.yml      
---
- name: Test (when,loop)
  hosts: clients
  become: true
  gather_facts: true

  tasks:
    - name: Generate pass on test-gw
      debug:
        var: lookup('ansible.builtin.password', '/tmp/mysql_pass.txt length=12 chars=ascii_letters,digits')

    - name: And copy generated pass to test-lan*
      template:
        dest: /tmp/mysql_pass.txt
        src: /tmp/mysql_pass.txt
        owner: ansible
        group: ansible
        mode: '0644'

    - name: display memsize script
      ansible.builtin.debug:
        msg: "Watch the script: {{ lookup('ansible.builtin.file', '/root/memsize') }}"

    - name: restart systemd service if it exists on RedHat
      service:
        state: restarted
        name: sshd.service
      when: (ansible_facts['os_family'] == "RedHat") and (ansible_facts['os_version'] is version('8', '>='))
#      when: ansible_facts['services']['sshd.service']['status'] | default('not-found') != 'not-found'
#      when: ansible.builtin.stat(path="/etc/ssh/ssh_config").stat.exists

```
