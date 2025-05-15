### Инструкции

1. Как пользователь *`student`* на рабочей станции создайте файл инвентаризации `/home/student/review-playbooks/inventory`, содержащий сервер `serverc.lab.example.com` в группе `ftpclients` и сервера `serverb.lab.example.com` и `serverd.lab.example.com` в группе `ftpservers`.

2. Создайте конфигурационный файл Ansible `/home/student/review-playbooks/ansible.cfg` и заполните его необходимыми параметрами для соответствия следующим требованиям:
   • Настройте проект Ansible на использование вновь созданного файла инвентаризации.
   • Подключайтесь к управляемым хостам как пользователь `devops`.
   • Используйте повышение привилегий с помощью команды `sudo` от имени пользователя `root`.
   • Повышайте привилегии по умолчанию для каждой задачи.

3. Создайте плейбук `/home/student/review-playbooks/ftpclients.yml`, содержащий сценарий (плей), направленный на хосты группы инвентаризационного файла `ftpclients`. Убедитесь, что пакет `lftp` установлен.

4. Поместите предоставленный шаблон конфигурации vsftpd — файл `vsftpd.conf.j2` — в подпапку `templates`.

5. Поместите предоставленный файл шаблона переменных `defaults-template.yml` в подпапку `vars`.

6. Создайте файл определения переменных `vars.yml` в подпапке `vars`, определяющий следующие три переменные и их значения:
   
| Переменная          | Значение               |
|---------------------|------------------------|
| `vsftpd_package`    | `vsftpd`               |
| `vsftpd_service`    | `vsftpd`               |
| `vsftpd_config_file`| `/etc/vsftpd/vsftpd.conf`|

7. Используя ранее созданные файлы шаблона Jinja2 и определения переменных, создайте второй плейбук `/home/student/review-playbooks/ansible-vsftpd.yml`, предназначенный для настройки службы vsftpd на серверах группы инвентаря `ftpservers`.

8. Создайте третий плейбук `/home/student/review-playbooks/site.yml`, включающий сценарии (плэйзы) из двух предыдущих плейбоков (`ftpclients.yml` и `ansible-vsftpd.yml`).

9. Запустите плейбук `/home/student/review-playbooks/site.yml`, чтобы убедиться, что он успешно выполняет требуемые задачи на управляемых узлах.

----------

### Решение задачки по Ansible

#### Шаг подготовительный/промежуточный
```bash
ansible -m ping all
ansible -i inventory all -m ping
ansible -i inventory all -a "whoami"
ansible -i inventory ftpservers -b -a "firewall-cmd --list-all --zone=public" --become-method=sudo
ansible -i inventory ftpservers -b -a "firewall-cmd --list-services --zone=public" --become-method=sudo
```

#### Шаг 1: Создание инвентарного файла (`inventory`)
Создаем файл `/home/student/review-playbooks/inventory` следующего содержания:
```bash
[ftpclients]
serverc.lab.example.com

[ftpservers]
serverb.lab.example.com
serverd.lab.example.com
```

#### Шаг 2: Настройка конфигурационного файла `ansible.cfg`
Создаем файл `/home/student/review-playbooks/ansible.cfg`, содержащий следующие настройки:
```ini
[defaults]
inventory = ./inventory
remote_user = devops

[privilege_escalation]
become = True
become_method = sudo
become_user = root
```

#### Шаг 3: Создание плейбука для установки пакета lftp (`ftpclients.yml`)
Создаем файл `/home/student/review-playbooks/ftpclients.yml` с таким содержимым:
```yaml
---
- name: Install lftp package on FTP clients
  hosts: ftpclients
  become: true
  tasks:
    - name: Ensure lftp package is installed
      yum:
        name: lftp
        state: present
```

#### Шаг 4: Перемещение шаблона конфигурации (`vsftpd.conf.j2`) в каталог `templates`
Перемещаем существующий шаблон `vsftpd.conf.j2` в подпапку `./templates`.
```bash
mkdir -p templates && mv vsftpd.conf.j2 templates/
```

#### Шаг 5: Перемещение файла `defaults-template.yml` в папку `vars`
Перемещаем файл `defaults-template.yml` в директорию `./vars`.
```bash
mkdir -p vars && mv defaults-template.yml vars/
```

#### Шаг 6: Определение переменных в файле `vars.yml`
Создаем файл `/home/student/review-playbooks/vars/vars.yml` с такими переменными:
```yaml
---
vsftpd_package: vsftpd
vsftpd_service: vsftpd
vsftpd_config_file: "/etc/vsftpd/vsftpd.conf"
```
убедиться, что текущий контекст соответствует ожидаемому типу с помощью команды: 
`ansible -i inventory ftpservers -b -a "ls -Z /etc/vsftpd/vsftpd.conf" --become-method=sudo`

