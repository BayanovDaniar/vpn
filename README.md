# Ansible VPN Router Configuration

Автоматическая настройка VPN (sing-box) и OpenWrt роутера через Ansible.

## Структура проекта

```
ansible/
├── ansible.cfg                 # Конфигурация Ansible
├── envs/
│   └── main/
│       ├── hosts.ini           # Инвентарь хостов
│       └── group_vars/
│           ├── all/
│           │   ├── main.yml          # Общие переменные
│           │   ├── secrets.yml       # Секреты (VPN серверы, пароли)
│           │   └── secrets.yml.example # Пример секретов
│           ├── local.yml            # Переменные для локального Mac
│           └── router.yml           # Переменные для роутера
├── playbooks/
│   ├── local.yml              # Настройка локального sing-box на Mac
│   ├── router.yml             # Настройка OpenWrt роутера
│   └── vpn.yml                # Обновление VPN серверов
└── roles/
    ├── vpn/                   # VPN роль (локальная + серверная)
    └── router/                # Роль роутера (OpenWrt)
```

## Подготовка

1. **Установите Ansible и зависимости:**
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   pip install ansible
   ```

2. **Скопируйте и настройте секреты:**
   ```bash
   cp envs/main/group_vars/all/secrets.yml.example envs/main/group_vars/all/secrets.yml
   nano envs/main/group_vars/all/secrets.yml
   ```

3. **Зашифруйте секреты (рекомендуется):**
   ```bash
   ansible-vault encrypt envs/main/group_vars/all/secrets.yml
   ```

## Конфигурация

### VPN серверы (`secrets.yml`)

```yaml
vault_vpn_servers:
  - name: amsterdam
    server: "45.32.185.78"
    server_ipv6: "2a05:f480:1400:0d7b:5400:05ff:fed7:7669"
    public_key: "tzYWe1QAWNClwPrAhrm6NJqJaQBP-DG8u0ibn9W78H0"
    short_id: "16e8e8db"
    clients:
      router: "81d6d74b-5fea-41f3-b7e5-db2cce711576"
      ione-MacBook: "95639b84-a41b-453d-bf45-3efc10f037c2"
```

**Поля:**
- `name`: Уникальное имя сервера
- `server`: IPv4 адрес
- `server_ipv6`: IPv6 адрес (опционально)
- `public_key`: TLS Reality public key
- `short_id`: Reality short ID
- `clients`: UUID клиентов для этого сервера

### Клиенты

В `group_vars/local.yml` или `group_vars/router.yml`:
```yaml
vpn_client_name: router  # или ione-MacBook
```

### Дополнительные настройки

**Роутер (`group_vars/router.yml`):**
```yaml
# WiFi SSID
router_wifi_ssid_2g: "OpenWRT_333_2G"
router_wifi_ssid_5g: "OpenWRT_333_5G"
```

**Общие переменные (`group_vars/all/main.yml`):**
```yaml
# Исключить сервисы из VPN
vpn_services_exclude:
  - kinovod
  - telegram

# Включить IPv6
vpn_ipv6_enabled: true
```

## Запуск плейбуков

### Локальная настройка (Mac)

```bash
# Полная настройка sing-box
ansible-playbook -i envs/main/hosts.ini playbooks/local.yml

# Только домены и PAC файл
ansible-playbook -i envs/main/hosts.ini playbooks/local.yml --tags local

# Check режим (без изменений)
ansible-playbook -i envs/main/hosts.ini playbooks/local.yml --check
```

### Настройка роутера (OpenWrt)

```bash
# Полная настройка роутера
ansible-playbook -i envs/main/hosts.ini playbooks/router.yml

# Только VPN
ansible-playbook -i envs/main/hosts.ini playbooks/router.yml --tags singbox

# Только WiFi
ansible-playbook -i envs/main/hosts.ini playbooks/router.yml --tags wifi

# Только домены
ansible-playbook -i envs/main/hosts.ini playbooks/router.yml --tags domains
```

### Обновление VPN серверов

```bash
# Обновить конфигурацию на всех хостах
ansible-playbook -i envs/main/hosts.ini playbooks/vpn.yml
```

## Теги

Для выборочного запуска используйте теги:

- `local`: Локальные задачи (PAC, конфигурация sing-box)
- `server`: Серверные задачи (VPS с x-ui)
- `packages`: Установка пакетов
- `network`: Сетевая конфигурация
- `wifi`: Настройка WiFi
- `domains`: Управление доменами
- `singbox`: Конфигурация sing-box
- `cleanup`: Очистка старых файлов

## Роли

### VPN роль

**Поддерживает:**
- Локальную настройку sing-box на macOS
- Серверную настройку на VPS (x-ui)
- Генерацию PAC файлов с внешними источниками
- Динамическую генерацию списка VPN серверов

**Основные переменные:**
- `vpn_ipv6_enabled`: Включить IPv6 (default: true)
- `vpn_domains`: Включить домены для сервисов
- `vpn_extra_sources`: Внешние источники доменов
- `vpn_local_config_dir`: Директория конфигурации (default: ~/.config/sing-box)

### Router роль

**Поддерживает:**
- Настройку OpenWrt роутера
- Управление WiFi сетями
- Интеграцию с sing-box
- Обновление доменов через getdomains.sh

**Основные переменные:**
- `router_vpn_enabled`: Включить VPN (default: true)
- `router_wifi_ssid_2g/5g`: Имена WiFi сетей
- `router_network_*`: Настройки сети
- `router_sysctl_*`: Настройки производительности

## Примеры использования

### Добавление нового VPN сервера

1. Добавьте в `secrets.yml`:
   ```yaml
   - name: frankfurt
     server: "1.2.3.4"
     public_key: "new-public-key"
     short_id: "new-short-id"
     clients:
       router: "new-uuid-for-router"
       ione-MacBook: "new-uuid-for-mac"
   ```

2. Запустите обновление:
   ```bash
   ansible-playbook -i envs/main/hosts.ini playbooks/local.yml
   ansible-playbook -i envs/main/hosts.ini playbooks/router.yml
   ```

### Добавление нового клиента

1. Для каждого сервера добавьте UUID:
   ```yaml
   clients:
     router: "..."
     ione-MacBook: "..."
     new-client: "generated-uuid"
   ```

2. Создайте файл `group_vars/new-client.yml`:
   ```yaml
   vpn_client_name: new-client
   ```

3. Запустите с инвентарём для нового клиента.

### Отключение IPv6

В `group_vars/all/main.yml`:
```yaml
vpn_ipv6_enabled: false
```

Или для конкретного хоста в его `group_vars` файле.

## Полезные команды

```bash
# Показать сгенерированные VPN серверы
ansible-playbook -i envs/main/hosts.ini playbooks/local.yml --tags local -v

# Проверить конфигурацию без изменений
ansible-playbook -i envs/main/hosts.ini playbooks/router.yml --check --diff

# Только обновить домены на роутере
ansible-playbook -i envs/main/hosts.ini playbooks/router.yml --tags domains

# Перезапустить sing-box
ansible-playbook -i envs/main/hosts.ini playbooks/router.yml --tags singbox
```

## Требования

- Python 3.8+
- Ansible 2.10+
- OpenWrt с python3 (для роутера)
- Доступ по SSH к целевым хостам

## Безопасность

- Все секреты хранятся в `secrets.yml` и должны быть зашифрованы через `ansible-vault`
- Файлы секретов добавлены в `.gitignore`
- Используйте SSH ключи для аутентификации вместо паролей