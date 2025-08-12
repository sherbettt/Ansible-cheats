# Установка и настройка csync2 через Ad-Hoc Ansible команды


Создаём **ansible.cfg** файлик:
```ini
[defaults]
inventory = ./inventory/hosts.ini        # либо ~/.ansible/infra-proj/inventory/hosts.ini
roles_path = ./roles
private_key_file = ~/.ssh/id_rsa         # Путь к вашему приватному ключу
#private_key_file = ~/.ssh/id_ed25519
host_key_checking = False                # Отключает проверку SSH-ключей хостов
interpreter_python = /usr/bin/python3    # какой интерпретатор Python использовать на управляемых узлах.
```

Создаём **inventory/hosts.ini** файл:
```ini
[gateways]
dmzgateway1 ansible_host=192.168.87.253 ansible_user=root
dmzgateway2 ansible_host=192.168.87.252 ansible_user=root
dmzgateway3 ansible_host=192.168.87.251 ansible_user=root

[proxmox]
prox4 ansible_host=192.168.87.17 ansible_user=root
pmx5 ansible_host=192.168.87.20 ansible_user=root
pmx6 ansible_host=192.168.87.6 ansible_user=root
```

Дерево проекта:
```c
└─ ~/.ansible/infra-proj $ tree
.
├── ./{
├── ./altlinux-release.xterm
├── ./ansible.cfg
├── ./csync2.conf
├── ./csync2.key
├── ./fedora-release.xterm
├── ./os-release.xterm
├── ./redhat-release.xterm
├── ./system-release.xterm
├── ./group_vars
├── ./inventory
│   └── ./inventory/hosts.ini
├── ./playbooks
│   └── ./playbooks/play_csync.yml
└── ./roles
    └── ./roles/csync2
        ├── ./roles/csync2/files
        │   └── ./roles/csync2/files/cluster.key
        ├── ./roles/csync2/tasks
        │   └── ./roles/csync2/tasks/main.yml
        └── ./roles/csync2/templates
            ├── ./roles/csync2/templates/csync2.cfg.j2
            └── ./roles/csync2/templates/csync2.service.j2
```



### 1. Проверка подключения.
```bash
└─ ~/.ansible/infra-proj $ ansible gateways -m ping
dmzgateway2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
dmzgateway1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
dmzgateway3 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
  #-------

ansible gateways -m shell -a "nc -zv dmzgateway1 30865"
ansible gateways -m shell -a "nc -zv dmzgateway2 30865"
ansible gateways -m shell -a "nc -zv dmzgateway3 30865"
```

### 2. Добавим записи в `/etc/hosts` на всех узлах и проверим
```bash
ansible gateways -m blockinfile -a "path=/etc/hosts block='192.168.87.253 dmzgateway1
192.168.87.252 dmzgateway2
192.168.87.251 dmzgateway3'"
```
```bash
ansible gateways -m shell -a "ping -c 2 dmzgateway1"
ansible gateways -m shell -a "ping -c 2 dmzgateway2"
ansible gateways -m shell -a "ping -c 2 dmzgateway3"
```

### 2. Установка пакета csync2 на все шлюзы/хосты
```bash
ansible gateways -m apt -a "name=csync2 state=present update_cache=yes"
# или:
ansible gateways -i hosts -u root -k -m apt -a "name=csync2 state=present update_cache=yes"
```

### 3. Создание конфигурационных каталогов
```bash
ansible gateways -m file -a "path=/etc/csync2 state=directory mode=0750"
ansible gateways -m file -a "path=/etc/csync2/key.d state=directory mode=0750"

# или:
ansible gateways -i ~/.ansible/infra-proj/inventory/hosts.ini -u root -k -m file -a "path=/etc/csync2 state=directory mode=0750"
ansible gateways -i ~/.ansible/infra-proj/inventory/hosts.ini -u root -k -m file -a "path=/etc/csync2/key.d state=directory mode=0750"
```