#### Шаг 7: Плейбук для конфигурирования сервиса vsftpd (`ansible-vsftpd.yml`)
Создаем второй плейбук `/home/student/review-playbooks/ansible-vsftpd.yml` следующим образом:
```yaml
---
- name: FTP server is installed
  become: true
  hosts:
   - ftpservers
  vars_files:
   - vars/defaults-template.yml
   - vars/vars.yml
  tasks:
   - name: Packages are installed
     yum:
       name: "{{ vsftpd_package }}"
       state: present
# https://docs.ansible.com/ansible/latest/collections/ansible/builtin/host_group_vars_vars.html -> vars_files
# https://docs.ansible.com/ansible/latest/reference_appendices/config.html -> vars_files


   - name: Ensure service is started
     service:
       name: "{{ vsftpd_service }}"
       state: started
       enabled: true

   - name: Configuration file is installed
     template:
       src: templates/vsftpd.conf.j2
       dest: "{{ vsftpd_config_file }}"
       owner: root
       group: root
       mode: '0600'
       setype: etc_t
     notify: restart vsftpd
# https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html#ansible-collections-ansible-builtin-template-module

   - name: firewalld is installed
     yum:
       name: firewalld
       state: present
# https://docs.ansible.com/ansible/latest/collections/ansible/builtin/dnf_module.html#ansible-collections-ansible-builtin-dnf-module

   - name: firewalld is started and enabled
     service:
       name: firewalld
       state: started
       enabled: yes

   - name: FTP port is open
     firewalld:
       service: ftp
       permanent: true
       state: enabled
       immediate: yes

# Поскольку порты мы открывааем первый раз и, чтобы после не было проблем с портами после повторного запуска плейбука и его блокировки,
# требуется сделать проверку через директиву when (хотя и не обязательно)
   - name: FTP passive data ports are open
     firewalld:
       port: 21000-20/tcp  # port: 21000-21020/tcp
       permanent: yes
       state: enabled
       immediate: yes
#     when: "'21000-20/tcp' not in firewalled_ports.stdout"  # проверки существования порта
#     when: >-
#       (
#         ansible_firewalld.query('port', '21000-20/tcp') != 'enabled'
#         or
#         ansible_firewalld.query('port', '21000-20/tcp', 'public') != 'enabled'
#       )

# https://docs.ansible.com/ansible/latest/collections/ansible/posix/firewalld_module.html

# Длинный вариант с проверкой портов - disabled/enabled
#  - name: Remove existing rule
#    firewalld:
#      port: 21000-20/tcp
#      permanent: yes
#      state: disabled
#      immediate: yes

#  - name: Add new rule
#    firewalld:
#      port: 21000-20/tcp
#      permanent: yes
#      state: enabled
#      immediate: yes


  handlers:
   - name: restart vsftpd
     service:
       name: "{{ vsftpd_service }}"
       state: restarted
# https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html#ansible-collections-ansible-builtin-service-module
```
сбросить состояние Firewall портов
```
# Удаление существующего правила `(state=disabled)`. Удаляет правило брандмауэра для диапазона портов 21000–20 по протоколу TCP, применяя изменения немедленно и делая их постоянными.
ansible all -m firewalld -a "port=21000-20/tcp permanent=true state=disabled immediate=true"

# Создание нового правила `(state=enabled)`. Добавляет новое правило брандмауэра для тех же портов, разрешая доступ и также сохраняя изменения постоянно и применяя их немедленно.
ansible all -m firewalld -a "port=21000-20/tcp permanent=true state=enabled immediate=true"
```

#### Шаг 8: Объединение плейбоков в единый плейбук (`site.yml`)
Создаем основной плейбук `/home/student/review-playbooks/site.yml`, включающий оба ранее созданных плейбука:
```yaml
---
# FTP Servers playbook
- import_playbook: ftpclients.yml

# FTP Clients playbook
- import_playbook: ansible-vsftpd.yml
```

#### Шаг 9: Запуск плейбуков для проверки
Запустим плейбук командой:
```bash
cd ~/review-playbooks
ansible-playbook site.yml
```


При успешном выполнении мы увидим вывод примерно такой формы:
```
PLAY RECAP *********************************************************************
serverb.lab.example.com : ok=3 changed=1 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0   
serverc.lab.example.com : ok=1 changed=1 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0   
serverd.lab.example.com : ok=3 changed=1 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0  
```

Таким образом, проверяется установка пакетов и настройка сервисов на целевых хостах согласно заданному плану.

---------------------------
кое какие пояснения.

Директива `setype: etc_t` используется для определения типа контекста SELinux для файла, создаваемого модулем `template` в Ansible. Давайте детально разберём причины выбора именно этого типа и почему не рекомендуется использовать `setype: _default`.

