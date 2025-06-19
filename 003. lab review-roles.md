
Давайте пошагово пройдем по решению данной задачи:

### Шаг 1. Проверка инвентаря и создание рабочей директории

Выполните следующие команды последовательно:
```bash
# Перейдите в рабочую директорию проекта
cd review-roles

# Проверьте конфигурацию инвентаря командой `ansible-inventory`
ansible-inventory -i inventory --list
```

### Шаг 2. Преобразование плейбука в роль

Создаем структуру роли:
```bash
mkdir -p roles/ansible-vsftpd/{tasks,handlers,templates,files,vars,defaults,meta}
```
Затем перемещаем содержимое существующего плейбука (`ansible-vsftpd.yml`) в файл `main.yml` внутри папки `tasks`.

Например, исходный контент плейбука выглядит примерно так:
```yaml
---
- name: Install VSFTPD Service
  hosts: all
  become: true
  tasks:
    # Сюда переносятся ваши задания...
```
Вы можете удалить оригинальный плейбук `ansible-vsftpd.yml`, так как теперь мы будем использовать новую роль.

### Шаг 3. Обновление метаданных роли

Откройте файл `meta/main.yml` и заполните его следующим образом:
```yaml
galaxy_info:
  author: Red Hat Training
  description: Example role for RH294
  company: Red Hat
  license: BSD
dependencies: []
```

### Шаг 4. Редактирование файла документации (README.md)

Изменяем содержание файла `README.md`:
```markdown
ansible-vsftpd
==============

Example ansible-vsftpd role from Red Hat's "Linux Automation" (RH294) course.

Role Variables
--------------
* `defaults/main.yml`: Contains variables used to configure the `vsftpd.conf` template.
* `vars/main.yml`: Contains the name of the vsftpd service, the name of the RPM package, and the location of the service's configuration file.

Dependencies
-------------
None.

Example Playbook
----------------
```yaml
- hosts: servers
  roles:
  - ansible-vsftpd
```

License
-------
BSD

Author Information
------------------
Red Hat (training@redhat.com)
```

### Шаг 5. Удаление неиспользуемых каталогов

Удалите ненужные пустые директории:
```bash
rm -rf roles/ansible-vsftpd/files
rm -rf roles/ansible-vsftpd/templates
```
Эти директории удаляются потому, что в данном задании файлы шаблонов и файлов ролей отсутствуют.

### Шаг 6. Создание нового плейбука

Создайте новый плейбук `vsftpd-configure.yml` со следующим содержимым:
```yaml
---
- name: Install and configure vsftpd
  hosts: ftpservers
  vars:
    vsftpd_anon_root: "/mnt/share/"
    vsftpd_local_root: "/mnt/share/"
  roles:
    - ansible-vsftpd
  tasks:
    - name: Partition /dev/vdb1
      parted:
        device: /dev/vdb
        number: 1
        label: gpt
        part_start: 1MiB
        part_end: 100%
        state: present
    - name: Format /dev/vdb1 as XFS
      filesystem:
        dev: /dev/vdb1
        fstype: xfs
        force: yes
    - name: Ensure /mnt/share exists
      file:
        path: '{{ vsftpd_anon_root }}'
        state: directory
    - name: Mount /dev/vdb1 on /mnt/share
      mount:
        path: '{{ vsftpd_anon_root }}'
        src: /dev/vdb1
        fstype: xfs
        state: mounted
        dump: '1'
        passno: '2'
        notify: restart vsftpd
    - name: Set proper permissions on /mnt/share
      file:
        path: '{{ vsftpd_anon_root }}'
        owner: root
        group: root
        mode: '0755'
        setype: "{{ vsftpd_setype }}"
        state: directory
    - name: Place README file into FTP share
      copy:
        dest: '{{ vsftpd_anon_root }}/README'
        content: "Welcome to the FTP server at {{ ansible_fqdn }}\n"
        setype: '{{ vsftpd_setype }}'
```

### Шаг 7. Изменение основного плейбука