### 4.1. Генерация ключа на dmzgateway1 и копирование на остальные узлы
```bash
# Генерация ключа на dmzgateway1
ansible dmzgateway1 -i inventory/hosts.ini  -m shell -a "csync2 -k /tmp/csync2.key"
ansible dmzgateway1 -i ~/.ansible/infra-proj/inventory/hosts.ini -u root -k -m shell -a "csync2 -k /tmp/csync2.key"

# Копирование с dmzgateway1 на локальную машину
ansible dmzgateway1 -i inventory/hosts.ini -m fetch -a "src=/tmp/csync2.key dest=./csync2.key flat=yes"
ansible dmzgateway1 -i ~/.ansible/infra-proj/inventory/hosts.ini -u root -k -m fetch -a "src=/tmp/csync2.key dest=./csync2.key flat=yes"

# Распространение ключа на все узлы
ansible gateways -i inventory/hosts.ini -m copy -a "src=./csync2.key dest=/etc/csync2/key.d/csync2.key mode=0600"
ansible gateways -i ~/.ansible/infra-proj/inventory/hosts.ini -u root -k -m copy -a "src=./csync2.key dest=/etc/csync2/key.d/csync2.key mode=0600"
```

### 4.2. Генерация ключа локальной машине (192.168.87.74) и копирование на остальные узлы
```bash
# Генерация ключа на локальной машине
csync2 -k ./csync2.key

# если Ximper Linux, то будет
sudo find /usr -name csync2
whereis csync2
/usr/sbin/csync2 -k ./csync2.key
chmod 600 ./csync2.key

# Копирование ключа на все узлы
ansible gateways -m copy -a "src=./csync2.key dest=/etc/csync2/key.d/csync2.key mode=0600"
#---------------------------------------------------
```


### 5. Создание конфигурационного файла `/etc/csync2/csync2.cfg`

```bash
ansible gateways -m copy -a "content='group cluster {
    host dmzgateway1(192.168.87.251);
    host dmzgateway2(192.168.87.252);
    host dmzgateway3(192.168.87.253);
    key /etc/csync2/key.d/csync2.key;
    include /etc/keepalived/keepalived.conf;
    include /etc/network/interfaces.d/*;
    action {
        pattern /etc/keepalived/keepalived.conf;
        exec /usr/sbin/service keepalived restart;
        logfile /var/log/csync2_action.log;
    }
}' dest=/etc/csync2/csync2.cfg mode=0640"
```
Или `/etc/csync2/csync2.cfg` без SSL сертификатов
```conf
nossl * *;
group cluster {
    host dmzgateway1;
    host dmzgateway2;
    host dmzgateway3;
    key /etc/csync2/key.d/csync2.key;
    include /etc/keepalived/keepalived.conf;
    include /etc/network/interfaces.d/*;
    action {
        pattern /etc/keepalived/keepalived.conf;
        exec /usr/sbin/service keepalived restart;
        logfile /var/log/csync2_action.log;
    }
}
```
Создание симлинка на нужный конф. файл:
```bash
ansible gateways -m shell -a "ln -sf /etc/csync2/csync2.cfg /etc/csync2.cfg"
```
Или перемещение конфига:
```bash
ansible gateways -m shell -a "mv /etc/csync2/csync2.cfg /etc/csync2.cfg"
```

### 5 [ALT]. Создание конфигурационного файла `/etc/csync2/csync2.cfg` с помощью  jinja
```jinja
nossl * *;
group cluster {
    host {{ ansible_hostname }};
    {% for host in groups['gateways'] %}
    {% if host != inventory_hostname %}
    host {{ host }};
    {% endif %}
    {% endfor %}

    key /etc/csync2/key.d/csync2.key;
    
    include /etc/keepalived/keepalived.conf;
    include /etc/keepalived/test_sync.txt;  # для проверки синхронизации
    include /etc/network/interfaces.d/*;
    
    action {
        pattern /etc/keepalived/keepalived.conf;
        exec /usr/sbin/service keepalived restart;
        logfile /var/log/csync2_action.log;
    }
}
```
Устанавливаем командой:
```bash
ansible gateways -m template -a \
"src=roles/csync2/templates/csync2.cfg.j2 \
dest=/etc/csync2.cfg \
owner=root group=root mode=0644" -b
```


