Ad-hoc-команды позволяют быстро выполнить одну операцию на одном или нескольких удалённых серверах без написания плейбуков. Они полезны для простых действий вроде установки пакетов, перезагрузки машин, копирования файлов и других однотипных задач.

Вот несколько примеров ad-hoc-команд:

---

### 📌 Установка пакета
Установить пакет `nginx` на всех машинах группы `[webservers]`:
```bash
ansible webservers -m apt -a "name=nginx state=present update_cache=true"
```

Или установить программу `htop` на CentOS/RedHat-машинах:
```bash
ansible webservers -b -m yum -a "name=htop state=latest"
```

---

### 📌 Запуск службы
Перезапустить службу Nginx на группе серверов:
```bash
ansible webservers -m service -a "name=nginx state=restarted"
```

---

### 📌 Проверка доступности служб
Проверить состояние Apache HTTPD:
```bash
ansible all -m wait_for -a "path=/var/run/httpd.pid timeout=30 port=80"
```

---

### 📌 Управление файлами
Создать директорию `/data/applogs` на всех серверах:
```bash
ansible all -m file -a "path=/data/applogs state=directory mode='u=rwx,g=rx,o=rx'"
```

---

### 📌 Копирование файла
Скопировать файл конфига на удалённый сервер:
```bash
ansible appserver -m copy -a "src=config.ini dest=/etc/myapp/config.ini owner=root group=root mode=0644"
```

---

### 📌 Выполнение произвольных команд
Выполнить команду непосредственно на сервере:
```bash
ansible dbservers -a "/usr/bin/mysqladmin ping"
```

---

### 📌 Работа с пользователями
Создать нового пользователя на машине:
```bash
ansible localhost -m user -a "name=testuser comment='Test User' shell=/bin/bash createhome=yes"
```

Удалить пользователя:
```bash
ansible all -m user -a "name=testuser state=absent remove=yes force=no"
```

---

### 📌 Собрать информацию о сети
Получить список IP-адресов сервера:
```bash
ansible target_server -m debug -a "msg={{ hostvars['target_server']['ansible_all_ipv4_addresses'] }}"
```
---

Команда `ansible -m setup` используется для запуска модуля Ansible под названием **setup**, который собирает факты (информацию) о хосте, такие как операционная система, версия ядра, сетевые интерфейсы, диски и другое.

## Основные моменты команды

### Формат команды
```bash
ansible <host_pattern> -m setup [-a args]
```

- `<host_pattern>` — шаблон хоста или группа серверов, заданная в инвентори-файле (`hosts`) или динамически, например IP-адрес сервера.
- `-m setup` — запуск модуля `setup`, собирающего информацию о целевом узле.
- `[-a args]` — дополнительные аргументы, передаваемые модулю.

### Как выполняется команда?

При выполнении данной команды Ansible подключается к указанному хосту, проверяет наличие установленного Python-интерпретатора и запускает модуль сбора фактов. После завершения процесса выводит собранную информацию в удобочитаемом виде.

### Примеры вывода команд

Пример простого запуска:
```bash
$ ansible all -i hosts -m setup
server1 | SUCCESS => {
    "ansible_facts": {
        "ansible_architecture": "x86_64",
        "ansible_distribution": "Ubuntu",
        "ansible_domain": "",
        "ansible_fqdn": "server1.example.com",
        ...
    },
    "changed": false,
    "_ansible_no_log": false
}
```

Можно также фильтровать вывод командой, используя аргумент `filter`. Например, вывести только информацию о сетевых устройствах:
```bash
$ ansible server1 -m setup -a 'filter=ansible_eth*'
```

Таким образом, эта команда полезна для быстрого получения подробной информации о серверах перед выполнением более сложных операций конфигурации или мониторинга.
