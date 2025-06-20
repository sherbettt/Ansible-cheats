Пример плейбука 1.

```yaml
# root/.ansible/project1/playbooks/apt-install.yml
######

---
- name: Basic APT management
  hosts: clients
  become: true
  gather_facts: true

  tasks:

      # Сбор фактов об удалённых узлах
    - name: Gather and display system information (OS, CPU, RAM, Processes)
      ansible.builtin.debug:
        msg: |
          Ansible version: {{ansible_version}}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }} (Kernel: {{ ansible_kernel }})
          CPU: {{ ansible_processor_vcpus }} vCPUs ({{ ansible_processor[1] if ansible_processor | length > 1 else ansible_processor[0] }})
          Total RAM: {{ (ansible_memtotal_mb / 1024) | round(2) }} GB
          Free RAM: {{ (ansible_memfree_mb / 1024) | round(2) }} GB
          Running processes: {{ ansible_facts['ansible_lsb']['processes'] if 'ansible_lsb' in ansible_facts and 'processes' in ansible_facts['ansible_lsb'] else 'N/A' }}

      # Обновить все пакеты на Debian машинах
    - name: Update all packages to their latest version
      ansible.builtin.apt:
        name: "*"
        state: latest

      # Установить "python" , "postgreql"
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


    # Узнать версию Postgresql командой pg_config --version
    # И записать в переменную pg_version
    - name: Check Postgresql version after update
      ansible.builtin.command: pg_config --version
      register: pg_version
      changed_when: false

    # Вывести на экран версию - значение переменной pg_version
    - name: Display Postgresql version
      ansible.builtin.debug:
        var: pg_version.stdout

     # Сократить pg_version до цифрого значения
    - name: Get PostgreSQL version (short form, e.g., "16")
      ansible.builtin.set_fact:
        pg_short_version: "{{ pg_version.stdout.split('.')[0] | regex_replace('[^0-9]', '') }}"

      # Вывести на экран номер версии из переменной pg_short_version
    - name: Debug PostgreSQL short version
      ansible.builtin.debug:
        msg: "PostgreSQL short version: {{ pg_short_version }}"

      # Зафиксировать результат в переменной pg_folder
    - name: Redister PSQL folder
      ansible.builtin.command: ls -alF /etc/postgresql/{{ pg_short_version }}/main
      register: pg_folder
      changed_when: false

    # Вывести на экран контент папки
    - name: Display PSQL folder contents
      ansible.builtin.debug:
        msg: "PostgreSQL folder contents: {{ pg_folder.stdout_lines }}"

```

Можно определить переменные в начале, чтобы было проще искать номера версий для postgresql.

```yaml
# root/.ansible/project1/playbooks/apt-install.yml
######

---
- name: Basic APT management
  hosts: clients
  become: true
  gather_facts: true

  vars:
    - pg_config_cmd: "pg_config --version"
    - pg_etc_path: "/etc/postgresql"
    - pg_main_folder: "main"

# Либо выше указанные переменные можно определить в отдельном файле
#  vars_files:
#    - /root/.ansible/project1/group_vars/psql_vars.yml

  tasks:

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
      register: py_vers
      changed_when: false

# More Informative stdout
#    - name: Print Py version on a screen
#      ansible.builtin.debug:
#        msg: Version is {{ py_vers }} /*/

    - name: Display Python version
      ansible.builtin.debug:
        var: py_vers.stdout


    - name: Check Postgresql version after update
      ansible.builtin.command: "{{ pg_config_cmd }}"
      register: pg_version
      changed_when: false

    - name: Display Postgresql version
      ansible.builtin.debug:
        var: pg_version.stdout

    - name: Get PostgreSQL version (short form, e.g., "16")
      ansible.builtin.set_fact:
        pg_short_version: "{{ pg_version.stdout.split('.')[0] | regex_replace('[^0-9]', '') }}"

    - name: Debug PostgreSQL short version
      ansible.builtin.debug:
        msg: "PostgreSQL short version: {{ pg_short_version }}"

    - name: Register PSQL folder
      ansible.builtin.command: "ls -alF {{ pg_etc_path }}/{{ pg_short_version }}/{{ pg_main_folder }}"
      register: pg_folder
      changed_when: false

    - name: Display PSQL folder contents
      ansible.builtin.debug:
        msg: "PostgreSQL folder contents: {{ pg_folder.stdout_lines }}"

```
