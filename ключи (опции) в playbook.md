Чтобы получить **полный список всех опций и ключей** в Ansible playbook, можно использовать несколько способов:  

---

### **1. Официальная документация Ansible**  
Лучший источник — **официальная документация**, где описаны все ключи и их параметры:  
🔹 [Ansible Playbook Keywords](https://docs.ansible.com/ansible/latest/reference_appendices/playbooks_keywords.html)  
🔹 [Спецификация Playbook (YAML-структура)](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)  

---

### **2. Команда `ansible-doc` для просмотра документации**  
Ansible включает утилиту `ansible-doc`, которая показывает документацию по модулям и ключам:  

#### **Просмотр документации по модулю**  
```bash
ansible-doc MODULE_NAME  # Например: ansible-doc copy
```  

#### **Просмотр всех плагинов и модулей**  
```bash
ansible-doc -l  # Выведет список всех доступных модулей
```  

#### **Просмотр документации по ключевым словам Playbook**  
```bash
ansible-doc -t keyword list  # Покажет все ключевые слова (hosts, vars, tasks и т. д.)
ansible-doc -t keyword NAME  # Например: ansible-doc -t keyword vars
```  

---

### **3. Исходный код Ansible (для глубокого изучения)**  
Если нужны **все возможные опции**, можно посмотреть исходный код Ansible:  
🔹 [GitHub Ansible Core](https://github.com/ansible/ansible)  
🔹 Файлы, определяющие структуру Playbook:  
   - `lib/ansible/playbook/play.py` (основные ключи Play)  
   - `lib/ansible/playbook/task.py` (ключи задач)  
   - `lib/ansible/playbook/role.py` (ключи ролей)  

---

### **4. Генерация схемы JSON (для автоматического анализа)**  
Можно сгенерировать JSON-схему всех возможных ключей:  
```bash
ansible-inspect --type keyword --format json
```  
*(требуется установка `ansible-lint` или `ansible-core` последней версии)*  

---

### **5. Примеры из официальных туториалов**  
🔹 [Ansible Galaxy Examples](https://galaxy.ansible.com/docs/)  
🔹 [Ansible Best Practices](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_best_practices.html)  

---

### **Вывод**  
Если нужен **официальный и полный список** — смотри **документацию** (`ansible-doc -t keyword list`) или исходники.  
Если нужны **примеры использования** — изучай туториалы и GitHub-репозитории с playbook’ами.  

---

В Ansible playbook используются различные ключи (опции) для определения задач, конфигурации и поведения playbook. Вот основные из них:  

### **Основные ключи Playbook**  
1. **`name`** – Описание play или задачи (необязательно, но рекомендуется).  
2. **`hosts`** – Определяет целевую группу хостов или конкретные хосты.  
3. **`gather_facts`** (`true/false`) – Собирать ли факты о хостах (по умолчанию `true`).  
4. **`vars`** – Определение переменных на уровне play.  
5. **`vars_files`** – Подключение файлов с переменными.  
6. **`vars_prompt`** – Запрос переменных у пользователя при запуске playbook.  
7. **`tasks`** – Список задач, выполняемых в play.  
8. **`handlers`** – Определение обработчиков (запускаются по уведомлению).  
9. **`become`** (`true/false`) – Активирует повышение привилегий (sudo).  
10. **`become_user`** – Пользователь, от имени которого выполняются задачи (по умолчанию `root`).  
11. **`become_method`** – Метод повышения прав (`sudo`, `su`, `doas` и др.).  
12. **`connection`** – Тип подключения (`local`, `ssh`, `docker` и др.).  
13. **`remote_user`** – Пользователь для подключения к удалённому хосту.  
14. **`environment`** – Установка переменных окружения.  
15. **`collections`** – Подключение коллекций Ansible для использования в playbook.  
16. **`pre_tasks`** – Задачи, выполняемые до основного блока `tasks`.  
17. **`post_tasks`** – Задачи, выполняемые после основного блока `tasks`.  
18. **`roles`** – Подключение ролей к playbook.  
19. **`tags`** – Теги для выборочного запуска задач.  
20. **`ignore_errors`** (`true/false`) – Игнорировать ошибки в задаче.  
21. **`any_errors_fatal`** (`true/false`) – Остановка playbook при первой ошибке.  
22. **`max_fail_percentage`** – Максимальный процент сбоев перед остановкой.  
23. **`serial`** – Количество хостов, обрабатываемых параллельно (по умолчанию все).  
24. **`strategy`** – Стратегия выполнения (`linear`, `free`, `debug`).  

### **Ключи внутри задач (`tasks`)**  
1. **`name`** – Описание задачи.  
2. **`module`** (например, `command`, `shell`, `copy`, `template`, `yum`, `apt`, `service` и др.).  
3. **`args`** / **`module arguments`** – Аргументы модуля.  
4. **`register`** – Сохранение вывода задачи в переменную.  
5. **`when`** – Условие выполнения задачи.  
6. **`loop`** / **`with_items`** – Цикл для повторения задачи.  
7. **`until`** – Повторять задачу, пока условие не станет истинным.  
8. **`retries`** – Количество попыток выполнения.  
9. **`delay`** – Задержка между попытками (в секундах).  
10. **`notify`** – Уведомление обработчика (handler).  
11. **`changed_when`** – Определение, когда задача считается изменяющей состояние.  
12. **`failed_when`** – Определение, когда задача считается неудачной.  
13. **`ignore_errors`** – Игнорировать ошибки в задаче.  
14. **`delegate_to`** – Делегирование задачи другому хосту.  
15. **`local_action`** – Выполнение задачи локально.  
16. **`tags`** – Теги для выборочного запуска.  

### **Пример Playbook**  
```yaml
---
- name: Example Playbook
  hosts: webservers
  become: true
  vars:
    http_port: 80
    max_clients: 200
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
      notify: restart nginx
    - name: Copy Nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: restart nginx
  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
```  

Это основные ключи, но Ansible поддерживает множество других опций в зависимости от модулей и плагинов.
