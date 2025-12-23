# Настройка Ansible на Ubuntu

## 1.  **Создали структуру Ansible**
```bash
# Создали каталог для глобальной конфигурации
sudo mkdir -p /etc/ansible

# Создали inventory-файл (список хостов)
sudo mcedit /etc/ansible/hosts

# Создать каталог для глобальных ролей
sudo mkdir -p /etc/ansible/roles

# Установить роль глобально (доступна всем)
sudo ansible-galaxy role install geerlingguy.nginx -p /etc/ansible/roles/

# Или создать свою роль вручную
sudo mkdir -p /etc/ansible/roles/my-role/{defaults,tasks,templates,files,handlers,vars,meta}

# Проверить доступные роли (включая системные)
ansible-galaxy role list
```

## 2.  **Создали конфигурационный файл**
```bash
# Создали основной конфиг-файл
sudo mcedit /etc/ansible/ansible.cfg
```

**Содержимое `/etc/ansible/ansible.cfg`:**
```ini
[defaults]
# Указываем путь к inventory-файлу
inventory = /etc/ansible/hosts

# Отключаем проверку SSH ключа (для тестов)
host_key_checking = False

# Указываем python интерпретатор
interpreter_python = /usr/bin/python3

# для поиска ролей:
roles_path = /etc/ansible/roles:~/.ansible/roles:/usr/share/ansible/roles

# Опционально: если хотите искать роли ещё и в текущем каталоге
# roles_path = ./roles:/etc/ansible/roles:~/.ansible/roles:/usr/share/ansible/roles
```

## 3.  **Настроили inventory-файл**
**Содержимое `/etc/ansible/hosts`:**
```ini
# Локальный хост (тестовый)
localhost ansible_connection=local

# Удалённые хосты с параметрами подключения
178.176.228.93 ansible_user=username_here  # Рабочий хост
178.177.40.201 ansible_user=username_here  # Проблемный хост
```

## 4.  **Проверили настройки**
```bash
# Проверяем, что Ansible видит конфиг
ansible --version
# Должно показать: config file = /etc/ansible/ansible.cfg

# Тестируем подключение ко всем хостам
ansible all -m ping
# localhost → SUCCESS (работает)
# 178.176.228.93 → SUCCESS (работает)
# 178.177.40.201 → UNREACHABLE (нужно настроить доступ)
```

## 5.  **Установили коллекции** (дополнительные модули)
```bash
# Установка отдельных коллекций
ansible-galaxy collection install community.general  # Основные модули
ansible-galaxy collection install ansible.posix      # POSIX-системы
ansible-galaxy collection install --upgrade ansible.posix ansible.utils

# ИЛИ создайте requirements.yml и установите всё
cat > requirements.yml << 'EOF'
collections:
  - name: community.general
  - name: ansible.posix
  - name: community.docker
EOF

ansible-galaxy collection install -r requirements.yml

# Проверяем установленные коллекции
ansible-galaxy collection list
```

## 6.  **Личный каталог Ansible** (`~/.ansible/`)
```bash
# Автоматически создаётся Ansible
# Хранит:
#   - ~/.ansible/collections/    # Установленные коллекции
#   - ~/.ansible/roles/          # Скачанные роли
#   - ~/.ansible/tmp/            # Временные файлы (НЕ ТРОГАТЬ)
```

## 7.  **Создали тестовый playbook**
```bash
# Создаём простой тестовый playbook
cat > test-playbook.yml << 'EOF'
---
- name: Test Playbook
  hosts: localhost
  tasks:
    - name: Show message
      debug:
        msg: "Ansible настроен и работает!"
EOF

# Запускаем
ansible-playbook test-playbook.yml
```

## ✅ **Что получили:**
- ✅ Глобальная конфигурация в `/etc/ansible/`
- ✅ Inventory с хостами в `/etc/ansible/hosts`
- ✅ Коллекции установлены в `~/.ansible/collections/`
- ✅ Локальный хост отвечает на `ansible all -m ping`

## 🔧 **Дальнейшие действия:**
1. **Настроить SSH-ключи** для удалённых хостов
2. **Добавить переменные** в inventory для разных сред
3. **Создать структуру проекта** с отдельными inventory для prod/dev

**Команда для быстрой проверки:** `ansible all -m ping -i /etc/ansible/hosts`