### Причины выбора `setype: etc_t`

1. **Типичный контекст для конфигурационных файлов:**
   - Каталог `/etc` традиционно хранит конфигурационные файлы различных сервисов и приложений. Поэтому для большинства конфигурационных файлов в SELinux предусмотрен специальный тип контекста — `etc_t`. Использование этого типа гарантирует, что ваши файлы будут защищены соответствующими правилами SELinux, применимыми ко всему каталогу `/etc`.

2. **Совместимость с существующей структурой правил SELinux:**
   - Когда вы создаёте новый файл в `/etc`, операционная система ожидает, что он будет помечен соответствующим контекстом SELinux. Установка правильного типа контекста предотвращает появление предупреждений или отказов в доступе со стороны SELinux.

3. **Поддерживаемый стандарт для SElinux:**
   - Type `etc_t` широко используется и поддерживается в системах с активированным SELinux. Он упрощает управление файлом, позволяя сохранять стандартные политики безопасности, разработанные специально для конфигурационных файлов.

4. **Соответствие документации Ansible:**
   Согласно официальной документации Ansible (), тип контекста позволяет явно задать уровень детализации управления доступом. Использование заранее известного типа значительно облегчает жизнь администратору, избавляя от необходимости вручную настраивать дополнительные правила SELinux.

### Почему не подходит `setype: _default`?

- `_default` обозначает автоматическое определение типа контекста SELinux. Хотя это удобно, иногда автоматика может ошибаться или выбрать неподходящий тип, особенно если существуют нестандартные политики SELinux. Таким образом, использование фиксированного значения типа контекста повышает надёжность и предсказуемость поведения системы.

- Важность точной настройки контекста особенно критична для конфигурационных файлов, таких как `/etc/vsftpd/vsftpd.conf`, которые влияют на важные сервисы, такие как FTP-сервер. Неправильно установленный контекст может привести к сбоям в работе сервиса или проблемам с безопасностью.

### Пример использования в плейбуке:

Ваш предыдущий плейбук корректно включает параметр `setype: etc_t`:

```yaml
tasks:
- name: Configuration file is installed
  template:
    src: templates/vsftpd.conf.j2
    dest: "{{ vsftpd_config_file }}"
    owner: root
    group: root
    mode: '0600'
    setype: etc_t
```

Это гарантирует, что созданный файл будет отмечен правильным контекстом SELinux, гарантирующим правильное поведение в среде с активным SELinux.

Итак, выбор `setype: etc_t` обоснован общепринятыми рекомендациями и требованиями безопасности, тогда как использование `_default` снижает управляемость и прозрачность работы системы.

------------------------------

Узнать доступные значения `setype` (типы контекста SELinux) можно несколькими способами:

### Способ №1: Список существующих типов контекста

Используя команду `semanage fcontext`, вы можете увидеть полный список зарегистрированных типов файлов в вашей системе SELinux:

```bash
semanage fcontext -l
```

Эта команда перечислит все зарегистрированные типы и пути, которые используют эти типы. Она даст представление обо всём спектре возможных значений `setype`, используемых в системе.

Например, некоторые часто встречающиеся типы:

- `etc_t` — тип для конфигурационных файлов в `/etc`.
- `var_log_t` — тип для журналов и лог-файлов в `/var/log`.
- `httpd_sys_content_t` — тип для документов веб-сервера.
- `ssh_home_t` — тип для личных файлов SSH.

### Способ №2: Получение контекста файла

Также можно посмотреть конкретный тип контекста отдельного файла или каталога, используя команду `ls -Z`:

```bash
ls -Z /etc/vsftpd/vsftpd.conf
```

Результатом будет строка вида:

```
-rw-------. root root system_u:object_r:etc_t:s0 /etc/vsftpd/vsftpd.conf
```

Здесь `etc_t` — это используемый тип контекста для данного файла.

### Способ №3: Консультация руководства SELinux

Можно обратиться к руководству SELinux вашего дистрибутива (например, Red Hat Enterprise Linux или CentOS). Документация доступна онлайн или локально через страницы справочной системы (`man selinux_types`).

Например, документация SELinux Red Hat подробно объясняет назначение и применение разных типов файлов.

### Полезные советы:

- Для создания нового типа контекста или добавления новых путей к уже существующему типу можно воспользоваться командой `semanage fcontext -a -t TYPE PATH`.
- Изменить контекст одного файла можно командой `chcon -t NEW_TYPE FILE_PATH`.

Таким образом, зная правильные типы контекста, вы сможете эффективно защищать файлы и ресурсы в своей системе с помощью SELinux.

------------------------------

Давай разберём, откуда берётся конструкция условия `when: "'21000-20/tcp' not in firewalled_ports.stdout"`.

### Поэтапное объяснение:

