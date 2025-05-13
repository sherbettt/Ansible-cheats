### Решение Лабораторной Работы по Ansible

Структура директории `/home/student/data-review` должна выглядеть следующим образом:

```
data-review/
├── ansible.cfg
├── hosts
└── files
    ├── httpd.conf
    ├── .htaccess
    └── htpasswd
```

### Подробное Руководство по Выполнению Задания

#### 1. Основные Переменные и Значения

Переменные, используемые в задаче:

| Переменная          | Значение                            |
|--------------------|-------------------------------------|
| firewall_pkg       | firewalld                           |
| firewall_svc       | firewalld                           |
| web_pkg            | httpd                               |
| web_svc            | httpd                               |
| ssl_pkg            | mod_ssl                             |
| httpdconf_src      | files/httpd.conf                    |
| httpdconf_dest     | /etc/httpd/conf/httpd.conf         |
| htaccess_src       | files/.htaccess                     |
| secrets_dir        | /etc/httpd/secrets                  |
| secrets_src        | files/htpasswd                      |
| secrets_dest       | "{{ secrets_dir }}/htpasswd"         |
| web_root           | /var/www/html                       |

---

## Плейбук Ansible

### Часть 1: Первичный плей — настройка сервера

```yaml
# Файл: playbook.yml

---
- name: Настройка веб-сервера с базовой аутентификацией
  hosts: webserver
  become: yes
  vars_files:
    - secret_vars.yml

  tasks:

  # Задача 1: Установка необходимых пакетов
  - name: Убедиться, что установлены необходимые пакеты
    yum:
      name: "{{ item }}"
      state: latest
    loop:
      - "{{ firewall_pkg }}"
      - "{{ web_pkg }}"
      - "{{ ssl_pkg }}"

  # Задача 2: Копирование конфигурационного файла Apache
  - name: Скопировать файл конфигурации Apache
    copy:
      src: "{{ httpdconf_src }}"
      dest: "{{ httpdconf_dest }}"
      owner: root
      group: root
      mode: '0644'

  # Задача 3: Создание директории для хранения секретов
  - name: Создать директорию для хранения секретов
    file:
      path: "{{ secrets_dir }}"
      state: directory
      owner: apache
      group: apache
      mode: '0500'

  # Задача 4: Копирование файла с пользователями и паролями
  - name: Скопировать файл с пользователями и паролями
    copy:
      src: "{{ secrets_src }}"
      dest: "{{ secrets_dest }}"
      owner: apache
      group: apache
      mode: '0400'

  # Задача 5: Копирование файла контроля доступа .htaccess
  - name: Скопировать файл контроля доступа .htaccess
    copy:
      src: "{{ htaccess_src }}"
      dest: "{{ web_root }}/.htaccess"
      owner: apache
      group: apache
      mode: '0400'

  # Задача 6: Создание файла индекса веб-сайта
  - name: Создать индексный файл веб-сайта
    copy:
      content: >-
        HOSTNAME (IPADDRESS) has been customized by Ansible.
      dest: "{{ web_root }}/index.html"
      owner: apache
      group: apache
      mode: '0644'
    vars:
      HOSTNAME: "{{ ansible_facts['ansible_hostname'] }}"
      IPADDRESS: "{{ ansible_facts['ansible_default_ipv4']['address'] }}"

  # Задача 7: Включить и запустить сервис Firewalld
  - name: Включить и запустить сервис Firewalld
    service:
      name: "{{ firewall_svc }}"
      enabled: true
      state: started

  # Задача 8: Открыть порт HTTPS в Firewalld
  - name: Разрешить доступ к HTTPS в Firewalld
    firewalld:
      service: https
      permanent: true
      immediate: true

  # Задача 9: Включить и запустить веб-сервис
  - name: Включить и запустить веб-сервис
    service:
      name: "{{ web_svc }}"
      enabled: true
      state: started
```

---

### Часть 2: Вторичный плей — проверка аутентификации

```yaml
- name: Проверка аутентификации на веб-сайте
  hosts: localhost
  gather_facts: no

  vars:
    web_user: guest

  vars_files:
    - vars/secret.yml

  tasks:

  # Задача 1: Отправить GET-запрос к веб-серверу с аутентификацией
  - name: Получить контент с веб-сервера
    uri:
      url: https://serverb.lab.example.com
      method: GET
      validate_certs: false
      user: "{{ web_user }}"
      password: "{{ web_pass }}"
      return_content: yes
    register: response

  # Задача 2: Вывести полученный контент
  - name: Отобразить контент страницы
    debug:
      var: response.content
```

---

## Шаг 3: Создание файла с секретами

Создайте файл `vars/secret.yml` и добавьте в него следующую строку:

```yaml
web_pass: redhat
```

Теперь зашифруйте файл командой:

```bash
ansible-vault encrypt vars/secret.yml
```
или одной командой `ansible-vault create vars/secret.yml`

Введите пароль `redhat` при запросе.

---

## Шаг 4: Запуск плейбука

Запустите плейбук командой:

```bash
ansible-playbook -i hosts playbook.yml --vault-id @prompt

ansible-playbook playbook.yml --ask-vault-pass --syntax-check
ansible-playbook playbook.yml --ask-vault-pass
```

При запросе введите пароль `redhat`.

---

## Итоги

Вы выполнили задание, создав плейбук Ansible, который:

- Устанавливает необходимые пакеты.
- Конфигурирует веб-сервер Apache с базовой аутентификацией.
- Настраивает правила брандмауэра.
- Проверяет успешность аутентификации на веб-сервере.
