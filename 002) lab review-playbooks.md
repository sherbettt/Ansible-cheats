### Решение задачки по Ansible

#### Шаг 1: Создание инвентарного файла (`inventory`)
Создаем файл `/home/student/review-playbooks/inventory` следующего содержания:
```bash
[ftpclients]
serverc.lab.example.com

[ftpservers]
serverb.lab.example.com
serverd.lab.example.com
```

#### Шаг 2: Настройка конфигурационного файла `ansible.cfg`
Создаем файл `/home/student/review-playbooks/ansible.cfg`, содержащий следующие настройки:
```ini
[defaults]
inventory = ./inventory
remote_user = devops
become = True
become_method = sudo
become_user = root
```

#### Шаг 3: Создание плейбука для установки пакета lftp (`ftpclients.yml`)
Создаем файл `/home/student/review-playbooks/ftpclients.yml` с таким содержимым:
```yaml
---
- name: Install lftp package on FTP clients
  hosts: ftpclients
  become: true
  tasks:
    - name: Ensure lftp package is installed
      yum:
        name: lftp
        state: present
```

#### Шаг 4: Перемещение шаблона конфигурации (`vsftpd.conf.j2`) в каталог `templates`
Перемещаем существующий шаблон `vsftpd.conf.j2` в подпапку `./templates`.
```bash
mkdir -p templates && mv vsftpd.conf.j2 templates/
```

#### Шаг 5: Перемещение файла `defaults-template.yml` в папку `vars`
Перемещаем файл `defaults-template.yml` в директорию `./vars`.
```bash
mkdir -p vars && mv defaults-template.yml vars/
```

#### Шаг 6: Определение переменных в файле `vars.yml`
Создаем файл `/home/student/review-playbooks/vars/vars.yml` с такими переменными:
```yaml
---
vsftpd_package: vsftpd
vsftpd_service: vsftpd
vsftpd_config_file: "/etc/vsftpd/vsftpd.conf"
```
убедиться, что текущий контекст соответствует ожидаемому типу с помощью команды: 
`ansible -i inventory ftpservers -b -a "ls -Z /etc/vsftpd/vsftpd.conf" --become-method=sudo`

#### Шаг 7: Плейбук для конфигурирования сервиса vsftpd (`ansible-vsftpd.yml`)
Создаем второй плейбук `/home/student/review-playbooks/ansible-vsftpd.yml` следующим образом:
```yaml
---
- name: FTP server is installed
  become: true
  hosts:
   - ftpservers
  vars_files:
   - vars/defaults-template.yml
   - vars/vars.yml
  tasks:
   - name: Packages are installed
     yum:
       name: "{{ vsftpd_package }}"
       state: present

   - name: Ensure service is started
     service:
       name: "{{ vsftpd_service }}"
       state: started
       enabled: true

   - name: Configuration file is installed
     template:
       src: templates/vsftpd.conf.j2
       dest: "{{ vsftpd_config_file }}"
       owner: root
       group: root
       mode: '0600'
       setype: etc_t
     notify: restart vsftpd

   - name: firewalld is installed
     yum:
       name: firewalld
       state: present

   - name: firewalld is started and enabled
     service:
       name: firewalld
       state: started
       enabled: yes

   - name: FTP port is open
     firewalld:
       service: ftp
       permanent: true
       state: enabled
       immediate: yes

   - name: FTP passive data ports are open
     firewalld:
       port: 21000-20/tcp  # port: 21000-21020/tcp
       permanent: yes
       state: enabled
       immediate: yes

  handlers:
   - name: restart vsftpd
     service:
       name: "{{ vsftpd_service }}"
       state: restarted
```

#### Шаг 8: Объединение плейбоков в единый плейбук (`site.yml`)
Создаем основной плейбук `/home/student/review-playbooks/site.yml`, включающий оба ранее созданных плейбука:
```yaml
---
# FTP Servers playbook
- import_playbook: ftpclients.yml

# FTP Clients playbook
- import_playbook: ansible-vsftpd.yml
```

#### Шаг 9: Запуск плейбуков для проверки
Запустим плейбук командой:
```bash
cd ~/review-playbooks
ansible-playbook site.yml
```

При успешном выполнении мы увидим вывод примерно такой формы:
```
PLAY RECAP *********************************************************************
serverb.lab.example.com : ok=3 changed=1 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0   
serverc.lab.example.com : ok=1 changed=1 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0   
serverd.lab.example.com : ok=3 changed=1 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0  
```

Таким образом, проверяется установка пакетов и настройка сервисов на целевых хостах согласно заданному плану.
