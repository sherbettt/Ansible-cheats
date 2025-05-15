Ad-Hoc команды в Ansible — это простой способ выполнить команду на удаленных хостах без написания полноценного плейбука. Они позволяют быстро тестировать конфигурационные изменения, проверять статус системы или запускать простые команды.


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




### Базовые команды

```bash
ansible all -m shell -a 'ls /tmp'
```
**Описание:** Выполняет команду `ls` в директории `/tmp` на всех хоста.

---

### Использование модуля `command`

```bash
ansible webservers -m command -a '/usr/bin/uptime'
```
**Описание:** Запускает команду `/usr/bin/uptime` на хостах группы `webservers`.

---

### Проверка файлов

```bash
ansible dbservers -m stat -a 'path=/var/lib/mysql'
```
**Описание:** Проверяет существование каталога `/var/lib/mysql` на хостах группы `dbservers`.

---

### Установка пакетов

```bash
ansible all -m yum -a 'name=httpd state=latest'
```
**Описание:** Устанавливает пакет `httpd` последней версии на всех хостах.

---

### Удаление пакетов

```bash
ansible servers -m apt -a 'name=nginx state=absent'
```
**Описание:** Удаляет пакет `nginx` с хостов группы `servers`.

---

### Создание файлов

```bash
ansible nodes -m copy -a 'content="Hello World" dest=/tmp/greeting.txt'
```
**Описание:** Создает файл `/tmp/greeting.txt` с содержимым `"Hello World"` на всех хостах группы `nodes`.

---

### Переименование файлов

```bash
ansible websrvs -m file -a 'src=/var/www/html/index.html dest=/var/www/html/old-index.html state=link'
```
**Описание:** Переименовывает файл `/var/www/html/index.html` в `/var/www/html/old-index.html`, сохраняя ссылку на исходный файл.

---

### Отправка файлов

```bash
ansible apps -m scp -a 'src=/home/user/myfile.tar.gz dest=/opt/apps/'
```
**Описание:** Копирует локально расположенный архив `/home/user/myfile.tar.gz` на все хосты группы `apps` в директорию `/opt/apps/`.

---

### Загрузка файлов с удаленного сервера

```bash
ansible hosts -m fetch -a 'src=/etc/passwd dest=/local/backup/'
```
**Описание:** Скачивает файл `/etc/passwd` с каждого хоста группы `hosts` в локальную директорию `/local/backup/`.

---

### Выключение серверов

```bash
ansible prod_servers -m shell -a 'shutdown now' --ask-pass
```
**Описание:** Отключает каждый сервер из группы `prod_servers`. После запуска потребуется ввести пароль суперпользователя.

---

### Уведомление администраторов

```bash
ansible all -m user -a 'name=admin notify=true'
```
**Описание:** Добавляет пользователя `admin` ко всем хостам и отправляет уведомление о завершении операции.

---

### Работа с правами пользователей

```bash
ansible devmachines -m user -a 'name=john groups=wheel append=true'
```
**Описание:** Добавляет пользователя `john` в группу `wheel` на всех машинах группы `devmachines`.

---

### Автоматическое создание каталогов

```bash
ansible app_hosts -m file -a 'path=/data/app state=directory mode=0755 owner=app_user group=app_group'
```
**Описание:** Создает каталог `/data/app` с указанными правами доступа и владельцем.

---

### Удаление пустых каталогов

```bash
ansible webhost -m file -a 'path=/var/www/emptydir state=absent'
```
**Описание:** Удаляет пустой каталог `/var/www/emptydir` на хостах группы `webhost`.

---

### Асинхронная команда (Background)

```bash
ansible servers -b -a 'yum update -y' --forks=10 --background=10
```
**Описание:** Запускает обновление пакетов командой `yum update -y` асинхронно на всех серверах группы `servers`, одновременно используя 10 потоков (`--forks=10`) и проверяя выполнение каждые 10 секунд (`--background=10`).

---

### Вывод результатов команды в JSON

```bash
ansible control_nodes -m setup | json_pp
```
**Описание:** Запрашивает данные о конфигурации хостов и выводит их в удобочитаемом JSON формате.

---

