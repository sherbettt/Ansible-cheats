Есть Ubuntu подобный роутер. На нём три интерфейса:
<br/> eth0 - `192.168.87.112` (Главная Управляющая Ansible машина (смотрит в интернет)), шлюз: `192.168.87.1/24`;
<br/> eth1 - `192.168.56.1` (не смотрит в интернет, для связи с машинами);
<br/> eth2 - `192.168.96.113` (смотрит в интернет), шлюз: `192.168.96.1/24`
<br/> Имеется дополнительно управляющая Ansible машина: `192.168.87.136/24`.

Также есть две другие Ubuntu управляемые машины, подключённые к выше указанном роутеру с адресами: `192.168.56.2/24`; `192.168.56.3/24`. Аутентификация на машины по ключу.

На управляемых машинах требуется провести действия:
  - собрать факты об удалённых узлах;
  - установить python;
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
[routers]
router ansible_host=192.168.56.1

[servers]
server1 ansible_host=192.168.56.2
server2 ansible_host=192.168.56.3

[all:children]
routers
servers
```
где:
1. Секция `[routers]` содержит хост router с указанием его IP-адреса
2. Секция `[servers]` содержит два сервера с их IP-адресами
3. Секция `[all:children]` объединяет группы routers и servers в одну группу all

Файл **`ansible.cfg`**:
```cfg
[defaults]
inventory = ./inventory/hosts.ini
remote_user = ansible  # Лучше не root
become = True
become_user = root
host_key_checking = False
log_path = /var/log/ansible.log
forks = 1
gathering = smart
#timeout = 10  # Добавлено для надёжности SSH-сессий

# Кэширование фактов
fact_caching = jsonfile
fact_caching_connection = ./facts_cache
fact_caching_timeout = 0 # В некоторых версиях Ansible это отключает таймаут

# Кеширование неограниченное
# fact_caching = never_expire
# fact_caching_connection = ./facts_cache
# fact_caching_timeout можно не указывать

[privilege_escalation]
become_method = sudo  # Явное указание метода повышения прав
```

| Параметр                     | Состояние | Комментарий |
|-------------------------------|-----------|-------------|
| `inventory = ./inventory/hosts.ini` | ✅ | Правильный путь к INI-инвентарю. |
| `remote_user = root`          | ✅ | Явное указание пользователя. |
| `become = True` + `become_user = root` | ✅ | Автоматическое повышение прав (аналог `sudo -i`). |
| `host_key_checking = False`    | ✅ | Для тестовой среды — норма. |
| `log_path = /var/log/ansible.log` | ✅ | Логирование в файл (убедитесь, что есть права на запись). |
| `forks = 1`                   | ✅ | Соответствует требованиям RUNTEL.RU (последовательное выполнение). |
| `gathering = smart`           | ✅ | Оптимальный выбор для кэширования фактов. |
| `fact_caching = jsonfile`     | ✅ | Корректный формат кэша. |
| `fact_caching_connection = ./facts_cache` | ✅ | Локальное хранение кэша. |
| `fact_caching_timeout = 86400` | ✅ | Актуальность данных — 24 часа. |
| `fact_caching_timeout = 9999999999` | ✅ | Актуальность данных — ~317 лет в секундах. |

