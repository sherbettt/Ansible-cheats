 [Lookups](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_lookups.html)

Плагин `lookup` в Ansible позволяет получать данные из внешних источников (файлы, базы данных, команды системы и др.). 
Он работает в Jinja2-шаблонах и может использоваться в модулях, директивах и переменных.

---

## **1. Основные типы `lookup` и примеры**
### **1.1. Чтение файла (`file`)**
Получает содержимое файла.

**Пример 1: Передача содержимого файла в переменную**
```yaml
vars:
  file_content: "{{ lookup('file', '/etc/motd') }}"

tasks:
  - debug:
      var: file_content
```

**Пример 2: Использование в модуле `copy`**
```yaml
tasks:
  - copy:
      content: "{{ lookup('file', '/tmp/source.txt') }}"
      dest: /tmp/destination.txt
```

---

### **1.2. Запуск команды (`pipe`)**
Выполняет команду в shell и возвращает её вывод.

**Пример: Получение версии ядра Linux**
```yaml
vars:
  kernel_version: "{{ lookup('pipe', 'uname -r') }}"

tasks:
  - debug:
      var: kernel_version
```

---

### **1.3. Чтение переменных окружения (`env`)**
Получает значение переменной окружения.

**Пример: Получение `$HOME`**
```yaml
vars:
  user_home: "{{ lookup('env', 'HOME') }}"

tasks:
  - debug:
      var: user_home
```

---

### **1.4. Генерация пароля (`password`)**
Создаёт случайный пароль и сохраняет его в файл (если файл существует, возвращает сохранённый пароль).

**Пример: Генерация пароля для БД**
```yaml
tasks:
  - debug:
      var: lookup('password', '/tmp/mysql_pass.txt length=12 chars=ascii_letters,digits')
```

---

### **1.5. Получение данных из CSV-файла (`csvfile`)**
Читает данные из CSV-файла.

**Пример: Получение IP-адреса сервера из `hosts.csv`**
```csv
# hosts.csv
name,ip,port
web1,192.168.1.10,80
db1,192.168.1.20,5432
```
```yaml
vars:
  db_ip: "{{ lookup('csvfile', 'db1 file=hosts.csv delimiter=, col=1') }}"

tasks:
  - debug:
      var: db_ip  # Выведет 192.168.1.20
```

---

### **1.6. Получение данных из Redis (`redis_kv`)**
Читает значение из Redis по ключу.

**Пример: Получение значения по ключу `app:config`**
```yaml
vars:
  redis_value: "{{ lookup('redis_kv', 'redis://localhost:6379,app:config') }}"

tasks:
  - debug:
      var: redis_value
```

---

### **1.7. Получение данных из Consul (`consul_kv`)**
Читает значение из Consul KV-хранилища.

**Пример: Получение конфигурации из Consul**
```yaml
vars:
  consul_data: "{{ lookup('consul_kv', 'config/app/settings', host='consul.example.com') }}"

tasks:
  - debug:
      var: consul_data
```

---

## **2. Использование `lookup` в разных директивах**
### **2.1. В `vars` (переменные)**
```yaml
vars:
  secret_key: "{{ lookup('password', '/tmp/secret.txt') }}"
```

### **2.2. В `tasks` (задачи)**
```yaml
tasks:
  - copy:
      content: "{{ lookup('file', '/tmp/template.conf') }}"
      dest: /etc/app/config.conf
```

### **2.3. В `templates` (Jinja2-шаблоны)**
```jinja
# config.j2
DB_PASSWORD = {{ lookup('password', '/tmp/db_pass.txt') }}
```

### **2.4. В `when` (условия)**
```yaml
tasks:
  - command: restart_service
    when: lookup('file', '/tmp/flag') == "1"
```

---

## **3. Комбинирование `lookup` с другими модулями**
### **3.1. С `lineinfile` (добавление строки в файл)**
```yaml
tasks:
  - lineinfile:
      path: /etc/ssh/sshd_config
      line: "PermitRootLogin {{ lookup('env', 'ALLOW_ROOT_LOGIN') | default('no') }}"
```

### **3.2. С `uri` (HTTP-запрос)**
```yaml
tasks:
  - uri:
      url: "https://api.example.com/data"
      headers:
        Authorization: "Bearer {{ lookup('file', '/tmp/api_token') }}"
```

------------

Штатный пример.

