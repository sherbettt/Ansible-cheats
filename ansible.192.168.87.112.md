Есть Ubuntu подобный роутер. На нём три интерфейса:
<br/> eth0 - `192.168.87.112` (смотрит в интернет), шлюз: `192.168.87.1/24`;
<br/> eth1 - `192.168.56.1` (не смотрит в интернет, для связи с машинами);
<br/> eth2 - `192.168.96.113` (смотрит в интернет), шлюз: `192.168.96.1/24`
<br/> Имеется дополнительно управляющий ноут: `192.168.87.136/24`.

Также есть две другие Ubuntu управляемые машины, подключённые к выше указанном роутеру с адресами: `192.168.56.2/24`; `192.168.56.3/24`.

На управляемых машинах требуется провести действия:
  - собрать факты об удалённых узлах;
  - установить python;
  - установить postgreql;
  - создать template postgresql.j2

См.: []()

-----------------------------------------------------------------------
## Ansible на роутере.

Для начала рассмотрим управление с роутера (`192.168.87.112/24`), установив на него предварительно Ansible.
Структура проекта будет выглядеть примерно так на начальном этапе:
```
~/.ansible/project/
├── ansible.cfg
├── inventory/
│   └── hosts.yml
├── main.yml
└── roles/
```

Создадим структура проекта:
```bash
mkdir -p ~/.ansible/project1/{inventory,group_vars,roles}
cd ~/.ansible/project
```

 Файл `~/.ansible/project1/inventory/hosts.ini`:
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

Файл `ansible.cfg`:
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
fact_caching_timeout = 86400

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

