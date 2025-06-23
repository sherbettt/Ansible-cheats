Переменная **`ansible_ssh_private_key_file`** в Ansible указывает путь к приватному SSH-ключу, который используется для аутентификации при подключении к удалённым хостам. 
Это полезно, когда у вас несколько ключей или стандартный ключ (`~/.ssh/id_rsa`) не подходит.  

---

## ** Основные способы указания `ansible_ssh_private_key_file`**
### ** 1 В инвентаре (inventory)**
#### **Для конкретного хоста**
```ini
[webservers]
server1.example.com ansible_ssh_private_key_file=~/.ssh/custom_key
server2.example.com ansible_ssh_private_key_file=/path/to/key.pem
```

#### **Для группы хостов**
```ini
[webservers:vars]
ansible_ssh_private_key_file=~/.ssh/deploy_key
```

---

### ** 2 В `group_vars` или `host_vars`**
```yaml
# group_vars/webservers.yml
ansible_ssh_private_key_file: ~/.ssh/webservers_key
```

```yaml
# host_vars/server1.yml
ansible_ssh_private_key_file: /opt/keys/server1_rsa
```

---

### ** 3 В командной строке (`ansible-playbook --private-key`)**
```bash
ansible-playbook -i inventory.ini playbook.yml --private-key=~/.ssh/custom_key
```
(В этом случае Ansible будет использовать указанный ключ для всех хостов.)

---

### ** 4 В `ansible.cfg` (глобально)**
```ini
[defaults]
private_key_file = ~/.ssh/ansible_key
```
(Теперь Ansible всегда будет использовать этот ключ, если не указан другой.)

---

## **🔧 Примеры использования**
### **Пример 1: Подключение к серверу с кастомным ключом**
```ini
# inventory.ini
[my_servers]
web1 ansible_host=192.168.1.10 ansible_ssh_private_key_file=~/.ssh/web1_deploy_key
db1  ansible_host=192.168.1.20 ansible_ssh_private_key_file=/etc/ansible/keys/db_rsa
```

### **Пример 2: Playbook с разными ключами для разных групп**
```yaml
# playbook.yml
- name: Configure web servers
  hosts: webservers
  vars:
    ansible_ssh_private_key_file: ~/.ssh/web_key
  tasks:
    - name: Ensure Nginx is installed
      apt:
        name: nginx
        state: present

- name: Configure database servers
  hosts: databases
  vars:
    ansible_ssh_private_key_file: ~/.ssh/db_key
  tasks:
    - name: Ensure PostgreSQL is installed
      apt:
        name: postgresql
        state: present
```

### **Пример 3: Использование ключа для AWS EC2 (обычно `.pem`)**
```ini
# inventory.ini
[aws_ec2]
ec2-54-210-123-456.compute-1.amazonaws.com ansible_ssh_private_key_file=~/.aws/my_instance_key.pem
```

---

## **⚠️ Важные замечания**
- **Права на ключ должны быть `600`** (`chmod 600 ~/.ssh/key_name`), иначе Ansible выдаст ошибку.
- Если ключ защищён паролем, Ansible запросит его при запуске (если не используется `ssh-agent`).
- Можно использовать несколько ключей для разных хостов/групп.
- Вместо `ansible_ssh_private_key_file` можно использовать короткий алиас **`ansible_private_key_file`**.

---

## **💡 Альтернативные способы аутентификации**
Если не хочется указывать ключ в инвентаре, можно:
1. **Использовать `ssh-agent`** (ключ добавляется в агент один раз):
   ```bash
   ssh-add ~/.ssh/custom_key
   ```
2. **Указать ключ в `~/.ssh/config`**:
   ```ini
   Host server1
       HostName 192.168.1.10
       User deploy
       IdentityFile ~/.ssh/server1_key
   ```

