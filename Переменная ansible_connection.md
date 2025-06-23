Переменная `ansible_connection` в Ansible определяет тип подключения к управляемым узлам (hosts). 
Она указывает, каким способом Ansible будет взаимодействовать с целевыми системами.  

### **Допустимые значения `ansible_connection`:**
1. **`local`** – выполнение задач на локальной машине (control node).  
2. **`ssh`** (по умолчанию) – подключение по SSH (требует настроенного SSH-доступа).  
3. **`paramiko`** – устаревшая реализация SSH на Python (используется, если нет native SSH).  
4. **`docker`** – прямое управление контейнерами Docker.  
5. **`winrm`** – подключение к Windows-хостам через WinRM.  
6. **`psrp`** – подключение к Windows через PowerShell Remoting Protocol.  
7. **`network_cli`** – для сетевых устройств (Cisco, Juniper и др.) через CLI.  
8. **`httpapi`** – для устройств с REST API (например, Fortinet, F5).  
9. **`netconf`** – для сетевых устройств с поддержкой NETCONF.  
10. **`smart`** – автоматический выбор (устаревший вариант).  

---

### **Примеры использования `ansible_connection`**

#### **1. Локальное выполнение (`local`)**
```yaml
- name: Run a local command
  hosts: localhost
  connection: local
  tasks:
    - name: Check disk space
      command: df -h
```

#### **2. Подключение по SSH (`ssh`)**
```yaml
- name: Install nginx on remote server
  hosts: webservers
  connection: ssh  # можно не указывать, так как это значение по умолчанию
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
```

#### **3. Управление Docker-контейнерами (`docker`)**
```yaml
- name: Manage Docker container
  hosts: localhost
  connection: docker
  tasks:
    - name: Start a container
      docker_container:
        name: my_nginx
        image: nginx
        state: started
```

#### **4. Подключение к Windows через WinRM (`winrm`)**
```yaml
- name: Configure Windows host
  hosts: windows_servers
  connection: winrm
  tasks:
    - name: Create a directory
      win_file:
        path: C:\Temp\Ansible
        state: directory
```

#### **5. Управление сетевым устройством через CLI (`network_cli`)**
```yaml
- name: Configure Cisco switch
  hosts: cisco_switch
  connection: network_cli
  tasks:
    - name: Enable interface Gig0/1
      cisco.ios.ios_config:
        lines:
          - no shutdown
        parents: interface GigabitEthernet0/1
```

#### **6. Подключение к устройству через API (`httpapi`)**
```yaml
- name: Configure FortiGate firewall
  hosts: fortigate_fw
  connection: httpapi
  tasks:
    - name: Create a new firewall policy
      fortios_firewall_policy:
        policyid: 1
        srcintf: ["port1"]
        dstintf: ["port2"]
        action: accept
```

---

### **Где указывается `ansible_connection`?**
1. **В инвентаре (inventory)**
   ```ini
   [webservers]
   server1 ansible_connection=ssh
   server2 ansible_connection=winrm
   ```
2. **В playbook (как показано выше)**  
3. **В переменных хоста/группы (`group_vars`, `host_vars`)**  
   ```yaml
   # group_vars/docker.yml
   ansible_connection: docker
   ```
4. **В конфиге Ansible (`ansible.cfg`)**  
   ```ini
   [defaults]
   connection = ssh
   ```


