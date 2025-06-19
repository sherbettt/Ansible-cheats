Есть Ubuntu подобный роутер. На нём три интерфейса:
<br/> eth0 - `192.168.87.112` (Главная Управляющая Ansible машина (смотрит в интернет)), шлюз: `192.168.87.1/24`;
<br/> eth1 - `192.168.56.1` (не смотрит в интернет, для связи с локальными машинами);
<br/> eth2 - `192.168.96.113` (смотрит в интернет), шлюз: `192.168.96.1/24`
<br/> Имеется дополнительно управляющая Ansible машина: `192.168.87.136/24`.

Также есть две другие Ubuntu управляемые машины, подключённые к выше указанном роутеру с адресами: `192.168.56.2/24`; `192.168.56.3/24`. Аутентификация на машины по ключу.

На управляемых машинах требуется провести действия:
  - собрать факты об удалённых узлах;
  - установить python (разные версии для теста);
  - установить postgreql;
  - создать template postgresql.j2

См.: [Описание и пример ansible.cfg](https://github.com/sherbettt/Ansible-cheats/blob/main/03.%20Описание%20и%20пример%20ansible.cfg.md), 
[Условие when](https://github.com/sherbettt/Ansible-cheats/blob/main/50.%20Условие%20when.md)

-----------------------------------------------------------------------
## Ansible на роутере.

#### 1. Перед созданием структуры требуется проверить соединение по SSH.
Так как аутентификация по ключу уже настроена, убедитесь, что:
- На роутере (`192.168.87.112`) есть приватный ключ (`~/.ssh/id_rsa`).
- Публичный ключ (`~/.ssh/id_rsa.pub`) добавлен в `~/.ssh/authorized_keys` на управляемых машинах (`192.168.56.2` и `192.168.56.3`).

Проверить подключение:
```bash
ssh -i ~/.ssh/id_rsa root@192.168.56.2
ssh -i ~/.ssh/id_rsa root@192.168.56.3
```

#### 2. Создадим структура проекта:
Для начала рассмотрим управление с роутера (`192.168.87.112/24`), установив на него предварительно Ansible.
Структура проекта будет выглядеть примерно так на начальном этапе:
```
~/.ansible/project1/
├── inventory/
│   ├── hosts.ini          # Инвентарный файл
├── group_vars/
│   └── all.yml            # Общие переменные
├── playbooks/
│   └── site.yml           # Основной playbook
└── ansible.cfg            # Конфигурация Ansible
```


```bash
mkdir -p ~/.ansible/project1/{inventory,group_vars,roles}
cd ~/.ansible/project
```

 Файл **`~/.ansible/project1/inventory/hosts.ini`**:
```ini
[masters]
test-gw ansible_host=192.168.87.112 ansible_user=root
#kiko0217 ansible_host=192.168.87.136 ansible_user=root

[clients]
test-lan ansible_host=192.168.56.2
test-lan2 ansible_host=192.168.56.3

[all:children]
masters
clients
```
где:
1. Секция `[masters]` содержит хосты управляющих машиг с указанием их IP-адреса и ansible-пользователя
2. Секция `[clients]` содержит два управляемых сервера с их IP-адресами
3. Секция `[all:children]` объединяет группы masters и clients в одну группу all

Файл **`ansible.cfg`**:
```ini
[defaults]
inventory = ./inventory/hosts.ini
remote_user = ansible  # vs root
host_key_checking = True
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

#[ssh_connection]
#host_key_checking = true
```
Проверка параметров командой: `ansible-config dump --only-changed`



