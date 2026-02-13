# Исправления конфигурации Vault Agent

## Проблема
При `SKIP_VAULT_INSTALL=false` (новая установка vault-agent) сертификаты не генерировались, файл `server_bundle.pem` был пустым.

## Причина
1. Конфигурация создавалась с **user-space путями** (`/home/.../monitoring/`)
2. Vault-agent (системный сервис) ожидает **системные пути** (`/opt/vault/...`)
3. Конфигурация не применялась к системному vault-agent
4. Vault-agent работал с дефолтной/пустой конфигурацией

## Выполненные исправления

### 1. Разные конфигурации для `SKIP_VAULT_INSTALL=false` и `true`

#### Для `SKIP_VAULT_INSTALL=false` (новая установка):
```hcl
pid_file = "/opt/vault/log/vault-agent.pidfile"
role_id_file_path = "/opt/vault/conf/role_id.txt"
secret_id_file_path = "/opt/vault/conf/secret_id.txt"
log_path = "/opt/vault/log"
destination = "/opt/vault/conf/data_sec.json"
```

#### Для `SKIP_VAULT_INSTALL=true` (уже установлен):
```hcl
pid_file = "$VAULT_LOG_DIR/vault-agent.pidfile"  # user-space
role_id_file_path = "$VAULT_ROLE_ID_FILE"        # user-space
destination = "$VAULT_CONF_DIR/data_sec.json"    # user-space
```

### 2. Применение конфигурации для `SKIP_VAULT_INSTALL=false`

Добавлена логика:
1. Копирование `role_id.txt` и `secret_id.txt` в `/opt/vault/conf/`
2. Копирование `agent.hcl` в `/opt/vault/conf/agent.hcl`
3. Перезапуск vault-agent
4. Проверка статуса vault-agent после перезапуска

### 3. Использование security wrappers

Все операции записи через `config-writer_launcher.sh` для соответствия требованиям безопасности.

### 4. Пути для сертификатов

Остались системными (правильно):
```hcl
destination = "/opt/vault/certs/server_bundle.pem"
destination = "/opt/vault/certs/ca_chain.crt"
destination = "/opt/vault/certs/grafana-client.pem"
```

## Изменения в файлах

### `monitoring-stack-automation-secure/install-monitoring-stack.sh`:

1. **Строки 1947-2016**: Добавлена условная логика для разных путей в зависимости от `SKIP_VAULT_INSTALL`
2. **Строки 2017-2028**: Разные пути для `data_sec.json`
3. **Строки 2319-2420**: Полная логика применения конфигурации для `SKIP_VAULT_INSTALL=false`
4. **Строка 2187**: Добавлено пояснение для `SKIP_VAULT_INSTALL=true`

### `monitoring-stack-automation-secure/wrappers/config-writer.sh`:

1. **Строки 23-61**: Обновлен whitelist для поддержки user-space путей

## Ожидаемый результат

### При `SKIP_VAULT_INSTALL=false`:
1. Создается конфигурация с системными путями
2. Конфигурация применяется к `/opt/vault/conf/agent.hcl`
3. Vault-agent перезапускается
4. Vault-agent генерирует сертификаты в `/opt/vault/certs/`
5. Скрипт копирует сертификаты в user-space
6. `server_bundle.pem` НЕ пустой

### При `SKIP_VAULT_INSTALL=true`:
1. Создается конфигурация с user-space путями (шаблон для справки)
2. Используются существующие сертификаты из `/opt/vault/certs/`
3. Сертификаты копируются в user-space

## Тестирование
- ✅ Нет синтаксических ошибок (проверено линтером)
- ✅ Конфигурационные пути исправлены
- ✅ Логика применения конфигурации добавлена
- ✅ Соответствие требованиям безопасности (использование wrappers)

## Заключение
Исправления должны решить проблему с пустым `server_bundle.pem` при `SKIP_VAULT_INSTALL=false`. Теперь vault-agent получит правильную конфигурацию с системными путями и сможет генерировать сертификаты.