### 6. Создание `/etc/systemd/system/csync2.service` юнита
```bash
ansible gateways -m copy -a \
"dest=/etc/systemd/system/csync2.service \
content='[Unit]
Description=csync2 cluster synchronization
After=network.target

[Service]
Type=simple
ExecStart=/usr/sbin/csync2 -D /var/lib/csync2 -ii -v
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target'" \
-b --diff
```
- **-m copy** - используем модуль для копирования файлов
- **dest=/etc/systemd/system/csync2.service** - путь назначения
- **content='...'** - содержимое файла (обратите внимание на многострочный формат)
- **-b** - выполнение с повышенными правами (sudo)
- **--diff** - покажет различия при изменении файла


т.е.
```ini
[Unit]
Description=csync2 cluster synchronization
After=network.target

[Service]
Type=simple
ExecStart=/usr/sbin/csync2 -D /var/lib/csync2 -ii -v
Restart=on-failure
RestartSec=5s
#
#Environment="CSYNC2_SYSTEM_DIR=/etc/csync2"
#ExecStart=/usr/sbin/csync2 -D /var/lib/csync2 -v

[Install]
WantedBy=multi-user.target
```
Возможно придётся руками задать переменную CSYNC2_SYSTEM_DIR и прописать в `/etc/systemd/system/csync2.service` юнит
```
ansible gateways -m shell -a "ls -l /etc/csync2/csync2.cfg"
ansible gateways -m shell -a "echo CSYNC2_SYSTEM_DIR=$CSYNC2_SYSTEM_DIR"
```


### 7. Включение и запуск службы
```bash
ansible gateways -m service -a "name=csync2 enabled=yes state=started"
# или:
ansible gateways -i hosts -u root -k -m service -a "name=csync2 enabled=yes state=started"
# или:
ansible gateways -m shell -a "systemctl enable csync2 --now"
ansible gateways -m shell -a "systemctl status csync2"
```

### 8. Инициализация синхронизации
```bash
ansible dmzgateway1 -m shell -a "csync2 -xv"
ansible dmzgateway1 -m shell -a "csync2 -N 192.168.87.251 -xvv"
# или:
ansible dmzgateway1 -i hosts -u root -k -m shell -a "csync2 -xv"
```

### 9. Тест синхронизации
```bash
# Создаем тестовый файл
ansible dmzgateway1 -m copy -a "content='test sync' dest=/etc/keepalived/test_sync.txt"

# Запускаем синхронизацию
ansible dmzgateway1 -m shell -a "csync2 -x"

# Проверяем на всех узлах
ansible gateways -m shell -a "ls -la /etc/keepalived/ | grep test_sync"
```

### 10. Проверка статуса
```bash
ansible gateways -m shell -a "systemctl status csync2"
ansible gateways -m shell -a "ls -la /etc/csync2"
# или:
ansible gateways -i hosts -u root -k -m shell -a "systemctl status csync2"
```

### 11. Дополнительные проверки
```bash
# Проверка логов
ansible gateways -m shell -a "tail -n 20 /var/log/csync2*"

# Проверка подключений
ansible gateways -m shell -a "netstat -tulpn | grep csync2"
```

### 12. Настроить правила iptables в директории `/etc/iptables/`
```bash
# Сначала сохраним текущие правила из /etc/iptables
ansible gateways -m shell -a "cp /etc/iptables /etc/iptables.rules.backup"

# Удалим файл /etc/iptables и создадим вместо него директорию
ansible gateways -m shell -a "rm -f /etc/iptables && mkdir -p /etc/iptables"

# Добавим правило для порта 30865 и сохраним всё в /etc/iptables/rules.v4
ansible gateways -m shell -a "iptables -A INPUT -p tcp --dport 30865 -j ACCEPT && iptables-save > /etc/iptables/rules.v4"

# Восстановим NAT-правила из бэкапа
ansible gateways -m shell -a "cat /etc/iptables.rules.backup >> /etc/iptables/rules.v4"

# Проверим, что всё сохранилось правильно
ansible gateways -m shell -a "cat /etc/iptables/rules.v4"

# Перезагрузим iptables (если нужно)
ansible gateways -m shell -a "iptables-restore < /etc/iptables/rules.v4"

# Проверим, что порт 30865 открыт
ansible gateways -m shell -a "iptables -L -n | grep 30865 || echo 'Порт не открыт'"
```



