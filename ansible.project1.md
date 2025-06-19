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

См.: [Описание и пример ansible.cfg](https://github.com/sherbettt/Ansible-cheats/blob/main/03.%20Описание%20и%20пример%20ansible.cfg.md), 
[Условие when](https://github.com/sherbettt/Ansible-cheats/blob/main/50.%20Условие%20when.md)

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
```ini
[masters]
# check group_vars/masters/ssh.yml
#test-gw ansible_host=192.168.87.112 ansible_user=ansible
test-gw ansible_host=192.168.87.112 ansible_user=root ansible_ssh_common_args=''
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

---

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
# ssh_args = -o BindAddress=192.168.56.1
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




