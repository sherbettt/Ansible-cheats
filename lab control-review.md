### Условие:
```
Instructions

On workstation, change to the /home/student/control-review project directory.
The project directory contains a partially completed playbook, playbook.yml. Using a texteditor, add a task that uses the fail module under the #Fail Fast Message comment.Be sure to provide an appropriate name for the task. This task should only be executed whenthe remote system does not meet the minimum requirements.The minimum requirements for the remote host are listed below:• Has at least the amount of RAM specified by the min_ram_mb variable. The min_ram_mbvariable is defined in the vars.yml file and has a value of 256.• Is running Red Hat Enterprise Linux.
Add a single task to the playbook under the #Install all Packages comment toinstall the latest version of any missing packages. Required packages are specified by thepackages variable, which is defined in the vars.yml file.The task name should be Ensure required packages are present.
Add a single task to the playbook under the #Enable and start services commentto start all services. All services specified by the services variable, which is defined in thevars.yml file, should be started and enabled. Be sure to provide an appropriate name forthe task.
Add a task block to the playbook under the #Block of config tasks comment. Thisblock contains two tasks:• A task to ensure the directory specified by the ssl_cert_dir variable exists on theremote host. This directory stores the web server's certificates.174 RH294-RHEL8.4-en-1-20210818Chapter 4 | Implementing Task Control• A task to copy all files specified by the web_config_files variable to the remotehost. Examine the structure of the web_config_files variable in the vars.yml file.Configure the task to copy each file to the correct destination on the remote host.This task should trigger the restart web service handler if any of these files arechanged on the remote server.Additionally, a debug task is executed if either of the two tasks above fail. In this case, thetask prints the message: One or more of the configuration changes failed,but the web service is still active.Be sure to provide an appropriate name for all tasks.
The playbook configures the remote host to listen for standard HTTPS requests. Add asingle task to the playbook under the #Configure the firewall comment to configurefirewalld.This task should ensure that the remote host allows standard HTTP and HTTPS connections.These configuration changes should be effective immediately and persist after a systemreboot. Be sure to provide an appropriate name for the task.
Define the restart web service handler.When triggered, this task should restart the web service defined by the web_servicevariable, defined in the vars.yml file.
From the project directory, ~/control-review, run the playbook.yml playbook. Theplaybook should execute without errors, and trigger the execution of the handler task.
Verify that the web server now responds to HTTPS requests, using the self-signed customcertificate to encrypt the connection. The web server response should match the stringConfigured for both HTTP and HTTPS.
```

### Решение задачи:

#### Шаг 1: Добавляем задачу Fail Fast Message
Необходимо добавить задачу, проверяющую выполнение минимальных требований удалённой системы. Если хотя бы одно требование не выполнено — задача должна завершаться ошибкой с использованием модуля `fail`.

```yaml
---
- hosts: all
  become: true
  vars_files:
    - vars.yml

  tasks:
    - name: Check Minimum Requirements
      fail:
        msg: "System does not meet minimum requirements."
      when:
        - ansible_memory_mb.real.total|int < min_ram_mb
        - ansible_distribution != 'Red Hat'
```

**Обоснование:**  
Здесь мы используем модуль `fail`, который выводит сообщение и прерывает выполнение плейбука, если условия проверки не выполнены. Мы проверяем два требования: количество оперативной памяти и операционную систему.

#### Шаг 2: Установка пакетов
Далее добавляем задачу, устанавливающую необходимые пакеты.

```yaml
    - name: Ensure required packages are present
      package:
        name: "{{ item }}"
        state: latest
      loop: "{{ packages }}"
```

**Обоснование:**  
Модуль `package` используется для установки пакета. Здесь мы выполняем цикл (`loop`) по списку переменной `packages`, обеспечивая установку всех необходимых пакетов последней доступной версии.

#### Шаг 3: Запуск и включение сервисов
Теперь обеспечиваем запуск и включение указанных сервисов.

```yaml
    - name: Start and enable required services
      service:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop: "{{ services }}"
```

**Обоснование:**  
Используя модуль `service`, задаём состояние каждого сервиса из списка переменных `services`. Условием является, что сервис должен быть запущен и включён автоматически при загрузке системы.

#### Шаг 4: Блок конфигурационных задач
Создаем блок задач для настройки директории сертификатов и копирования файлов конфигурации веб-сервера.