### 13. Создание SSL-сертификатов

#### Можно руками на мастер ноде руками создать и руками расзложить по нодам:
```bash
openssl req -x509 -newkey rsa:4096 -keyout csync2_ssl_key.pem -out csync2_ssl_cert.pem -days 3650 -nodes -subj "/CN=dmzgateway1";
# раскидываем:
for node in dmzgateway2 dmzgateway3; do
  scp /etc/csync2_ssl_{key,cert}.pem root@$node:/etc/
done
```
#### Либо попробуем развернуть сертификаты на все узлы через Ansible:
```bash
# Копируем ключ и сертификат
ansible gateways -m copy -a "src=csync2_ssl_key.pem dest=/etc/csync2_ssl_key.pem mode=0600"
ansible gateways -m copy -a "src=csync2_ssl_cert.pem dest=/etc/csync2_ssl_cert.pem mode=0644"

# Альтернативно, можно создать сертификаты прямо на узлах (если есть openssl):
ansible gateways -m shell -a "openssl req -x509 -newkey rsa:4096 -keyout /etc/csync2/csync2_ssl_key.pem -out /etc/csync2/csync2_ssl_cert.pem -days 3650 -nodes -subj '/CN={{ inventory_hostname }}'"
ansible gateways -m shell -a "chmod 600 /etc/csync2/csync2_ssl_key.pem; chmod 644 /etc/csync2/csync2_ssl_cert.pem"
```

#### Если ключи были уже, рекомендуется их удалить предварительно:
```bash
ansible gateways -m shell -a "rm -f /etc/csync2/key.d/csync2.key && csync2 -k /etc/csync2/key.d/csync2.key && chmod 600 /etc/csync2/key.d/csync2.key"
```
Исправить правила iptables:
```bash
ansible gateways -m shell -a "rm -f /etc/iptables && mkdir -p /etc/iptables"
ansible gateways -m shell -a "iptables -A INPUT -p tcp --dport 30865 -j ACCEPT && iptables-save > /etc/iptables/rules.v4"
```

#### Ad-Hoc командой создаём сертификаты на всех узлах:
```bash
ansible gateways -m shell -a "openssl req -x509 -newkey rsa:4096 -keyout /etc/csync2_ssl_key.pem -out /etc/csync2_ssl_cert.pem -days 3650 -nodes -subj '/CN={{ inventory_hostname }}'"
```
Права:
```bash
ansible gateways -m shell -a "chmod 600 /etc/csync2_ssl_key.pem; chmod 644 /etc/csync2_ssl_cert.pem"
```


Обновляем конфигурацию **csync2**:
```conf
ansible gateways -m copy -a "content='group cluster {
    host (dmzgateway1 dmzgateway2 dmzgateway3);
    key /etc/csync2/key.d/csync2.key;
    include /etc/keepalived/keepalived.conf;
    ssl_cert /etc/csync2/csync2_ssl_cert.pem;
    ssl_key /etc/csync2/csync2_ssl_key.pem;
}' dest=/etc/csync2/csync2.cfg mode=0640"
```
```bash
ansible gateways -m shell -a "systemctl restart csync2"
ansible gateways -m shell -a "csync2 -xv"
```
```bash
ansible gateways -m shell -a "grep host /etc/csync2/csync2.cfg"
```


