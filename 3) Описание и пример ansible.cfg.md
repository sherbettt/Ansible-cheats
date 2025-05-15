### Описание `/etc/ansible/ansible.cfg`  

Файл `ansible.cfg` — это конфигурационный файл Ansible, который определяет глобальные настройки поведения утилиты. Он может содержать множество секций и параметров.  

#### Основные секции и переменные:  

1. **[defaults]** – основные настройки Ansible.  
   - `inventory = /etc/ansible/hosts` – путь к инвентарному файлу.  
   - `remote_user = root` – пользователь для подключения к удалённым хостам.  
   - `ask_pass = True` – запрашивать пароль при подключении.  
   - `ask_sudo_pass = True` – запрашивать sudo-пароль.  
   - `host_key_checking = False` – отключает проверку SSH-ключа хоста (небезопасно, но удобно для тестов).  
   - `log_path = /var/log/ansible.log` – путь к лог-файлу.  
   - `forks = 5` – количество параллельных процессов.  
   - `gathering = smart` – политика сбора фактов (smart, implicit, explicit).  
   - `timeout = 10` – таймаут SSH-подключения.  

2. **[privilege_escalation]** – настройки повышения привилегий.  
   - `become = True` – активирует режим повышения прав.  
   - `become_method = sudo` – метод повышения прав (sudo/su/pbrun и др.).  
   - `become_user = root` – пользователь, от которого выполняются команды.  
   - `become_ask_pass = False` – запрашивать пароль для `become`.  

3. **[ssh_connection]** – настройки SSH.  
   - `ssh_args = -C -o ControlMaster=auto -o ControlPersist=60s` – дополнительные аргументы SSH.  
   - `control_path = ~/.ssh/ansible-%%r@%%h:%%p` – путь к ControlSocket.  
   - `pipelining = False` – ускоряет выполнение, но требует `requiretty` в sudoers.  

4. **[colors]** – настройки цветов вывода.  
   - `highlight = white`  
   - `verbose = blue`  
   - `warn = bright purple`  

---

### Простой пример `ansible.cfg`  

```ini
[defaults]
inventory = ./inventory.ini
remote_user = ansible
host_key_checking = False
log_path = ./ansible.log
forks = 5
gathering = smart

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
ssh_args = -C -o ControlMaster=auto -o ControlPersist=60s
pipelining = True

[colors]
highlight = white
verbose = blue
error = red
warn = yellow
```

Этот пример:  
- Использует инвентарь `inventory.ini` в текущей директории.  
- Подключается от имени пользователя `ansible` с автоматическим повышением прав через `sudo`.  
- Логирует действия в `ansible.log`.  
- Ускоряет выполнение за счёт `pipelining`.  
- Настраивает цвета вывода.  

Можно разместить в:  
- `/etc/ansible/ansible.cfg` (глобально)  
- `~/.ansible.cfg` (для текущего пользователя)  
- `./ansible.cfg` (локально в проекте, имеет наивысший приоритет).  

Если нужны дополнительные параметры – можно посмотреть полный список командой:  
```sh
ansible-config list
```

---
Чтобы заставить Ansible использовать стандартный конфигурационный файл `ansible.cfg` вместо кастомного, у вас есть несколько вариантов:

### 1. Удалить или переименовать кастомный `ansible.cfg`
Просто переместите или удалите файл `/home/student/playbook-review/ansible.cfg`, и Ansible автоматически начнёт использовать стандартный конфиг из одного из расположений по умолчанию:
```bash
mv /home/student/playbook-review/ansible.cfg /home/student/playbook-review/ansible.cfg.bak
```

### 2. Указать другой конфигурационный файл через переменную окружения
Вы можете задать путь к другому конфигурационному файлу через переменную `ANSIBLE_CONFIG`:
```bash
export ANSIBLE_CONFIG=/etc/ansible/ansible.cfg
```
или (если хотите вообще не использовать конфиг):
```bash
unset ANSIBLE_CONFIG
```

### 3. Указать другой конфигурационный файл через параметр командной строки
При запуске Ansible можно явно указать другой конфигурационный файл:
```bash
ansible --config /etc/ansible/ansible.cfg ...
```

### Где Ansible ищет конфигурационный файл по умолчанию?
Ansible ищет `ansible.cfg` в следующем порядке:
1. Переменная окружения `ANSIBLE_CONFIG` (указывает на файл)
2. `./ansible.cfg` (в текущей директории)
3. `~/.ansible.cfg` (в домашней директории пользователя)
4. `/etc/ansible/ansible.cfg` (глобальный конфиг)

Таким образом, если вы удалите кастомный файл из текущей директории, Ansible будет использовать следующий доступный вариант в этом списке.
