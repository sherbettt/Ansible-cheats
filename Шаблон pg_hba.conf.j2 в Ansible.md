См.: [Templating (Jinja2)](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html)

### Пример `pg_hba.conf.j2`

```jinja
## ~/.ansible/project1/templates/pg_hba.conf.j2
#####
# TYPE  DATABASE  USER      ADDRESS           METHOD
local   all       all                         peer
host    all       all       127.0.0.1/32      scram-sha-256
host    all       all       ::1/128           scram-sha-256
#host    all       all       192.168.56.0/24   scram-sha-256  # Allow LAN

{% for ip in ['192.168.56.2/32' , '192.168.56.3/32'] %}
host    all       all       {{ip}}            md5
{% endfor %}

```
или

```jinja
## ~/.ansible/project1/templates/pg_hba.conf.j2
#####
# TYPE  DATABASE  USER      ADDRESS           METHOD
local   all       all                         peer
host    all       all       127.0.0.1/32      scram-sha-256
host    all       all       ::1/128           scram-sha-256
#host    all       all       192.168.56.0/24   scram-sha-256  # Allow LAN

{% for ip in ips  %}
host    all       all       {{ip}}            md5
{% endfor %}
```

```yaml
# ~/.ansible/project1/group_vars/clients/pg_vars.yml
ips: ['192.168.56.2/32' , '192.168.56.3/32']
```

### Частный пример использования jinja2:
```jinja
# ~/.ansible/project1/templates/test.j2 
My name is {{ ansible_facts['hostname'] }}
Distribution is {{ ansible_facts['distribution'] }}
Interface is {{ ansible_facts['eth0'] }}
```

```yaml
# pcat ~/.ansible/project1/playbooks/00_test4j2.yml 
---
- name: Write hostname
  hosts: clients
  tasks:
  - name: write hostname using jinja2
    ansible.builtin.template:
       src: /root/.ansible/project1/templates/test.j2
       dest: /tmp/hostname
```


