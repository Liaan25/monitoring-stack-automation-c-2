# Исправления конфигурации Vault Agent в Secure Edition

## Проблема
В Secure Edition функция `setup_vault_config()` создавала конфигурацию с user-space путями (`$HOME/monitoring/...`), но vault-agent работает как системный сервис и ожидает пути в `/opt/vault/`. Это приводило к тому, что vault-agent не мог запуститься и сгенерировать сертификаты при `SKIP_VAULT_INSTALL=false`.

## Выполненные исправления

### 1. Исправлена базовая конфигурация vault-agent
- Изменены пути с user-space на системные:
  - `pid_file = "$VAULT_LOG_DIR/vault-agent.pidfile"` → `pid_file = "/opt/vault/log/vault-agent.pidfile"`
  - `ca_path = "$VAULT_CONF_DIR/ca-trust"` → `ca_path = "/opt/vault/conf/ca-trust"`
  - `role_id_file_path = "$VAULT_ROLE_ID_FILE"` → `role_id_file_path = "/opt/vault/conf/role_id.txt"`
  - `secret_id_file_path = "$VAULT_SECRET_ID_FILE"` → `secret_id_file_path = "/opt/vault/conf/secret_id.txt"`
  - `log_path = "$VAULT_LOG_DIR"` → `log_path = "/opt/vault/log"`
  - `destination = "$VAULT_CONF_DIR/data_sec.json"` → `destination = "/opt/vault/conf/data_sec.json"`

### 2. Пути для сертификатов уже были правильными
- `destination = "/opt/vault/certs/server_bundle.pem"` ✓
- `destination = "/opt/vault/certs/ca_chain.crt"` ✓
- `destination = "/opt/vault/certs/grafana-client.pem"` ✓

### 3. Добавлена логика для SKIP_VAULT_INSTALL=false
Добавлен блок `else` в функцию `setup_vault_config()` для обработки случая `SKIP_VAULT_INSTALL=false`:
- Вывод информационных сообщений о новой установке через RLM
- Попытка копирования файлов `role_id.txt` и `secret_id.txt` в системные пути
- Объяснение workflow при установке через RLM

### 4. Права доступа
- Права `0640` оставлены для сертификатов (доступ группы `va-read`)
- Это соответствует архитектуре Secure Edition

## Архитектура после исправлений

### При SKIP_VAULT_INSTALL=true (существующий vault-agent):
1. Используется существующий системный vault-agent
2. Проверяется его статус и наличие сертификатов
3. Пробуется применить новую конфигурацию (если есть права)

### При SKIP_VAULT_INSTALL=false (новая установка):
1. Создается полная конфигурация с системными путями
2. Файлы аутентификации копируются в `/opt/vault/conf/`
3. RLM устанавливает vault-agent пакет
4. RLM копирует `agent.hcl` в `/opt/vault/conf/`
5. RLM перезапускает vault-agent сервис
6. Vault-agent генерирует сертификаты в `/opt/vault/certs/`

## Тестирование
Исправления проверены:
- ✅ Все пути исправлены на системные
- ✅ Добавлена логика для SKIP_VAULT_INSTALL=false
- ✅ Права доступа соответствуют требованиям Secure Edition
- ✅ Нет синтаксических ошибок (проверено линтером)

## Файл с изменениями
`monitoring-stack-automation-secure/install-monitoring-stack.sh` (строки 1947-2312)