1. **Регистрация результатов выполнения команды (`register:`)**:
   Мы использовали команду `command: firewall-cmd --list-all --zone=public`, которая выводит список всех открытых портов в публичной зоне FirewallD. Эти результаты сохраняются в переменной `firewalled_ports`.

2. **Структура переменной `firewalled_ports`**:
   Результат выполнения команды сохраняется в виде простого текста в атрибутах объекта `firewalled_ports`. Атрибут `.stdout` содержит сам вывод команды.

3. **Анализ вывода команды**:
   Если порт открыт, он появится в выводе команды. Вывод может выглядеть примерно так:

   ```
   21000-20/tcp
   80/tcp
   ...
   ```

4. **Формулировка условия (`when:`)**:
   Нам нужно пропустить дальнейшие действия, если нужный порт уже присутствует в списке. Вот тут вступает оператор проверки включения строки:

   ```yaml
   when: "'21000-20/tcp' not in firewalled_ports.stdout"
   ```

   Эта конструкция означает следующее:
   - Берётся стандартный вывод предыдущей команды (`firewalled_ports.stdout`).
   - Проводится проверка на отсутствие строки `'21000-20/tcp'` среди этого вывода.
   - Если строка отсутствует, условие выполняется и открывается порт. Если строка уже есть, пропускаем дальнейшее выполнение задачи.

### Почему именно такая формулировка?
- Символьные одинарные кавычки вокруг значения (`'21000-20/tcp'`) нужны, чтобы точно сопоставлять строку. Иначе Ансибл мог бы интерпретировать выражение неверно.
- Конструкция `"not in"` проверяет отсутствие конкретной строки в результате вывода команды.

### Чем плоха альтернативная логика?
Альтернатива, вроде проверки наличия какого-нибудь другого элемента в результатах (например, другого порта), нарушила бы логику нашей цели — нам важно следить конкретно за указанным портом (`21000-20/tcp`).

Таким образом, данная конструкция условий оптимальна и удобна для быстрого обнаружения уже открытых портов и предотвращения повторных попыток открытия.

----------------

### Вопрос 1: На каком этапе нужно выполнять команду?

Команду `ansible -i inventory ftpservers -b -a "firewall-cmd --list-all --zone=public" --become-method=sudo` нужно выполнять на подготовительном этапе диагностики или тестирования плейбука, когда необходимо быстро выяснить, какие порты и зоны открыты в FirewallD на серверах группы `ftpservers`.

Цель этой команды — вывести весь список активных зон и правил FirewallD, чтобы понимать текущее состояние брандмауэра. Затем, основываясь на полученных данных, можно принять решение о дальнейших действиях.

Рекомендуемый этап выполнения:
- До начала основного плейбука.
- Во время предварительного тестирования или разработки плейбука.

### Вопрос 2: Как ты понял, что результаты сохраняются в переменную `firewalled_ports`?

Значение переменной `firewalled_ports` устанавливается посредством инструкции `register:` в Ansible. Рассмотрим детальнее:

```yaml
- name: Check if port is already opened
  command: firewall-cmd --list-all-ports --zone=public
  register: firewalled_ports
  changed_when: false
```

Инструкция `register: firewalled_ports` сохраняет вывод команды `firewall-cmd --list-all-ports --zone=public` в переменную `firewalled_ports`. Так устроен механизм регистрации результатов выполнения команд в Ansible.

### Вопрос 3: Где и как посмотреть содержимое переменной `firewalled_ports.stdout`?

Для просмотра переменной `firewalled_ports.stdout` можно сделать следующее:

1. Добавьте дополнительную задачу для вывода этой переменной в плейбук:

```yaml
- name: Display contents of firewalled_ports.stdout
  debug:
    msg: "{{ firewalled_ports.stdout }}"
```

Эта задача использует модуль `debug`, который выводит содержимое переменной в консоль при выполнении плейбука.

2. Можно посмотреть результаты прямо в выводе терминала при запуске плейбука. Если ваша структура плейбука выглядит так:

```yaml
- name: List all public zone ports
  command: firewall-cmd --list-all-ports --zone=public
  register: firewalled_ports
  changed_when: false

- name: Display contents of firewalled_ports.stdout
  debug:
    msg: "{{ firewalled_ports.stdout }}"
```

То при запуске плейбука вы увидите результат в таком виде:

```
ok: [serverb.lab.example.com] =>
  msg: "21000-20/tcp\n80/tcp\n..."
```

Эти действия позволят наглядно видеть, какие порты открыты в публичной зоне FirewallD.

### Резюме:

- Команду лучше запускать на этапе подготовки или предварительной диагностики.
- Результаты сохраняются в переменную через инструкцию `register:`.
- Просмотреть содержимое переменной можно с помощью модуля `debug` в самом плейбуке.
