# ansible_ssh_common_args - описание и возможные значения

Переменная `ansible_ssh_common_args` позволяет задать дополнительные аргументы командной строки, которые будут передаваться всем SSH-вызовам Ansible (включая sftp, scp и rsync).

## Возможные значения

Переменная может принимать любые допустимые аргументы SSH-клиента. Некоторые распространённые примеры:

1. **Прокси-команды**:
   ```yaml
   ansible_ssh_common_args: '-o ProxyCommand="ssh -W %h:%p -q user@jump-host"'
   ```

2. **Настройки таймаута**:
   ```yaml
   ansible_ssh_common_args: '-o ConnectTimeout=30 -o ConnectionAttempts=5'
   ```

3. **Альтернативные порты**:
   ```yaml
   ansible_ssh_common_args: '-o Port=2222'
   ```

4. **Специфичные ключи**:
   ```yaml
   ansible_ssh_common_args: '-o IdentityFile=/path/to/private/key'
   ```

5. **Отключение проверки хоста**:
   ```yaml
   ansible_ssh_common_args: '-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'
   ```

6. **Несколько параметров**:
   ```yaml
   ansible_ssh_common_args: '-o ControlMaster=auto -o ControlPersist=60s'
   ```

## Где посмотреть все возможные параметры

Полный список доступных SSH-параметров можно посмотреть:

1. В man-странице SSH:
   ```bash
   man ssh_config
   ```

2. В документации OpenSSH:
   - https://www.openssh.com/manual.html
   - Раздел "CONFIGURATION FILES" → "ssh_config"

3. В документации Ansible:
   - https://docs.ansible.com/ansible/latest/inventory_guide/connection_details.html
   - https://docs.ansible.com/ansible/latest/reference_appendices/config.html

## Где задаётся переменная

Переменную можно определить:
- В инвентарном файле (для хоста или группы):
  ```ini
  [web_servers]
  web1 ansible_host=192.168.1.1 ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p jump-host"'
  ```

- В групповых переменных:
  ```yaml
  # group_vars/all.yml
  ansible_ssh_common_args: '-o ConnectTimeout=20'
  ```

- В playbook:
  ```yaml
  vars:
    ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
  ```

Эта переменная особенно полезна при работе через bastion-хосты или в сложных сетевых конфигурациях.