```yaml
    - block:
        - name: Create SSL Certificate Directory
          file:
            path: "{{ ssl_cert_dir }}"
            state: directory
        
        - name: Copy Web Configuration Files
          copy:
            src: "{{ item.src }}"
            dest: "{{ item.dest }}"
          notify: restart web service
          loop: "{{ web_config_files }}"
      
      rescue:
        - name: Debug on Failure
          debug:
            msg: "One or more of the configuration changes failed, but the web service is still active."
```

**Обоснование:**  
Мы создаем блок задач (`block`), состоящий из двух основных шагов: создание каталога сертификатов и копирование файлов конфигурации веб-сервера. Для обоих действий предусмотрены обработчики ошибок. В случае возникновения проблем выводится отладочное сообщение.

#### Шаг 5: Настройка фаерволла
Настраиваем правила фаерволла для стандартных портов HTTP и HTTPS.

```yaml
    - name: Configure Firewall Rules
      firewalld:
        port: ["{{ http_port }}/tcp","{{ https_port }}/tcp"]
        permanent: yes
        immediate: yes
        state: enabled
```

**Обоснование:**  
Задача добавляет правило в фаерволл, разрешая доступ по стандартным портам HTTP и HTTPS. Эти изменения сохраняются даже после перезагрузки сервера.

#### Шаг 6: Определение хэндлера для перезапуска веб-сервиса
Описываем хэндлер для перезапуска службы веб-сервера.

```yaml
handlers:
  - name: restart web service
    service:
      name: "{{ web_service }}"
      state: restarted
```




### Финальная версия playbook.yml

```yaml
---
- name: Playbook Control Lab
  hosts: webservers
  vars_files: vars.yml
  tasks:
    #Fail Fast Message
    - name: Проверка соответствия минимальным требованиям
      fail:
        msg: "Минимальные требования не соблюдаются! Оперативная память ниже {{ min_ram_mb }} Мб или неподдерживаемая ОС."
      when:
        - ansible_memory_mb.real.total|int < min_ram_mb
        - ansible_distribution != 'Red Hat'

    #Install all Packages
    - name: Установить требуемые пакеты
      package:
        name: "{{ item }}"
        state: latest
      loop: "{{ packages }}"

    #Enable and start services
    - name: Включить и запустить нужные сервисы
      service:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop: "{{ services }}"

    #Block of config tasks
    - block:
        - name: Создать директорию сертификатов
          file:
            path: "{{ ssl_cert_dir }}"
            state: directory

        - name: Копировать файлы конфигурации веб-сервера
          copy:
            src: "{{ item.src }}"
            dest: "{{ item.dest }}"
          notify: restart web service
          loop: "{{ web_config_files }}"

      rescue:
        - name: Сообщение об ошибке конфигурации
          debug:
            msg: "Ошибка при настройке одного или нескольких элементов конфигурации, но веб-сервер всё ещё работает."

    #Configure the firewall
    - name: Настроить фаерволл
      firewalld:
        port: ["http/tcp", "https/tcp"]  # Используем стандартные имена служб
        permanent: yes
        immediate: yes
        state: enabled

  #Add handlers
  handlers:
    - name: Перезапустить веб-сервис
      service:
        name: "{{ web_service }}"
        state: restarted
```

---

### Объяснение изменений:

1. **Проверка требований**:  
   Название задачи изменено на «Проверка соответствия минимальным требованиям». Используется условие на основании значений из файла `vars.yml`: проверка объёма оперативной памяти и операционной системы.

2. **Установить пакеты**:  
   Модифицирована задача установки пакетов, которая циклически проходит по каждому пакету из списка переменной `packages`.

3. **Сервисы**:  
   Аналогично предыдущему пункту, создаётся список сервисов для запуска и включения из переменной `services`.

4. **Конфигурационные задачи**:  
   Во-первых, создана директория для хранения сертификатов SSL. Затем выполняется копирование файлов конфигурации из локального хранилища в целевые пути на удалённом хосте.

5. **Обработка ошибок**:  
   В блоке предусмотрено резервное задание ("rescue"), которое выводит диагностическое сообщение, если одна из предыдущих задач завершилась неудачно.

6. **Настройки фаерволла**:  
   Применяются стандартные значения имен портов для HTTP и HTTPS, позволяющие избежать путаницы с числами портов. Правила применяются немедленно и сохранятся после перезагрузки.

7. **Хэндлер**:  
   Реализован механизм перезапуска веб-сервера при любом изменении файлов конфигурации, перечисленных в списке `web_config_files`.

---

```bash
ansible-playbook playbook.yml
```

Убедитесь, что удалённый узел удовлетворяет минимальным требованиям и имеет доступ к репозиториям пакетов перед выполнением плейбука.