### 14. Настройка автоматической синхронизации (опционально)
```bash
# Синхронизация Каждые 5 минут
ansible gateways -m cron -a "name='csync2 sync' minute='*/5' job='/usr/sbin/csync2 -x'"

# Синхронизация Каждые 10 минут
ansible gateways -m cron -a "name='csync2 sync' minute='*/10' job='/usr/sbin/csync2 -x'"

# Синхронизация Каждые 30 минут
ansible gateways -m cron -a "name='30-min csync2 sync' minute='*/30' job='/usr/sbin/csync2 -x'"

# Синхронизация каждый час (в 0 минут каждого часа)
ansible gateways -m cron -a "name='Hourly csync2 sync' minute='0' job='/usr/sbin/csync2 -x'"

# Синхронизация каждые 4 часа (в 0 минут каждые 4 часа)
ansible gateways -m cron -a "name='4-hour csync2 sync' minute='0' hour='*/4' job='/usr/sbin/csync2 -x'"

## Удаление cron-заданий (если нужно)
# Удалить ежечасную синхронизацию
ansible gateways -m cron -a "name='Hourly csync2 sync' state=absent"

# Удалить синхронизацию каждые 4 часа
ansible gateways -m cron -a "name='4-hour csync2 sync' state=absent"
```
Проверка cron триггеров:
```
ansible gateways -m shell -a "crontab -l | grep csync2"
```


### Ключевые особенности:
1. Все команды используют SSH-ключ (как настроено в ansible.cfg)
2. Ключ генерируется один раз локально
3. Конфигурация применяется одинаково на всех узлах
4. Не требуется ввод паролей
5. Для аутентификации используется пароль root (`-k` флаг)
6. Ключ генерируется один раз на dmzgateway1 и распространяется на остальные узлы

### Дополнительные проверки:
```bash
# Проверка синхронизированных файлов
ansible gateways -i hosts -u root -k -m shell -a "ls -la /etc/csync2"

# Проверка логов
ansible gateways -i hosts -u root -k -m shell -a "tail -n 20 /var/log/csync2*"
```

--------------------------------------------------------
### Если xinetd уже занимает порт 30865:
```bash
# Проверка порта
ansible gateways -m shell -a "netstat -tulpn | grep 30865 || ss -tulpn | grep 30865"

# Остановим xinetd (так как csync2 должен работать самостоятельно)
ansible gateways -m shell -a "systemctl stop xinetd && systemctl disable xinetd"

# Убедимся, что порт освободился
ansible gateways -m shell -a "ss -tulpn | grep 30865 || echo 'Port 30865 is free'"
```
```bash
# Перезапустим csync2
ansible gateways -m shell -a "systemctl restart csync2"

# Проверим статус
ansible gateways -m shell -a "systemctl status csync2"
ansible gateways -m shell -a "ss -tulpn | grep csync2 || echo 'csync2 not listening'"

# Инициализируем синхронизацию
ansible dmzgateway1 -m shell -a "/usr/sbin/csync2 -xvv"
```

#### Вариант A: Настроим csync2 работать через xinetd
```bash
# 1. Включим xinetd обратно
ansible gateways -m shell -a "systemctl enable xinetd --now"

# 2. Настроим конфиг xinetd для csync2
ansible gateways -m copy -a "content='service csync2
{
    disable = no
    socket_type = stream
    protocol = tcp
    wait = no
    user = root
    server = /usr/sbin/csync2
    server_args = -i
    only_from = 192.168.87.0/24
}' dest=/etc/xinetd.d/csync2 mode=0640"

# 3. Перезапустим xinetd
ansible gateways -m shell -a "systemctl restart xinetd"
```

#### Вариант B: Изменим порт csync2 (если нужно оставить xinetd для других сервисов)
```bash
# В конфиге /etc/csync2/csync2.cfg добавьте:
ansible gateways -m lineinfile -a "path=/etc/csync2/csync2.cfg line='    port 30866;' insertafter='group cluster'"

# Затем перезапустите:
ansible gateways -m shell -a "systemctl restart csync2"
```

#### Важно:
1. Выберите только один вариант (A или B)
2. После настройки проверьте:
```bash
ansible gateways -m shell -a "csync2 -T"  # Проверка конфигурации
ansible gateways -m shell -a "csync2 -xv" # Тест синхронизации
```
Рекомендуется **Вариант A** (использовать xinetd), так как это стандартный подход в ALT Linux для csync2.

--------------------------------------------------------
### Если openbsd-inetd (простой inetd) уже занимает порт 30865:
### 1. Остановим inetd и освободим порт 30865:
```bash
ansible gateways -m shell -a "systemctl stop openbsd-inetd && systemctl disable openbsd-inetd"
```