Обновляем основной плейбук `site.yml`, заменяя ссылку на старый плейбук ссылкой на вновь созданный:
```yaml
---
- import_playbook: vsftpd-configure.yml
```

### Шаг 8. Запуск проверки работоспособности плейбуков

Запустите проверку выполнения вашего нового плейбука:
```bash
ansible-playbook -i inventory site.yml
```
Убедитесь, что выполнение прошло успешно и сервис настроился правильно.

Таким образом, задача выполнена согласно инструкции.

-----------------------

Инструкции

1. Перейдите в рабочий каталог `review-roles`. Настройте проект Ansible для использования статического файла инвентаря `inventory`. Проверьте конфигурацию инвентаря командой `ansible-inventory`.

2. Преобразуйте плейбук `ansible-vsftpd.yml` в роль `ansible-vsftpd`.


3. Обновите содержимое файла `roles/ansible-vsftpd/meta/main.yml`, заполнив следующие значения переменных:
| Переменная     | Значение           |
|-----------------|--------------------|
| author          | Red Hat Training   |
| description     | пример роли для RH294 |
| company         | Red Hat            |
| license         | BSD                |

4. Измените содержимое файла `roles/ansible-vsftpd/README.md`, добавив соответствующую информацию о роли. После изменения файл должен содержать следующее:
```
ansible-vsftpd
==============
Примерная роль ansible-vsftpd из курса "Автоматизация Linux" (RH294), предоставляемого компанией Red Hat.
Переменные роли
---------------
Файл `defaults/main.yml` содержит переменные, используемые для настройки шаблона `vsftpd.conf`. Файл `vars/main.yml` включает имя службы vsftpd, название пакета RPM и расположение конфигурационного файла службы.
Зависимости
-----------
Отсутствуют.
Пример плейбука
----------------
```yaml
- hosts: servers
  roles:
  - ansible-vsftpd
```
Лицензия
-------
BSD
Информация об авторе
------------------
Red Hat (training@redhat.com)
```

5. Удалите неиспользуемые директории из новой роли.

6. Создайте новый плейбук `vsftpd-configure.yml`, содержащий следующий код:
```yaml
---
- name: Установка и настройка vsftpd
  hosts: ftpservers
  vars:
    vsftpd_anon_root: "/mnt/share/"
    vsftpd_local_root: "/mnt/share/"
  roles:
    - ansible-vsftpd
  tasks:
    - name: Раздел /dev/vdb1 размечен
      parted:
        device: /dev/vdb
        number: 1
        label: gpt
        part_start: 1MiB
        part_end: 100%
        state: present
    - name: ФС XFS существует на разделе /dev/vdb1
      filesystem:
        dev: /dev/vdb1
        fstype: xfs
        force: yes
    - name: Точка монтирования anon_root создана
      file:
        path: '{{ vsftpd_anon_root }}'
        state: directory
    - name: /dev/vdb1 смонтирован на anon_root
      mount:
        path: '{{ vsftpd_anon_root }}'
        src: /dev/vdb1
        fstype: xfs
        state: mounted
        dump: '1'
        passno: '2'
      notify: перезагрузка vsftpd
    - name: Проверяем права доступа к смонтированной ФС
      file:
        path: '{{ vsftpd_anon_root }}'
        owner: root
        group: root
        mode: '0755'
        setype: "{{ vsftpd_setype }}"
        state: directory
    - name: Копируем README в корневую директорию анонимного FTP
      copy:
        dest: '{{ vsftpd_anon_root }}/README'
        content: "Добро пожаловать на FTP-сервер {{ ansible_fqdn }}\n"
        setype: '{{ vsftpd_setype }}'
```

7. Измените плейбук `site.yml`, чтобы он использовал вновь созданный плейбук `vsftpd-configure.yml` вместо старого плейбука `ansible-vsftpd.yml`.

8. Убедитесь, что плейбук `site.yml` работает правильно, запустив его командой `ansible-playbook`.
