Директива `delegate_to` в Ansible позволяет выполнять задачу на другом хосте, отличном от того, который определен в текущем плейбуке или инвентаре.

## Основная концепция

По умолчанию Ansible выполняет задачи на хостах, указанных в `hosts:` секции плейбука. `delegate_to` позволяет временно "делегировать" выполнение конкретной задачи другому хосту.

## Синтаксис

```yaml
- name: Пример задачи с делегированием
  ansible.builtin.command: some_command
  delegate_to: other_host
```

## Основные случаи использования

### 1. Локальное выполнение

```yaml
- name: Выполнить команду локально
  ansible.builtin.shell: echo "Эта задача выполняется на control node"
  delegate_to: localhost
```

### 2. Выполнение на control node

```yaml
- name: Скачать файл на control node
  ansible.builtin.get_url:
    url: http://example.com/file.tar.gz
    dest: /tmp/
  delegate_to: localhost
```

### 3. Работа с балансировщиками

```yaml
- name: Убрать сервер из балансировщика
  ansible.builtin.uri:
    url: http://loadbalancer/api/remove/{{ inventory_hostname }}
  delegate_to: loadbalancer.example.com
```

### 4. Работа с мониторингом

```yaml
- name: Отключить мониторинг для сервера
  ansible.builtin.uri:
    url: http://monitoring/api/disable/{{ inventory_hostname }}
  delegate_to: monitoring.example.com
```

## Особенности работы

### Переменные при делегировании

```yaml
- name: Показать разницу в переменных
  ansible.builtin.debug:
    msg: "Inventory hostname: {{ inventory_hostname }}"
  delegate_to: other_host
```

### Делегирование с регистрацией результата

```yaml
- name: Проверить доступность сервиса с другого хоста
  ansible.builtin.uri:
    url: http://{{ inventory_hostname }}:8080/health
  register: health_check
  delegate_to: monitoring_server
```

## Специальные значения

### delegate_to: localhost

```yaml
- name: Создать backup на control node
  ansible.builtin.archive:
    path: /var/log/
    dest: /backups/{{ inventory_hostname }}-logs.tar.gz
  delegate_to: localhost
```

### delegate_facts

```yaml
- name: Собрать факты о другом хосте
  ansible.builtin.setup:
  delegate_to: other_host
  delegate_facts: true
```

## Практические примеры

### Развертывание с балансировщиком

```yaml
- name: Развертывание приложения
  hosts: app_servers
  tasks:
    - name: Убрать сервер из балансировщика
      ansible.builtin.uri:
        url: http://lb/disable/{{ inventory_hostname }}
      delegate_to: loadbalancer
    
    - name: Остановить приложение
      ansible.builtin.service:
        name: myapp
        state: stopped
    
    - name: Развернуть новую версию
      ansible.builtin.copy:
        src: /tmp/app-v2.war
        dest: /opt/tomcat/webapps/
    
    - name: Запустить приложение
      ansible.builtin.service:
        name: myapp
        state: started
    
    - name: Вернуть сервер в балансировщик
      ansible.builtin.uri:
        url: http://lb/enable/{{ inventory_hostname }}
      delegate_to: loadbalancer
```

### Мониторинг деплоя

```yaml
- name: Проверить здоровье после деплоя
  ansible.builtin.uri:
    url: http://{{ inventory_hostname }}/health
    return_content: yes
  register: health
  until: health.status == 200
  retries: 10
  delay: 5
  delegate_to: monitoring_host
```

## Важные особенности

1. **Переменные**: При делегировании доступны переменные исходного хоста
2. **Facts**: По умолчанию факты не делегируются (используйте `delegate_facts: true`)
3. **Соединение**: Используются настройки соединения целевого хоста
4. **Ограничения**: Делегирование работает только для одного хоста за раз

## Ограничения

- Нельзя делегировать на несколько хостов одновременно
- Некоторые модули могут работать некорректно при делегировании
- Требует правильной настройки подключения к целевому хосту

