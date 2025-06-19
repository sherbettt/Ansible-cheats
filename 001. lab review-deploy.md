Условие.
```
Before You BeginLog in to workstation as student using student as the password.On workstation, run the lab review-deploy start command. This script ensuresthat the managed hosts, servera and serverb, are reachable on the network. The scriptcreates a lab subdirectory named review-deploy in the student's home directory.[student@workstation ~]$ lab review-deploy startInstructions

 Install Ansible on workstation so that it can serve the control node.
 On the control node, create an inventory file, /home/student/review-deploy/inventory, containing a group called dev. This group should consist of the managed hostsservera.lab.example.com and serverb.lab.example.com.
Create the Ansible configuration file in /home/student/review-deploy/ansible.cfg.The configuration file should reference the /home/student/review-deploy/inventory inventory file.
Execute an ad hoc command using privilege escalation to modify the contents of the /etc/motd file on servera and serverb to contain the string Managed by Ansible\n. Usedevops as the remote user.
Execute an ad hoc command to verify that the contents of the /etc/motd file on serveraand serverb are identical.
```


Для решения данной задачи выполните следующие шаги подробно:

### Шаг 1: Установка Ansible на рабочую станцию (workstation)

```bash
sudo dnf install epel-release -y
sudo dnf install ansible -y
```

---

### Шаг 2: Создание файла инвентаря `/home/student/review-deploy/inventory`

Создайте файл инвентаря и добавьте в него группу `dev` с хостами `servera.lab.example.com` и `serverb.lab.example.com`.

```bash
cat > /home/student/review-deploy/inventory <<EOF
[dev]
servera.lab.example.com
serverb.lab.example.com
EOF
```

---

### Шаг 3: Создание конфигурационного файла `/home/student/review-deploy/ansible.cfg`

Настройте файл конфигурации таким образом, чтобы он указывал на созданный ранее файл инвентаря.

```bash
cat > /home/student/review-deploy/ansible.cfg <<EOF
[defaults]
inventory = /home/student/review-deploy/inventory
remote_user = devops
become = True
become_method = sudo
EOF
```

---

### Шаг 4: Выполнение команды ad-hoc для изменения содержимого файла `/etc/motd`

Используйте команду ad-hoc для записи строки `"Managed by Ansible"` в файл `/etc/motd` на обоих серверах (`servera` и `serverb`) с использованием привилегий суперпользователя (privilege escalation).

```bash
ansible dev -m lineinfile -a "line='Managed by Ansible' dest=/etc/motd state=present"
```
или
```bash
ansible dev -m copy -a "content='Managed by Ansible\n' dest=/etc/motd"
```
или если не создавали cfg файл
```bash
ansible dev -m copy -a 'content="Managed by Ansible\n" dest=/etc/motd' -b -u devops
```
---

### Шаг 5: Проверка идентичности содержимого файла `/etc/motd` на обоих серверах

Выполните следующую команду ad-hoc для проверки одинаковости содержимого файла `/etc/motd` на обеих машинах:
```bash
ansible dev -m shell -a "cat /etc/motd"
или
ansible dev -m command -a "cat /etc/motd"
```
Ожидается, что вывод будет одинаковым и содержать строку `"Managed by Ansible"`.

---

### Полная последовательность команд:

```bash
# Шаг 1: Установка Ansible
sudo dnf install epel-release -y
sudo dnf install ansible -y

# Шаг 2: Создаем файл инвентаря
cat > /home/student/review-deploy/inventory <<EOF
[dev]
servera.lab.example.com
serverb.lab.example.com
EOF

# Шаг 3: Создаем файл конфигурации Ansible
cat > /home/student/review-deploy/ansible.cfg <<EOF
[defaults]
inventory = /home/student/review-deploy/inventory
remote_user = devops
become = True
become_method = sudo
EOF

# Шаг 4: Запись строки в файл motd на сервере с правами root
ansible dev -m copy -a "content='Managed by Ansible\n' dest=/etc/motd"

# Шаг 5: Проверяем содержимое файла motd на обоих серверах
ansible dev -m shell -a "cat /etc/motd"
```

Вот эквивалентные задачи для указанных команд ad-hoc в виде плейбука Ansible:

## Плейбук для установки текста "Managed by Ansible" в файл `/etc/motd`:

```yaml
---
- name: Update /etc/motd content on servers
  hosts: dev
  become: true
  vars:
    motd_content: "Managed by Ansible\n"
  
  tasks:
    - name: Ensure /etc/motd contains specific message
      copy:
        content: "{{ motd_content }}"
        dest: "/etc/motd"
      remote_user: devops
```

---

## Плейбук для вывода содержимого файла `/etc/motd`:

```yaml
---
- name: Verify /etc/motd content on servers
  hosts: dev
  tasks:
    - name: Output current /etc/motd content
      shell: cat /etc/motd
      register: motd_output
    
    - name: Display the output from /etc/motd
      debug:
        var: motd_output.stdout_lines
```

---

## Объяснения:

- **Первый плейбук** обновляет файл `/etc/motd`, используя модуль `copy`. Мы используем переменную `motd_content`, чтобы сделать код более читаемым и легко настраиваемым.
- **Второй плейбук** выводит содержимое файла `/etc/motd` с помощью модуля `shell`. Команда регистрирует вывод в переменной `motd_output`, которую затем отображаем с помощью модуля `debug`.

Запуск плейбуков осуществляется командой:

```bash
ansible-playbook path/to/your-playbook.yml
```

Эти плейбуки будут выполнять аналогичные действия указанным командам ad-hoc, обеспечивая лучший контроль и повторяемость операций.