### 1. Секция `vars`:
```yaml
vars:
  motd_value: "{{ lookup('file', '/etc/motd') }}"
```
- Здесь определяется переменная с именем `motd_value`.
- Значение переменной получается с помощью плагина `lookup` типа `file`, который читает содержимое файла `/etc/motd`.
- `/etc/motd` - это стандартный файл в Unix/Linux системах, содержащий "Message Of The Day" (сообщение дня), которое обычно показывается пользователям при входе в систему.
- Функция lookup('file', '/etc/motd') прочитает содержимое файла /etc/motd и вернёт его как строку.

### 2. Секция `tasks`:
```yaml
tasks:
  - debug:
      msg: "motd value is {{ motd_value }}"
```
- Здесь определяется задача (task), которая использует модуль `debug`.
- Модуль `debug` выводит сообщение во время выполнения playbook.
- В данном случае он выведет текст "motd value is " и затем содержимое переменной `motd_value` (то есть содержимое файла `/etc/motd`).

### Итоговое объяснение:
Этот Ansible код:
1. Читает содержимое файла `/etc/motd` с помощью плагина lookup
2. Сохраняет это содержимое в переменную `motd_value`
3. Выводит значение этой переменной с помощью модуля debug

Пример вывода, если в `/etc/motd` содержится "Добро пожаловать на сервер!":
```
motd value is Добро пожаловать на сервер!
```
Этот код может быть частью playbook'а для проверки или отладки содержимого MOTD на управляемых серверах.

-------
-------
Этот блок кода на Ansible демонстрирует два способа использования **lookup-плагинов** (или **query-плагинов**) для итерации по списку элементов. Оба варианта делают по сути одно и то же, но с разным синтаксисом. Давайте разберём его подробно.



## **1. Первая задача: `lookup` с `wantlist=True`**
```yaml
- debug:
    msg: "{{ item }}"
  loop: "{{ lookup('ns.col.lookup_items', wantlist=True) }}"
```
### **Что происходит?**
1. **`lookup('ns.col.lookup_items', wantlist=True)`**  
   - Это вызов lookup-плагина `ns.col.lookup_items` (предположительно, кастомного или из коллекции).
   - Параметр `wantlist=True` гарантирует, что результат всегда будет возвращён **в виде списка**, даже если элемент один.  
     *(Без этого, если lookup вернёт один элемент, он может быть передан как строка, а не список, что сломает `loop`.)*

2. **`loop`**  
   - Ansible перебирает каждый элемент из списка, возвращённого lookup-плагином.
   - Для каждого элемента выводится его значение через модуль `debug`.

### **Пример вывода**
Если `ns.col.lookup_items` вернёт `["item1", "item2", "item3"]`, то вывод будет:
```
ok: [localhost] => (item=item1) => {"msg": "item1"}
ok: [localhost] => (item=item2) => {"msg": "item2"}
ok: [localhost] => (item=item3) => {"msg": "item3"}
```



## **2. Вторая задача: `q` (сокращение для `query`)**
```yaml
- debug:
    msg: "{{ item }}"
  loop: "{{ q('ns.col.lookup_items') }}"
```
### **Что происходит?**
1. **`q('ns.col.lookup_items')`**  
   - `q` — это **сокращение** (alias) для `query`, который появился в Ansible 2.5+.
   - `query` работает почти так же, как `lookup`, но **всегда возвращает список** (аналог `wantlist=True`).  
   - Это более современный и предпочтительный способ, чем `lookup` + `wantlist`.

2. **`loop`**  
   - Аналогично первой задаче, перебирает элементы и выводит их через `debug`.

### **Пример вывода**
Если `ns.col.lookup_items` вернёт `["a", "b", "c"]`, то вывод будет:
```
ok: [localhost] => (item=a) => {"msg": "a"}
ok: [localhost] => (item=b) => {"msg": "b"}
ok: [localhost] => (item=c) => {"msg": "c"}
```



## **Ключевые отличия**
| Метод | Синтаксис | Всегда возвращает список? | Рекомендуется? |
|-------|----------|--------------------------|---------------|
| `lookup` | `lookup('ns.col.lookup_items', wantlist=True)` | Да (благодаря `wantlist`) | Устаревший способ |
| `query` (`q`) | `q('ns.col.lookup_items')` | Да (по умолчанию) | ✅ Современный способ |

### **Вывод**
- Обе задачи делают одно и то же: получают список элементов и выводят их по одному.
- **Первый вариант** использует `lookup` с явным `wantlist=True`, чтобы гарантировать список.
- **Второй вариант** использует `query` (или `q`), который делает то же самое, но короче и современнее.

**Рекомендация:**  
Лучше использовать `q()` или `query()`, так как это более новый и лаконичный синтаксис. `lookup` с `wantlist` — устаревший подход.

