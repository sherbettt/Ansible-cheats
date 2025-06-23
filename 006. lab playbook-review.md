Старт:
1. `lab playbook-review start`

```bash
[student@workstation playbook-review]$ cat ansible.cfg
[defaults]
inventory=inventory
remote_user=devops

[privilege_escalation]
become=False
become_method=sudo
become_user=root
become_ask_pass=False
[student@workstation playbook-review]$
[student@workstation playbook-review]$ cat inventory
serverb.lab.example.com
```


Весь файл inventory (playbook) выглядить так:

```yaml
---
- name: Enable internet services
  hosts: serverb.lab.example.com
  become: yes
  tasks:
    - name: latest version of packages installed
      yum:
        name:
          - httpd
          - firewalld
          - php
          - php-mysqlnd
          - mariadb-server
        state: latest

    - name: firewalld enabled and running
      service:
        name: firewalld
        enabled: true
        state: started

    - name: firewalld permits access to httpd service
      firewalld:
        service: http
        permanent: true
        state: enabled
        immediate: yes

    - name: httpd enabled and running
      service:
        name: httpd
        enabled: true
        state: started

    - name: test php page is installed
      get_url:
        url: "http://materials.example.com/labs/playbook-review/index.php"
        dest: /var/www/html/index.php
        mode: 0644

- name: Test internet web server
  hosts: localhost
  become: no
  tasks:
    - name: connect to internet web server
      uri:
        url: http://serverb.lab.example.com
        status_code: 200
```

Проверка: 
1. `ansible-playbook --syntax-check  internet.yml`
2. `ansible-playbook internet.yml`
3. `lab playbook-review grade`
4. `lab playbook-review finish`