### 2. Проверим освобождение порта:
```bash
ansible gateways -m shell -a "ss -tulpn | grep 30865 || echo 'Port 30865 is free'"
```

### 3. Перезапустим csync2:
```bash
ansible gateways -m shell -a "systemctl restart csync2"
```

### 4. Проверим работу csync2:
```bash
ansible gateways -m shell -a "systemctl status csync2"
ansible gateways -m shell -a "ss -tulpn | grep csync2"
```

### 5. Инициализируем синхронизацию:
```bash
ansible dmzgateway1 -m shell -a "/usr/sbin/csync2 -xv"  # как бы на одной ноде делается
ansible gateways -m shell -a "/usr/sbin/csync2 -xv"
```

### Если нужно оставить inetd для других сервисов:

#### Вариант A: Настроим csync2 через inetd
```bash
# 1. Добавим конфиг для inetd
ansible gateways -m copy -a "content='30865 stream tcp nowait root /usr/sbin/csync2 csync2 -i' dest=/etc/inetd.conf mode=0640"

# 2. Перезапустим inetd
ansible gateways -m shell -a "systemctl restart openbsd-inetd"
```

#### Вариант B: Изменим порт csync2
```bash
# 1. В конфиге csync2 добавим порт
ansible gateways -m lineinfile -a "path=/etc/csync2/csync2.cfg line='    port 30866;' insertafter='group cluster'"

# 2. Перезапустим csync2
ansible gateways -m shell -a "systemctl restart csync2"
```

### Проверка после настройки:
```bash
ansible gateways -m shell -a "csync2 -T"  # Проверка конфигурации
ansible gateways -m shell -a "csync2 -xv" # Тест синхронизации
```
Рекомендуется **Вариант A** (настройка через inetd), так как это стандартный подход в системах с openbsd-inetd. Это обеспечит правильную работу csync2 без конфликтов портов.
<br/>
<br/>




# IOSTAT хостов
```bash
┌─ kirill kiko0217
└─ ~/.ansible/infra-proj $ ansible gateways -m shell -a "uptime" -b
dmzgateway1 | CHANGED | rc=0 >>
 14:59:07 up  3:36,  2 users,  load average: 18,40, 21,04, 22,14
dmzgateway2 | CHANGED | rc=0 >>
 14:59:07 up  3:50,  3 users,  load average: 13,69, 10,78, 8,52
dmzgateway3 | CHANGED | rc=0 >>
 14:59:07 up 4 days,  1:23,  2 users,  load average: 4,24, 4,32, 4,28
```
Видим прроблему на dmzgateway1. Смотрим iostat хоста.
```bash
┌─ kirill kiko0217
└─ ~/.ansible/infra-proj $ ansible dmzgateway1 -m shell -a "top -b -n 1 | head -n 15" -b
dmzgateway1 | CHANGED | rc=0 >>
top - 14:58:45 up  3:36,  2 users,  load average: 21,85, 21,89, 22,44
Tasks:  30 total,   1 running,  29 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0,0 us,  0,0 sy,  0,0 ni,100,0 id,  0,0 wa,  0,0 hi,  0,0 si,  0,0 st 
MiB Mem :   1024,0 total,    286,5 free,     78,6 used,    659,1 buff/cache     
MiB Swap:      0,0 total,      0,0 free,      0,0 used.    945,4 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
      1 root      20   0  102628  12288   8960 S   0,0   1,2   0:03.78 systemd
     56 root      20   0  135440  80884  79860 S   0,0   7,7   0:06.93 systemd+
     73 systemd+  20   0   17908   8448   7424 S   0,0   0,8   0:00.02 systemd+
    169 _rpc      20   0    7884   3584   3328 S   0,0   0,3   0:00.00 rpcbind
    172 root      20   0    6612   2304   2304 S   0,0   0,2   0:00.00 cron
    174 message+  20   0    9264   4608   4096 S   0,0   0,4   0:00.82 dbus-da+
    180 root      20   0   29000   9472   8192 S   0,0   0,9   0:00.00 keepali+
    191 root      20   0   17228   7936   6912 S   0,0   0,8   0:00.36 systemd+
```













