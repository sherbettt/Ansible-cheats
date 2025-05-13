# Работа с Ansible Vault

Ansible Vault - это инструмент для шифрования конфиденциальных данных в Ansible, таких как пароли, ключи API и другие секреты.

## Основные команды ansible-vault

### Создание нового зашифрованного файла
```bash
ansible-vault create secret.yml
```
После выполнения команды вам будет предложено ввести пароль, который будет использоваться для шифрования/дешифрования.

### Шифрование существующего файла
```bash
ansible-vault encrypt secret.yml
```

### Просмотр зашифрованного файла
```bash
ansible-vault view secret.yml
```

### Редактирование зашифрованного файла
```bash
ansible-vault edit secret.yml
```

### Расшифровка файла
```bash
ansible-vault decrypt secret.yml
```

### Изменение пароля Vault
```bash
ansible-vault rekey secret.yml
```

## Примеры использования

### 1. Шифрование переменных

**secret.yml** до шифрования:
```yaml
db_password: Sup3rS3cr3tP@ss
api_key: A1B2-C3D4-E5F6-G7H8
```

После выполнения `ansible-vault encrypt secret.yml`:
```yaml
$ANSIBLE_VAULT;1.1;AES256
33613361326662303438633635313062666561613038386534376234376233386464386530373961
3838653137333530323639663034396661343033366134380a393464646162633539313763353239
...
```

### 2. Использование в playbook

**playbook.yml**:
```yaml
- hosts: all
  vars_files:
    - secret.yml
  tasks:
    - name: Use secret password
      debug:
        msg: "DB password is {{ db_password }}"
```

Запуск playbook с запросом пароля:
```bash
ansible-playbook playbook.yml --ask-vault-pass
```

Или с файлом пароля:
```bash
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass.txt
```

### 3. Шифрование отдельных переменных

**group_vars/all/vault.yml**:
```yaml
vault_db_password: "Sup3rS3cr3tP@ss"
vault_api_key: "A1B2-C3D4-E5F6-G7H8"
```

Шифруем файл:
```bash
ansible-vault encrypt group_vars/all/vault.yml
```

### 4. Использование зашифрованных переменных с незашифрованными

**group_vars/all/vars.yml** (незашифрованный):
```yaml
db_user: app_user
db_name: my_app_db
```

**group_vars/all/vault.yml** (зашифрованный):
```yaml
vault_db_password: "Sup3rS3cr3tP@ss"
```

### 5. Шифрование строки для использования в playbook

```bash
ansible-vault encrypt_string 'S3cr3tP@ss' --name 'vaulted_password'
```

Результат (который можно вставить в playbook или vars файл):
```yaml
vaulted_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          66386439653236336462626566653063336164623961343765663438343832653639323138363862
          3132646465363166663862633535303535323662666137620a316338383939393839666336353339
          32373631613238623461386361396162616537396237396534376431343964353938383761616232
          3639636437633961620a383666613936383834633939376365303765373363323066313365373331
          6564
```

## Советы по работе с Ansible Vault

1. **Хранение пароля**: Лучше хранить пароль в файле с ограниченными правами (600) и использовать `--vault-password-file`
2. **.gitignore**: Добавьте файл с паролем в .gitignore
3. **Идентификация**: Используйте префикс `vault_` для зашифрованных переменных
4. **Множественные пароли**: Можно использовать разные пароли для разных файлов
5. **Автоматизация**: Для CI/CD можно передавать пароль через переменную окружения:
   ```bash
   ansible-playbook playbook.yml --vault-password-file <(echo $VAULT_PASS)
   ```

6. **Проверка**: Проверить, что файл зашифрован можно командой:
   ```bash
   ansible-vault is_encrypted secret.yml && echo "Encrypted"
   ```
