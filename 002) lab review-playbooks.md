
### Анализ задачи и пошаговые решения

#### Исходные данные:
- **Каталог структуры**:
```bash
tree .
.
├── ansible.cfg
├── defaults-template.yml
├── inventory
├── vars.yml
└── vsftpd.conf.j2
```

- **Файл vars.yml**:
```yaml
---
# vars file for ansible-vsftpd
vsftpd_package: vsftpd
vsftpd_service: vsftpd
vsftpd_config_file: /etc/vsftpd/vsftpd.conf
```

- **Шаблон vsftpd.conf.j2**:
```j2
# Vsftpd configuration
# {{ ansible_managed }}

connect_from_port_20={{ 'YES' if vsftpd_connect_from_port_20 else 'NO' }}
listen={{ 'YES' if vsftpd_listen else 'NO' }}
pam_service_name=vsftpd
syslog_enable={{ 'YES' if vsftpd_syslog_enable else 'NO' }}

anonymous_enable={{ 'YES' if vsftpd_anonymous_enable else 'NO' }}
{% if vsftpd_anon_root is defined %}
anon_root={{ vsftpd_anon_root }}
{% endif %}

local_enable={{ 'YES' if vsftpd_local_enable else 'NO' }}
{% if vsftpd_local_root is defined %}
local_root={{ vsftpd_local_root }}
{% endif %}
local_umask=022

write_enable={{ 'YES' if vsftpd_write_enable else 'NO' }}

chroot_local_user={{ 'YES' if vsftpd_chroot_local_user else 'NO' }}

pasv_enable=Yes
pasv_min_port=21000
pasv_max_port=21020
```

- **Файл defaults-template.yml**:
```yaml
---
# defaults file for ansible-vsftpd
vsftpd_anonymous_enable: true
vsftpd_connect_from_port_20: true
vsftpd_listen: true
vsftpd_local_enable: false
vsftpd_setype: public_content_t
vsftpd_syslog_enable: true
vsftpd_write_enable: true
vsftpd_chroot_local_user: true
vsftpd_anon_root: /var/ftp
vsftpd_local_root: /var/ftp
```

- **Файл inventory**:
```bash
[ftpservers]
serverb.lab.example.com
serverd.lab.example.com

[ftpclients]
serverc.lab.example.com
```

- **Файл ansible.cfg**:
```ini
[defaults]
remote_user = devops
inventory = ./inventory

[privilege_escalation]
become_user = root
become_method = sudo
become = true
```

### Шаги для выполнения задачи:

1. **Создать инвенторный файл**: Уже выполнен.
2. **Создать ансибл-конфигурационный файл**: Уже выполнен.
3. **Создать плейбук ftpclients.yml**: Нужно создать плейбук, который устанавливает пакет lftp на клиентские хосты.
4. **Скопировать предоставленный шаблон**: Нужно поместить файл vsftpd.conf.j2 в каталог templates.
5. **Поместить файл defaults-template.yml в каталог vars**: Скопировать defaults-template.yml в vars.
6. **Создать vars.yml файл**: Нужен файл vars.yml для определения основных переменных.
7. **Создать плейбук ansible-vsftpd.yml**: Должен обеспечить установку и конфигурацию vsftpd на серверах.
8. **Создать плейбук site.yml**: Он включает предыдущие плейбуки.
9. **Исполнить плейбук site.yml**: Проверить правильность работы.

### Реализация

#### 1. Создаем плейбук ftpclients.yml:
```yaml
---
- name: Установка LFTP на FTP клиенты
  hosts: ftpclients
  become: true
  tasks:
    - name: Ensure lftp is installed
      package:
        name: lftp
        state: latest
```

#### 2. Помещаем шаблон vsftpd.conf.j2 в каталог templates:
```bash
mkdir -v templates
mv vsftpd.conf.j2 templates/
```

#### 3. Помещаем defaults-template.yml в каталог vars:
```bash
mkdir -v vars
mv defaults-template.yml vars/
```

#### 4. Создаем vars.yml:
```yaml
---
# vars file for ansible-vsftpd
vsftpd_package: vsftpd
vsftpd_service: vsftpd
vsftpd_config_file: /etc/vsftpd/vsftpd.conf
```

#### 5. Создаем плейбук ansible-vsftpd.yml:
```yaml
---
- name: Setup VSFTPD Servers
  hosts: ftpservers
  become: true
  vars_files:
    - vars/defaults-template.yml
    - vars/vars.yml
  tasks:
    - name: Ensure vsftpd package is installed
      package:
        name: "{{ vsftpd_package }}"
        state: present

    - name: Render vsftpd configuration file
      template:
        src: templates/vsftpd.conf.j2
        dest: "{{ vsftpd_config_file }}"
        owner: root
        group: root
        mode: '0644'
      notify: Restart vsftpd

    - name: Enable and start vsftpd service
      service:
        name: "{{ vsftpd_service }}"
        state: started
        enabled: true

  handlers:
    - name: Restart vsftpd
      service:
        name: "{{ vsftpd_service }}"
        state: restarted​<footnote id="1" index=1 href="https://linuxlife.page/posts/15-ftp-server-ansible/" title="FTPS-сервер. Быстрая установка с помощью Ansible и vsftp"/>
```

#### 6. Создаем плейбук site.yml:
```yaml
---
- name: Site-wide deployment
  hosts: all
  become: true
  pre_tasks:
    - name: Include playbooks
      include_tasks: ftpclients.yml
      when: inventory_hostname in groups.ftpclients

    - name: Include playbooks
      include_tasks: ansible-vsftpd.yml
      when: inventory_hostname in groups.ftpservers​<footnote id="1" index=2 href="https://linuxlife.page/posts/15-ftp-server-ansible/" title="FTPS-сервер. Быстрая установка с помощью Ansible и vsftp"/>
```

### Исполнение плейбука site.yml:
```bash
ansible-playbook site.yml
```

### Итоги:
- Созданы нужные плейбуки и файлы.
- Вся инфраструктура описана с учетом существующих требований.
- Тестируется работа плейбуков с нужными действиями.

Таким образом, задача успешно решена, и весь процесс автоматизирован средствами Ansible.
