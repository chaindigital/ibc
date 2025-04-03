# ibc
# Полное руководство по развертыванию IBC-соединения между Osmosis и Cosmos Hub с настройкой комиссий

Ниже представлен полный пошаговый список команд для настройки IBC-релеера между Osmosis и Cosmos Hub с использованием Hermes релеера, включая настройку получения комиссий.

## Предварительные требования

- Сервер с Linux (Ubuntu/Debian рекомендуется)
- Минимум 4 ГБ RAM, 2 CPU, 100 ГБ SSD
- Установленный Rust и Cargo
- Доступ к RPC-узлам Osmosis и Cosmos Hub (или собственные синхронизированные узлы)
- Кошельки с небольшим количеством OSMO и ATOM для оплаты транзакций

## Шаг 1: Установка Hermes релеера

```bash
# Обновление системы
sudo apt update && sudo apt upgrade -y

# Установка необходимых зависимостей
sudo apt install -y build-essential git curl jq pkg-config libssl-dev

# Установка Rust (если еще не установлен)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

# Клонирование и установка Hermes
git clone https://github.com/informalsystems/hermes.git
cd hermes
git checkout v1.7.4  # Используйте актуальную стабильную версию
cargo install --locked --path crates/relayer-cli

# Проверка установки
hermes version
```

## Шаг 2: Настройка конфигурации Hermes с поддержкой комиссий

```bash
# Создание директории для конфигурации
mkdir -p $HOME/.hermes/keys

# Создание файла конфигурации с поддержкой комиссий
cat > $HOME/.hermes/config.toml << EOF
[global]
log_level = 'info'
clear_packets_interval = 100
tx_confirmation = true

[mode]
[mode.clients]
enabled = true
refresh = true
misbehaviour = true

[mode.connections]
enabled = true

[mode.channels]
enabled = true

[mode.packets]
enabled = true
clear_interval = 100
clear_on_start = true
tx_confirmation = true

[rest]
enabled = true
host = '127.0.0.1'
port = 3000

[telemetry]
enabled = true
host = '127.0.0.1'
port = 3001

[[chains]]
id = 'cosmoshub-4'
type = 'CosmosSdk'
rpc_addr = 'https://rpc.cosmos.network:443'
grpc_addr = 'https://grpc.cosmos.network:443'
websocket_addr = 'wss://rpc.cosmos.network:443/websocket'
rpc_timeout = '10s'
account_prefix = 'cosmos'
key_name = 'cosmos-key'
key_store_type = 'Test'
store_prefix = 'ibc'
default_gas = 100000
max_gas = 3000000
gas_price = { price = 0.025, denom = 'uatom' }
gas_multiplier = 1.3
max_msg_num = 30
max_tx_size = 180000
clock_drift = '5s'
max_block_time = '30s'
trusting_period = '14days'
trust_threshold = { numerator = '1', denominator = '3' }
address_type = { derivation = 'cosmos' }

[[chains.packet_filter]]
policy = 'allow'
list = [
  ['transfer', '*'],
]

[chains.packet_filter.min_fees]
enabled = true
recv_fee = [
  { amount = "100", denom = "uatom" },
]
ack_fee = [
  { amount = "100", denom = "uatom" },
]
timeout_fee = [
  { amount = "100", denom = "uatom" },
]

[[chains]]
id = 'osmosis-1'
type = 'CosmosSdk'
rpc_addr = 'https://rpc.osmosis.zone:443'
grpc_addr = 'https://grpc.osmosis.zone:443'
websocket_addr = 'wss://rpc.osmosis.zone:443/websocket'
rpc_timeout = '10s'
account_prefix = 'osmo'
key_name = 'osmo-key'
key_store_type = 'Test'
store_prefix = 'ibc'
default_gas = 100000
max_gas = 3000000
gas_price = { price = 0.025, denom = 'uosmo' }
gas_multiplier = 1.3
max_msg_num = 30
max_tx_size = 180000
clock_drift = '5s'
max_block_time = '30s'
trusting_period = '14days'
trust_threshold = { numerator = '1', denominator = '3' }
address_type = { derivation = 'cosmos' }

[[chains.packet_filter]]
policy = 'allow'
list = [
  ['transfer', '*'],
]

[chains.packet_filter.min_fees]
enabled = true
recv_fee = [
  { amount = "100", denom = "uosmo" },
]
ack_fee = [
  { amount = "100", denom = "uosmo" },
]
timeout_fee = [
  { amount = "100", denom = "uosmo" },
]
EOF
```

## Шаг 3: Добавление ключей для обеих сетей

```bash
# Создание файлов с мнемоническими фразами (или используйте существующие)
# ВАЖНО: Храните эти файлы в безопасном месте!
echo "your cosmos mnemonic phrase here" > $HOME/cosmos_mnemonic.txt
echo "your osmosis mnemonic phrase here" > $HOME/osmosis_mnemonic.txt
chmod 600 $HOME/cosmos_mnemonic.txt $HOME/osmosis_mnemonic.txt

# Импорт ключа для Cosmos Hub
hermes keys add --chain cosmoshub-4 --mnemonic-file $HOME/cosmos_mnemonic.txt

# Импорт ключа для Osmosis
hermes keys add --chain osmosis-1 --mnemonic-file $HOME/osmosis_mnemonic.txt

# Альтернативно, можно создать новые ключи:
# hermes keys add --chain cosmoshub-4
# hermes keys add --chain osmosis-1

# Проверка добавленных ключей
hermes keys list --chain cosmoshub-4
hermes keys list --chain osmosis-1
```

## Шаг 4: Проверка конфигурации и соединения с сетями

```bash
# Проверка конфигурации
hermes config validate

# Проверка соединения с сетями
hermes health-check
```

## Шаг 5: Создание клиентов, соединения и канала

```bash
# Создание клиентов для обеих сетей
hermes create client --host-chain cosmoshub-4 --reference-chain osmosis-1
hermes create client --host-chain osmosis-1 --reference-chain cosmoshub-4

# Проверка созданных клиентов
hermes query clients --host-chain cosmoshub-4
hermes query clients --host-chain osmosis-1

# Создание соединения между клиентами
hermes create connection --a-chain cosmoshub-4 --b-chain osmosis-1

# Проверка созданного соединения
hermes query connections --chain cosmoshub-4
hermes query connections --chain osmosis-1

# Создание канала для трансфера токенов
hermes create channel --a-chain cosmoshub-4 --b-chain osmosis-1 --a-port transfer --b-port transfer --new-client-connection

# Проверка созданного канала
hermes query channels --chain cosmoshub-4
hermes query channels --chain osmosis-1
```

## Шаг 6: Запуск релеера для постоянного мониторинга и релеинга пакетов

```bash
# Запуск релеера в фоновом режиме с логированием
nohup hermes start > hermes.log 2>&1 &

# Проверка статуса релеера
ps aux | grep hermes

# Просмотр логов
tail -f hermes.log
```

## Шаг 7: Настройка systemd сервиса для автоматического запуска

```bash
# Создание systemd сервиса
sudo tee /etc/systemd/system/hermes.service > /dev/null << EOF
[Unit]
Description=Hermes IBC Relayer
After=network-online.target

[Service]
User=$USER
ExecStart=$(which hermes) start
Restart=always
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

# Активация и запуск сервиса
sudo systemctl daemon-reload
sudo systemctl enable hermes
sudo systemctl start hermes

# Проверка статуса сервиса
sudo systemctl status hermes
```

## Шаг 8: Тестирование IBC-трансфера

```bash
# Пример трансфера ATOM из Cosmos Hub в Osmosis
# Замените адреса на свои реальные адреса
COSMOS_ADDRESS="cosmos1your_address"
OSMOSIS_ADDRESS="osmo1your_address"

# Выполнение трансфера через CLI Cosmos Hub
cosmos-hub-cli tx ibc-transfer transfer transfer channel-X $OSMOSIS_ADDRESS 1000000uatom --from $COSMOS_ADDRESS --chain-id cosmoshub-4 --gas-prices 0.025uatom --gas-adjustment 1.3 --gas auto

# Пример трансфера OSMO из Osmosis в Cosmos Hub
osmosis-cli tx ibc-transfer transfer transfer channel-Y $COSMOS_ADDRESS 1000000uosmo --from $OSMOSIS_ADDRESS --chain-id osmosis-1 --gas-prices 0.025uosmo --gas-adjustment 1.3 --gas auto
```

## Шаг 9: Мониторинг и обслуживание

```bash
# Проверка статуса пакетов
hermes query packet commitments --chain cosmoshub-4 --port transfer --channel channel-X
hermes query packet commitments --chain osmosis-1 --port transfer --channel channel-Y

# Очистка неподтвержденных пакетов (если необходимо)
hermes clear packets --chain cosmoshub-4 --port transfer --channel channel-X
hermes clear packets --chain osmosis-1 --port transfer --channel channel-Y

# Обновление клиентов (если необходимо)
hermes update client --host-chain cosmoshub-4 --client client-X
hermes update client --host-chain osmosis-1 --client client-Y
```

## Шаг 10: Настройка мониторинга комиссий и доходности

```bash
# Создание скрипта для мониторинга баланса и комиссий
cat > $HOME/monitor_relayer.sh << 'EOF'
#!/bin/bash

COSMOS_ADDRESS="cosmos1your_address"
OSMOSIS_ADDRESS="osmo1your_address"

echo "=== Cosmos Hub Balance ==="
cosmos-hub-cli query bank balances $COSMOS_ADDRESS --node https://rpc.cosmos.network:443

echo "=== Osmosis Balance ==="
osmosis-cli query bank balances $OSMOSIS_ADDRESS --node https://rpc.osmosis.zone:443

# Проверка статуса релеера
echo "=== Hermes Status ==="
systemctl status hermes | grep Active

# Проверка последних логов
echo "=== Recent Logs ==="
tail -n 20 $HOME/hermes.log | grep -E "success|error|fee"
EOF

chmod +x $HOME/monitor_relayer.sh

# Настройка автоматического запуска скрипта мониторинга через cron
(crontab -l 2>/dev/null; echo "0 */6 * * * $HOME/monitor_relayer.sh > $HOME/relayer_stats.log") | crontab -

# Создание скрипта для оповещений (опционально)
cat > $HOME/alert_relayer.sh << 'EOF'
#!/bin/bash

MIN_ATOM=1000000  # 1 ATOM в uatom
MIN_OSMO=1000000  # 1 OSMO в uosmo

COSMOS_ADDRESS="cosmos1your_address"
OSMOSIS_ADDRESS="osmo1your_address"

# Получение балансов
ATOM_BALANCE=$(cosmos-hub-cli query bank balances $COSMOS_ADDRESS --node https://rpc.cosmos.network:443 -o json | jq -r '.balances[] | select(.denom=="uatom") | .amount')
OSMO_BALANCE=$(osmosis-cli query bank balances $OSMOSIS_ADDRESS --node https://rpc.osmosis.zone:443 -o json | jq -r '.balances[] | select(.denom=="uosmo") | .amount')

# Проверка статуса релеера
HERMES_STATUS=$(systemctl is-active hermes)

# Отправка оповещений, если баланс ниже минимума или релеер не работает
if [ "$ATOM_BALANCE" -lt "$MIN_ATOM" ] || [ "$OSMO_BALANCE" -lt "$MIN_OSMO" ] || [ "$HERMES_STATUS" != "active" ]; then
    echo "ALERT: Relayer needs attention!" > $HOME/relayer_alert.log
    echo "Cosmos Hub Balance: $ATOM_BALANCE uatom" >> $HOME/relayer_alert.log
    echo "Osmosis Balance: $OSMO_BALANCE uosmo" >> $HOME/relayer_alert.log
    echo "Hermes Status: $HERMES_STATUS" >> $HOME/relayer_alert.log
    
    # Здесь можно добавить отправку уведомлений через Telegram, Email и т.д.
    # Например, для Telegram:
    # curl -s -X POST https://api.telegram.org/bot<YOUR_BOT_TOKEN>/sendMessage -d chat_id=<YOUR_CHAT_ID> -d text="$(cat $HOME/relayer_alert.log)"
fi
EOF

chmod +x $HOME/alert_relayer.sh

# Настройка автоматического запуска скрипта оповещений через cron
(crontab -l 2>/dev/null; echo "*/30 * * * * $HOME/alert_relayer.sh") | crontab -
```

## Шаг 11: Оптимизация производительности релеера

### Настройка параметров системы для оптимальной производительности

```bash
cat > /etc/sysctl.d/99-relayer-tuning.conf << EOF
# Увеличение лимитов сетевых буферов
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Оптимизация TCP
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15
EOF

# Применение параметров
sudo sysctl -p /etc/sysctl.d/99-relayer-tuning.conf

# Настройка лимитов открытых файлов
cat > /etc/security/limits.d/99-relayer.conf << EOF
* soft nofile 65536
* hard nofile 65536
EOF

# Оптимизация параметров Hermes для высокой нагрузки
cat > $HOME/.hermes/config.toml.optimized << EOF
[global]
log_level = 'info'
clear_packets_interval = 50
tx_confirmation = true

[mode]
[mode.clients]
enabled = true
refresh = true
misbehaviour = true

[mode.connections]
enabled = true

[mode.channels]
enabled = true

[mode.packets]
enabled = true
clear_interval = 50
clear_on_start = true
tx_confirmation = true

[rest]
enabled = true
host = '127.0.0.1'
port = 3000

[telemetry]
enabled = true
host = '127.0.0.1'
port = 3001

# Настройки для Cosmos Hub с оптимизированными параметрами газа
[[chains]]
id = 'cosmoshub-4'
type = 'CosmosSdk'
rpc_addr = 'https://rpc.cosmos.network:443'
grpc_addr = 'https://grpc.cosmos.network:443'
websocket_addr = 'wss://rpc.cosmos.network:443/websocket'
rpc_timeout = '15s'
account_prefix = 'cosmos'
key_name = 'cosmos-key'
key_store_type = 'Test'
store_prefix = 'ibc'
default_gas = 150000
max_gas = 5000000
gas_price = { price = 0.025, denom = 'uatom' }
gas_multiplier = 1.5
max_msg_num = 50
max_tx_size = 180000
clock_drift = '10s'
max_block_time = '30s'
trusting_period = '14days'
trust_threshold = { numerator = '1', denominator = '3' }
address_type = { derivation = 'cosmos' }
memo_prefix = 'Relayed by Hermes'

[[chains.packet_filter]]
policy = 'allow'
list = [
  ['transfer', '*'],
]

[chains.packet_filter.min_fees]
enabled = true
recv_fee = [
  { amount = "100", denom = "uatom" },
]
ack_fee = [
  { amount = "100", denom = "uatom" },
]
timeout_fee = [
  { amount = "100", denom = "uatom" },
]

# Настройки для Osmosis с оптимизированными параметрами газа
[[chains]]
id = 'osmosis-1'
type = 'CosmosSdk'
rpc_addr = 'https://rpc.osmosis.zone:443'
grpc_addr = 'https://grpc.osmosis.zone:443'
websocket_addr = 'wss://rpc.osmosis.zone:443/websocket'
rpc_timeout = '15s'
account_prefix = 'osmo'
key_name = 'osmo-key'
key_store_type = 'Test'
store_prefix = 'ibc'
default_gas = 150000
max_gas = 5000000
gas_price = { price = 0.025, denom = 'uosmo' }
gas_multiplier = 1.5
max_msg_num = 50
max_tx_size = 180000
clock_drift = '10s'
max_block_time = '30s'
trusting_period = '14days'
trust_threshold = { numerator = '1', denominator = '3' }
address_type = { derivation = 'cosmos' }
memo_prefix = 'Relayed by Hermes'

[[chains.packet_filter]]
policy = 'allow'
list = [
  ['transfer', '*'],
]

[chains.packet_filter.min_fees]
enabled = true
recv_fee = [
  { amount = "100", denom = "uosmo" },
]
ack_fee = [
  { amount = "100", denom = "uosmo" },
]
timeout_fee = [
  { amount = "100", denom = "uosmo" },
]
EOF

# Применение оптимизированной конфигурации
cp $HOME/.hermes/config.toml.optimized $HOME/.hermes/config.toml

# Перезапуск сервиса с новой конфигурацией
sudo systemctl restart hermes
```

## Шаг 12: Настройка резервного копирования и восстановления

```bash
# Создание скрипта для резервного копирования
cat > $HOME/backup_relayer.sh << 'EOF'
#!/bin/bash

BACKUP_DIR="$HOME/hermes_backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/hermes_backup_$TIMESTAMP.tar.gz"

# Создание директории для резервных копий
mkdir -p $BACKUP_DIR

# Остановка сервиса перед резервным копированием
sudo systemctl stop hermes

# Создание резервной копии
tar -czf $BACKUP_FILE $HOME/.hermes

# Запуск сервиса после резервного копирования
sudo systemctl start hermes

# Удаление старых резервных копий (оставляем только 5 последних)
ls -t $BACKUP_DIR/hermes_backup_*.tar.gz | tail -n +6 | xargs -r rm

echo "Backup completed: $BACKUP_FILE"
EOF

chmod +x $HOME/backup_relayer.sh

# Настройка автоматического резервного копирования через cron
(crontab -l 2>/dev/null; echo "0 0 * * 0 $HOME/backup_relayer.sh > $HOME/backup.log 2>&1") | crontab -

# Создание скрипта для восстановления
cat > $HOME/restore_relayer.sh << 'EOF'
#!/bin/bash

# Проверка наличия аргумента с путем к файлу резервной копии
if [ $# -ne 1 ]; then
    echo "Usage: $0 <backup_file>"
    exit 1
fi

BACKUP_FILE=$1

# Проверка существования файла резервной копии
if [ ! -f "$BACKUP_FILE" ]; then
    echo "Backup file not found: $BACKUP_FILE"
    exit 1
fi

# Остановка сервиса перед восстановлением
sudo systemctl stop hermes

# Создание резервной копии текущей конфигурации
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
CURRENT_BACKUP="$HOME/hermes_current_$TIMESTAMP.tar.gz"
tar -czf $CURRENT_BACKUP $HOME/.hermes
echo "Current configuration backed up to: $CURRENT_BACKUP"

# Удаление текущей конфигурации
rm -rf $HOME/.hermes

# Восстановление из резервной копии
tar -xzf $BACKUP_FILE -C $HOME

# Запуск сервиса после восстановления
sudo systemctl start hermes

echo "Restoration completed from: $BACKUP_FILE"
EOF

chmod +x $HOME/restore_relayer.sh
```

## Шаг 13: Настройка автоматического обновления Hermes

```bash
# Создание скрипта для обновления Hermes
cat > $HOME/update_hermes.sh << 'EOF'
#!/bin/bash

# Версия Hermes для обновления (измените на нужную)
TARGET_VERSION="v1.7.4"

# Проверка текущей версии
CURRENT_VERSION=$(hermes version | grep -oP 'hermes \K[^ ]+')
echo "Current Hermes version: $CURRENT_VERSION"
echo "Target Hermes version: $TARGET_VERSION"

# Если текущая версия совпадает с целевой, выходим
if [ "$CURRENT_VERSION" = "$TARGET_VERSION" ]; then
    echo "Hermes is already at the target version. No update needed."
    exit 0
fi

# Создание резервной копии перед обновлением
$HOME/backup_relayer.sh

# Остановка сервиса
sudo systemctl stop hermes

# Переход в директорию с исходным кодом Hermes
cd $HOME/hermes

# Обновление репозитория
git fetch --all
git checkout $TARGET_VERSION

# Сборка и установка новой версии
cargo install --locked --path crates/relayer-cli

# Проверка новой версии
NEW_VERSION=$(hermes version | grep -oP 'hermes \K[^ ]+')
echo "Updated Hermes version: $NEW_VERSION"

# Запуск сервиса
sudo systemctl start hermes

echo "Hermes update completed successfully."
EOF

chmod +x $HOME/update_hermes.sh
```

## Шаг 14: Настройка расширенного мониторинга с Prometheus и Grafana

```bash
# Установка Prometheus
sudo apt-get update
sudo apt-get install -y prometheus prometheus-node-exporter

# Настройка Prometheus для сбора метрик Hermes
cat > /etc/prometheus/prometheus.yml << EOF
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'hermes'
    static_configs:
      - targets: ['localhost:3001']
EOF

# Перезапуск Prometheus
sudo systemctl restart prometheus

# Установка Grafana
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install -y grafana

# Запуск и включение Grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

echo "Prometheus и Grafana установлены. Доступ к Grafana: http://YOUR_SERVER_IP:3000 (логин/пароль по умолчанию: admin/admin)"
echo "Импортируйте дашборд для Hermes в Grafana, используя ID: 14151 или создайте свой собственный."
```


## Шаг 15: Настройка автоматического управления комиссиями

```bash
# Создание скрипта для динамического управления комиссиями
cat > $HOME/manage_fees.sh << 'EOF'
#!/bin/bash

# Параметры для настройки
MIN_FEE_ATOM=50    # Минимальная комиссия в uatom
MAX_FEE_ATOM=500   # Максимальная комиссия в uatom
MIN_FEE_OSMO=50    # Минимальная комиссия в uosmo
MAX_FEE_OSMO=500   # Максимальная комиссия в uosmo

# Функция для логирования
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $HOME/fee_management_detailed.log
}

log "Начало процесса управления комиссиями"

# Получение текущей загрузки сетей
log "Запрос ожидающих пакетов в Cosmos Hub..."
COSMOS_PENDING=$(hermes query packet pending --chain cosmoshub-4 --port transfer --channel channel-X 2>/dev/null | grep -c "packet" || echo "0")
log "Запрос ожидающих пакетов в Osmosis..."
OSMOSIS_PENDING=$(hermes query packet pending --chain osmosis-1 --port transfer --channel channel-Y 2>/dev/null | grep -c "packet" || echo "0")

# Получение текущей загрузки сети (альтернативный метод)
log "Получение данных о загрузке сети Cosmos Hub..."
COSMOS_TPS=$(curl -s https://rpc.cosmos.network/status | jq -r '.result.validator_info.voting_power' 2>/dev/null || echo "0")
log "Получение данных о загрузке сети Osmosis..."
OSMOSIS_TPS=$(curl -s https://rpc.osmosis.zone/status | jq -r '.result.validator_info.voting_power' 2>/dev/null || echo "0")

# Получение текущих цен газа
log "Получение текущей цены газа в Cosmos Hub..."
COSMOS_GAS_PRICE=$(curl -s https://api.cosmos.network/cosmos/tx/v1beta1/gas_prices | jq -r '.gas_prices[] | select(.denom=="uatom") | .amount' 2>/dev/null || echo "0.025")
log "Получение текущей цены газа в Osmosis..."
OSMOSIS_GAS_PRICE=$(curl -s https://api.osmosis.zone/cosmos/tx/v1beta1/gas_prices | jq -r '.gas_prices[] | select(.denom=="uosmo") | .amount' 2>/dev/null || echo "0.025")

# Расчет новых комиссий на основе загрузки
# Чем больше ожидающих пакетов, тем выше комиссия
calculate_fee() {
    local pending=$1
    local min_fee=$2
    local max_fee=$3
    local network_load=$4
    local gas_price=$5
    
    # Базовый расчет на основе ожидающих пакетов
    local base_fee
    if [ $pending -lt 10 ]; then
        base_fee=$min_fee
    elif [ $pending -gt 100 ]; then
        base_fee=$max_fee
    else
        # Линейная интерполяция между минимальной и максимальной комиссией
        base_fee=$(( min_fee + (max_fee - min_fee) * (pending - 10) / 90 ))
    fi
    
    # Корректировка на основе загрузки сети
    local load_factor=$(echo "scale=2; 1 + ($network_load / 10000)" | bc)
    
    # Корректировка на основе цены газа
    local gas_factor=$(echo "scale=2; $gas_price * 1000" | bc)
    
    # Финальный расчет комиссии
    local final_fee=$(echo "scale=0; $base_fee * $load_factor * $gas_factor / 1" | bc)
    
    # Проверка на минимальное и максимальное значение
    if [ $(echo "$final_fee < $min_fee" | bc) -eq 1 ]; then
        final_fee=$min_fee
    elif [ $(echo "$final_fee > $max_fee" | bc) -eq 1 ]; then
        final_fee=$max_fee
    fi
    
    echo $final_fee
}

COSMOS_FEE=$(calculate_fee $COSMOS_PENDING $MIN_FEE_ATOM $MAX_FEE_ATOM $COSMOS_TPS $COSMOS_GAS_PRICE)
OSMOSIS_FEE=$(calculate_fee $OSMOSIS_PENDING $MIN_FEE_OSMO $MAX_FEE_OSMO $OSMOSIS_TPS $OSMOSIS_GAS_PRICE)

log "Cosmos Hub: ожидающие пакеты: $COSMOS_PENDING, загрузка сети: $COSMOS_TPS, цена газа: $COSMOS_GAS_PRICE, новая комиссия: $COSMOS_FEE uatom"
log "Osmosis: ожидающие пакеты: $OSMOSIS_PENDING, загрузка сети: $OSMOSIS_TPS, цена газа: $OSMOSIS_GAS_PRICE, новая комиссия: $OSMOSIS_FEE uosmo"

# Создание резервной копии текущей конфигурации
cp $HOME/.hermes/config.toml $HOME/.hermes/config.toml.bak.$(date +%Y%m%d%H%M%S)
log "Создана резервная копия конфигурации"

# Обновление конфигурации с новыми комиссиями
log "Обновление конфигурации с новыми комиссиями..."

# Обновление комиссий для Cosmos Hub
sed -i "/\[chains\.packet_filter\.min_fees\]/,/\[/ s/recv_fee = \[\n.*{ amount = \"[0-9]*\", denom = \"uatom\" },/recv_fee = \[\n    { amount = \"$COSMOS_FEE\", denom = \"uatom\" },/" $HOME/.hermes/config.toml
sed -i "/\[chains\.packet_filter\.min_fees\]/,/\[/ s/ack_fee = \[\n.*{ amount = \"[0-9]*\", denom = \"uatom\" },/ack_fee = \[\n    { amount = \"$COSMOS_FEE\", denom = \"uatom\" },/" $HOME/.hermes/config.toml
sed -i "/\[chains\.packet_filter\.min_fees\]/,/\[/ s/timeout_fee = \[\n.*{ amount = \"[0-9]*\", denom = \"uatom\" },/timeout_fee = \[\n    { amount = \"$COSMOS_FEE\", denom = \"uatom\" },/" $HOME/.hermes/config.toml

# Обновление комиссий для Osmosis
sed -i "/\[chains\.packet_filter\.min_fees\]/,/\[/ s/recv_fee = \[\n.*{ amount = \"[0-9]*\", denom = \"uosmo\" },/recv_fee = \[\n    { amount = \"$OSMOSIS_FEE\", denom = \"uosmo\" },/" $HOME/.hermes/config.toml
sed -i "/\[chains\.packet_filter\.min_fees\]/,/\[/ s/ack_fee = \[\n.*{ amount = \"[0-9]*\", denom = \"uosmo\" },/ack_fee = \[\n    { amount = \"$OSMOSIS_FEE\", denom = \"uosmo\" },/" $HOME/.hermes/config.toml
sed -i "/\[chains\.packet_filter\.min_fees\]/,/\[/ s/timeout_fee = \[\n.*{ amount = \"[0-9]*\", denom = \"uosmo\" },/timeout_fee = \[\n    { amount = \"$OSMOSIS_FEE\", denom = \"uosmo\" },/" $HOME/.hermes/config.toml

# Проверка конфигурации
log "Проверка обновленной конфигурации..."
hermes config validate
if [ $? -ne 0 ]; then
    log "ОШИБКА: Конфигурация недействительна. Восстановление из резервной копии."
    cp $HOME/.hermes/config.toml.bak.$(date +%Y%m%d%H%M%S) $HOME/.hermes/config.toml
    exit 1
fi

# Перезапуск Hermes для применения новых настроек
log "Перезапуск Hermes для применения новых настроек..."
sudo systemctl restart hermes

# Проверка статуса после перезапуска
sleep 10
if systemctl is-active --quiet hermes; then
    log "Hermes успешно перезапущен с новыми настройками комиссий"
else
    log "ОШИБКА: Hermes не запустился после обновления конфигурации. Проверьте журналы системы."
    sudo systemctl status hermes
fi

# Сохранение истории комиссий для анализа
echo "$(date +%Y-%m-%d,%H:%M:%S),$COSMOS_PENDING,$COSMOS_TPS,$COSMOS_GAS_PRICE,$COSMOS_FEE,$OSMOSIS_PENDING,$OSMOSIS_TPS,$OSMOSIS_GAS_PRICE,$OSMOSIS_FEE" >> $HOME/fee_history.csv

log "Управление комиссиями завершено в $(date)"
EOF

chmod +x $HOME/manage_fees.sh

# Создание заголовка для файла истории комиссий
echo "date,cosmos_pending,cosmos_tps,cosmos_gas_price,cosmos_fee,osmosis_pending,osmosis_tps,osmosis_gas_price,osmosis_fee" > $HOME/fee_history.csv

# Настройка автоматического запуска скрипта управления комиссиями через cron
(crontab -l 2>/dev/null; echo "0 */4 * * * $HOME/manage_fees.sh > $HOME/fee_management.log 2>&1") | crontab -

# Создание скрипта для анализа истории комиссий
cat > $HOME/analyze_fees.sh << 'EOF'
#!/bin/bash

# Проверка наличия файла истории комиссий
if [ ! -f "$HOME/fee_history.csv" ]; then
    echo "Файл истории комиссий не найден!"
    exit 1
fi

# Подсчет количества записей
RECORDS=$(wc -l < $HOME/fee_history.csv)
echo "Всего записей в истории комиссий: $((RECORDS-1))"

# Анализ комиссий Cosmos Hub
echo "Анализ комиссий Cosmos Hub:"
AVG_COSMOS_FEE=$(tail -n +2 $HOME/fee_history.csv | awk -F, '{sum+=$5} END {print sum/(NR)}')
MIN_COSMOS_FEE=$(tail -n +2 $HOME/fee_history.csv | awk -F, '{print $5}' | sort -n | head -1)
MAX_COSMOS_FEE=$(tail -n +2 $HOME/fee_history.csv | awk -F, '{print $5}' | sort -n | tail -1)
echo "  Средняя комиссия: $AVG_COSMOS_FEE uatom"
echo "  Минимальная комиссия: $MIN_COSMOS_FEE uatom"
echo "  Максимальная комиссия: $MAX_COSMOS_FEE uatom"

# Анализ комиссий Osmosis
echo "Анализ комиссий Osmosis:"
AVG_OSMOSIS_FEE=$(tail -n +2 $HOME/fee_history.csv | awk -F, '{sum+=$9} END {print sum/(NR)}')
MIN_OSMOSIS_FEE=$(tail -n +2 $HOME/fee_history.csv | awk -F, '{print $9}' | sort -n | head -1)
MAX_OSMOSIS_FEE=$(tail -n +2 $HOME/fee_history.csv | awk -F, '{print $9}' | sort -n | tail -1)
echo "  Средняя комиссия: $AVG_OSMOSIS_FEE uosmo"
echo "  Минимальная комиссия: $MIN_OSMOSIS_FEE uosmo"
echo "  Максимальная комиссия: $MAX_OSMOSIS_FEE uosmo"

# Анализ корреляции между загрузкой и комиссиями
echo "Корреляция между ожидающими пакетами и комиссиями:"
echo "  Cosmos Hub: $(tail -n +2 $HOME/fee_history.csv | awk -F, 'BEGIN {sum_x=0; sum_y=0; sum_xy=0; sum_x2=0; sum_y2=0; n=0} {sum_x+=$2; sum_y+=$5; sum_xy+=$2*$5; sum_x2+=$2*$2; sum_y2+=$5*$5; n++} END {print (n*sum_xy-sum_x*sum_y)/sqrt((n*sum_x2-sum_x*sum_x)*(n*sum_y2-sum_y*sum_y))}')"
echo "  Osmosis: $(tail -n +2 $HOME/fee_history.csv | awk -F, 'BEGIN {sum_x=0; sum_y=0; sum_xy=0; sum_x2=0; sum_y2=0; n=0} {sum_x+=$6; sum_y+=$9; sum_xy+=$6*$9; sum_x2+=$6*$6; sum_y2+=$9*$9; n++} END {print (n*sum_xy-sum_x*sum_y)/sqrt((n*sum_x2-sum_x*sum_x)*(n*sum_y2-sum_y*sum_y))}')"

# Вывод последних 5 записей
echo "Последние 5 записей в истории комиссий:"
tail -n 5 $HOME/fee_history.csv | column -t -s,
EOF

chmod +x $HOME/analyze_fees.sh

# Запуск скрипта управления комиссиями в первый раз
$HOME/manage_fees.sh
```

## Шаг 16: Настройка мониторинга и оповещений

```bash
# Установка необходимых инструментов для мониторинга
sudo apt-get update
sudo apt-get install -y prometheus prometheus-node-exporter grafana

# Создание конфигурации Prometheus для мониторинга Hermes
cat > /etc/prometheus/prometheus.yml << EOF
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'hermes'
    metrics_path: /metrics
    static_configs:
      - targets: ['localhost:3001']
EOF

# Перезапуск Prometheus для применения новой конфигурации
sudo systemctl restart prometheus

# Создание скрипта для мониторинга состояния Hermes
cat > $HOME/monitor_hermes.sh << 'EOF'
#!/bin/bash

# Параметры
LOG_FILE="$HOME/hermes_monitor.log"
ALERT_THRESHOLD=5
ALERT_EMAIL="your_email@example.com"
TELEGRAM_BOT_TOKEN="YOUR_TELEGRAM_BOT_TOKEN"
TELEGRAM_CHAT_ID="YOUR_TELEGRAM_CHAT_ID"

# Функция для логирования
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

# Функция для отправки оповещений
send_alert() {
    local message="$1"
    log "ALERT: $message"
    
    # Отправка по электронной почте
    echo "$message" | mail -s "Hermes IBC Relayer Alert" $ALERT_EMAIL
    
    # Отправка в Telegram
    curl -s -X POST "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/sendMessage" \
        -d chat_id="$TELEGRAM_CHAT_ID" \
        -d text="🚨 IBC Relayer Alert: $message" \
        -d parse_mode="Markdown"
}

# Проверка статуса службы Hermes
check_hermes_service() {
    if ! systemctl is-active --quiet hermes; then
        send_alert "Hermes service is not running! Attempting to restart..."
        sudo systemctl restart hermes
        sleep 10
        if ! systemctl is-active --quiet hermes; then
            send_alert "Failed to restart Hermes service. Manual intervention required."
        else
            send_alert "Hermes service successfully restarted."
        fi
    fi
}

# Проверка состояния каналов
check_channels() {
    # Проверка канала Cosmos Hub -> Osmosis
    local cosmos_channel_status=$(hermes query channel end --chain cosmoshub-4 --port transfer --channel channel-X 2>&1)
    if echo "$cosmos_channel_status" | grep -q "error"; then
        send_alert "Issue with Cosmos Hub channel: $cosmos_channel_status"
    fi
    
    # Проверка канала Osmosis -> Cosmos Hub
    local osmosis_channel_status=$(hermes query channel end --chain osmosis-1 --port transfer --channel channel-Y 2>&1)
    if echo "$osmosis_channel_status" | grep -q "error"; then
        send_alert "Issue with Osmosis channel: $osmosis_channel_status"
    fi
}

# Проверка ожидающих пакетов
check_pending_packets() {
    # Проверка ожидающих пакетов в Cosmos Hub
    local cosmos_pending=$(hermes query packet pending --chain cosmoshub-4 --port transfer --channel channel-X 2>/dev/null | grep -c "packet" || echo "0")
    
    # Проверка ожидающих пакетов в Osmosis
    local osmosis_pending=$(hermes query packet pending --chain osmosis-1 --port transfer --channel channel-Y 2>/dev/null | grep -c "packet" || echo "0")
    
    log "Pending packets - Cosmos Hub: $cosmos_pending, Osmosis: $osmosis_pending"
    
    # Оповещение, если количество ожидающих пакетов превышает порог
    if [ "$cosmos_pending" -gt "$ALERT_THRESHOLD" ]; then
        send_alert "High number of pending packets on Cosmos Hub: $cosmos_pending"
    fi
    
    if [ "$osmosis_pending" -gt "$ALERT_THRESHOLD" ]; then
        send_alert "High number of pending packets on Osmosis: $osmosis_pending"
    fi
}

# Проверка баланса кошельков релеера
check_wallet_balances() {
    # Получение адреса кошелька для Cosmos Hub
    local cosmos_address=$(hermes keys show --chain cosmoshub-4 2>/dev/null | grep "address:" | awk '{print $2}')
    
    # Получение адреса кошелька для Osmosis
    local osmosis_address=$(hermes keys show --chain osmosis-1 2>/dev/null | grep "address:" | awk '{print $2}')
    
    # Проверка баланса Cosmos Hub
    local cosmos_balance=$(gaiad query bank balances $cosmos_address --node https://rpc.cosmos.network:443 -o json 2>/dev/null | jq -r '.balances[] | select(.denom=="uatom") | .amount' || echo "0")
    
    # Проверка баланса Osmosis
    local osmosis_balance=$(osmosisd query bank balances $osmosis_address --node https://rpc.osmosis.zone:443 -o json 2>/dev/null | jq -r '.balances[] | select(.denom=="uosmo") | .amount' || echo "0")
    
    log "Wallet balances - Cosmos Hub: $cosmos_balance uatom, Osmosis: $osmosis_balance uosmo"
    
    # Пороговые значения для оповещений (в микроединицах)
    local cosmos_threshold=1000000  # 1 ATOM
    local osmosis_threshold=1000000  # 1 OSMO
    
    if [ "$cosmos_balance" -lt "$cosmos_threshold" ]; then
        send_alert "Low balance on Cosmos Hub wallet: $cosmos_balance uatom"
    fi
    
    if [ "$osmosis_balance" -lt "$osmosis_threshold" ]; then
        send_alert "Low balance on Osmosis wallet: $osmosis_balance uosmo"
    fi
}

# Проверка доходов от комиссий
check_fee_earnings() {
    # Получение адреса кошелька для Cosmos Hub
    local cosmos_address=$(hermes keys show --chain cosmoshub-4 2>/dev/null | grep "address:" | awk '{print $2}')
    
    # Получение адреса кошелька для Osmosis
    local osmosis_address=$(hermes keys show --chain osmosis-1 2>/dev/null | grep "address:" | awk '{print $2}')
    
    # Получение истории транзакций для анализа комиссий
    local cosmos_txs=$(gaiad query txs --events "transfer.recipient=$cosmos_address" --node https://rpc.cosmos.network:443 -o json 2>/dev/null)
    local osmosis_txs=$(osmosisd query txs --events "transfer.recipient=$osmosis_address" --node https://rpc.osmosis.zone:443 -o json 2>/dev/null)
    
    # Анализ доходов от комиссий (упрощенно)
    local cosmos_fee_count=$(echo "$cosmos_txs" | jq -r '.total_count' 2>/dev/null || echo "0")
    local osmosis_fee_count=$(echo "$osmosis_txs" | jq -r '.total_count' 2>/dev/null || echo "0")
    
    log "Fee transactions - Cosmos Hub: $cosmos_fee_count, Osmosis: $osmosis_fee_count"
}

# Основная функция мониторинга
main() {
    log "Starting Hermes monitoring..."
    
    # Проверка статуса службы
    check_hermes_service
    
    # Проверка состояния каналов
    check_channels
    
    # Проверка ожидающих пакетов
    check_pending_packets
    
    # Проверка баланса кошельков
    check_wallet_balances
    
    # Проверка доходов от комиссий
    check_fee_earnings
    
    log "Monitoring completed."
}

# Запуск основной функции
main
EOF

chmod +x $HOME/monitor_hermes.sh

# Настройка автоматического запуска скрипта мониторинга через cron
(crontab -l 2>/dev/null; echo "*/15 * * * * $HOME/monitor_hermes.sh > /dev/null 2>&1") | crontab -

# Создание дашборда Grafana для Hermes
cat > $HOME/hermes_dashboard.json << 'EOF'
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": "-- Grafana --",
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "gnetId": null,
  "graphTooltip": 0,
  "id": 1,
  "links": [],
  "panels": [
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "Prometheus",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 0
      },
      "hiddenSeries": false,
      "id": 2,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.7",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "hermes_client_updates_total",
          "interval": "",
          "legendFormat": "Client Updates",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "IBC Client Updates",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "Prometheus",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 0
      },
      "hiddenSeries": false,
      "id": 4,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.7",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "hermes_packets_relayed_total",
          "interval": "",
          "legendFormat": "Packets Relayed",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "IBC Packets Relayed",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    }
  ],
  "refresh": "5s",
  "schemaVersion": 26,
  "style": "dark",
  "tags": [],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-6h",
    "to": "now"
  },
  "timepicker": {},
  "timezone": "",
  "title": "Hermes IBC Relayer Dashboard",
  "uid": "hermes_ibc",
  "version": 1
}
EOF


# Настройка Grafana для импорта дашборда (продолжение)
sudo systemctl enable grafana-server
sudo systemctl start grafana-server

# Создание API ключа для автоматического импорта дашборда
GRAFANA_API_KEY=$(curl -X POST -H "Content-Type: application/json" -d '{"name":"auto-import", "role": "Admin"}' http://admin:admin@localhost:3000/api/auth/keys | jq -r .key)

# Импорт дашборда в Grafana
curl -X POST \
  -H "Authorization: Bearer $GRAFANA_API_KEY" \
  -H "Content-Type: application/json" \
  -d @$HOME/hermes_dashboard.json \
  http://localhost:3000/api/dashboards/db

# Создание скрипта для проверки здоровья релеера
cat > $HOME/health_check.sh << 'EOF'
#!/bin/bash

# Проверка статуса Hermes
hermes_status=$(systemctl is-active hermes)
if [ "$hermes_status" != "active" ]; then
    echo "CRITICAL: Hermes service is not running!"
    exit 2
fi

# Проверка соединения с цепями
cosmos_status=$(hermes query client state --chain cosmoshub-4 --client 07-tendermint-0 2>&1)
if echo "$cosmos_status" | grep -q "error"; then
    echo "WARNING: Cannot connect to Cosmos Hub: $cosmos_status"
    exit 1
fi

osmosis_status=$(hermes query client state --chain osmosis-1 --client 07-tendermint-0 2>&1)
if echo "$osmosis_status" | grep -q "error"; then
    echo "WARNING: Cannot connect to Osmosis: $osmosis_status"
    exit 1
fi

# Проверка задержки клиентов
cosmos_delay=$(hermes query client consensus --chain cosmoshub-4 --client 07-tendermint-0 2>&1 | grep -o "latest_height=.*" | cut -d'=' -f2)
osmosis_delay=$(hermes query client consensus --chain osmosis-1 --client 07-tendermint-0 2>&1 | grep -o "latest_height=.*" | cut -d'=' -f2)

if [ -z "$cosmos_delay" ] || [ -z "$osmosis_delay" ]; then
    echo "WARNING: Cannot determine client delays"
    exit 1
fi

echo "OK: Hermes is running properly. Cosmos Hub delay: $cosmos_delay, Osmosis delay: $osmosis_delay"
exit 0
EOF

chmod +x $HOME/health_check.sh

# Настройка Prometheus Node Exporter для пользовательских метрик
cat > $HOME/node_exporter_textfile.sh << 'EOF'
#!/bin/bash

METRICS_DIR="/var/lib/node_exporter/textfile_collector"
sudo mkdir -p $METRICS_DIR

# Функция для получения баланса
get_balance() {
    local chain=$1
    local address=$2
    local denom=$3
    local node=$4
    
    if [ "$chain" == "cosmoshub" ]; then
        gaiad query bank balances $address --node $node -o json 2>/dev/null | \
        jq -r --arg denom "$denom" '.balances[] | select(.denom==$denom) | .amount' || echo "0"
    elif [ "$chain" == "osmosis" ]; then
        osmosisd query bank balances $address --node $node -o json 2>/dev/null | \
        jq -r --arg denom "$denom" '.balances[] | select(.denom==$denom) | .amount' || echo "0"
    else
        echo "0"
    fi
}

# Получение адресов кошельков
cosmos_address=$(hermes keys show --chain cosmoshub-4 2>/dev/null | grep "address:" | awk '{print $2}')
osmosis_address=$(hermes keys show --chain osmosis-1 2>/dev/null | grep "address:" | awk '{print $2}')

# Получение балансов
cosmos_balance=$(get_balance "cosmoshub" "$cosmos_address" "uatom" "https://rpc.cosmos.network:443")
osmosis_balance=$(get_balance "osmosis" "$osmosis_address" "uosmo" "https://rpc.osmosis.zone:443")

# Запись метрик в файл
cat > $METRICS_DIR/hermes_balances.prom << EOF
# HELP hermes_wallet_balance Current balance of the relayer wallet
# TYPE hermes_wallet_balance gauge
hermes_wallet_balance{chain="cosmoshub",denom="uatom"} $cosmos_balance
hermes_wallet_balance{chain="osmosis",denom="uosmo"} $osmosis_balance
EOF

# Получение статистики по пакетам
cosmos_pending=$(hermes query packet pending --chain cosmoshub-4 --port transfer --channel channel-X 2>/dev/null | grep -c "packet" || echo "0")
osmosis_pending=$(hermes query packet pending --chain osmosis-1 --port transfer --channel channel-Y 2>/dev/null | grep -c "packet" || echo "0")

# Запись метрик пакетов в файл
cat > $METRICS_DIR/hermes_packets.prom << EOF
# HELP hermes_pending_packets Number of pending IBC packets
# TYPE hermes_pending_packets gauge
hermes_pending_packets{chain="cosmoshub",direction="outgoing"} $cosmos_pending
hermes_pending_packets{chain="osmosis",direction="outgoing"} $osmosis_pending
EOF

# Получение статуса здоровья
health_status=$($HOME/health_check.sh > /dev/null 2>&1; echo $?)

# Запись метрики здоровья в файл
cat > $METRICS_DIR/hermes_health.prom << EOF
# HELP hermes_health_status Health status of the Hermes relayer (0=OK, 1=WARNING, 2=CRITICAL)
# TYPE hermes_health_status gauge
hermes_health_status $health_status
EOF
EOF

chmod +x $HOME/node_exporter_textfile.sh

# Настройка cron для регулярного обновления метрик
(crontab -l 2>/dev/null; echo "*/5 * * * * $HOME/node_exporter_textfile.sh > /dev/null 2>&1") | crontab -

# Настройка Node Exporter для чтения пользовательских метрик
sudo sed -i 's/^ExecStart=.*/ExecStart=\/usr\/bin\/prometheus-node-exporter --collector.textfile.directory=\/var\/lib\/node_exporter\/textfile_collector/' /etc/systemd/system/prometheus-node-exporter.service

sudo systemctl daemon-reload
sudo systemctl restart prometheus-node-exporter

# Создание скрипта для резервного копирования ключей и конфигурации
cat > $HOME/backup_hermes.sh << 'EOF'
#!/bin/bash

BACKUP_DIR="$HOME/hermes_backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/hermes_backup_$TIMESTAMP.tar.gz"

# Создание директории для резервных копий
mkdir -p $BACKUP_DIR

# Остановка службы Hermes перед резервным копированием
sudo systemctl stop hermes

# Создание резервной копии
tar -czf $BACKUP_FILE \
    $HOME/.hermes/config.toml \
    $HOME/.hermes/keys \
    $HOME/hermes_service.sh \
    $HOME/monitor_hermes.sh \
    $HOME/health_check.sh \
    $HOME/node_exporter_textfile.sh

# Запуск службы Hermes после резервного копирования
sudo systemctl start hermes

# Удаление старых резервных копий (оставляем только 7 последних)
ls -t $BACKUP_DIR/hermes_backup_*.tar.gz | tail -n +8 | xargs -r rm

echo "Backup completed: $BACKUP_FILE"
EOF

chmod +x $HOME/backup_hermes.sh

# Настройка еженедельного резервного копирования
(crontab -l 2>/dev/null; echo "0 0 * * 0 $HOME/backup_hermes.sh > /dev/null 2>&1") | crontab -

# Создание скрипта для автоматического обновления Hermes
cat > $HOME/update_hermes.sh << 'EOF'
#!/bin/bash

# Параметры
LOG_FILE="$HOME/hermes_update.log"
BACKUP_SCRIPT="$HOME/backup_hermes.sh"
HERMES_REPO="https://github.com/informalsystems/hermes.git"
HERMES_VERSION="v1.7.4"  # Обновите до нужной версии

# Функция для логирования
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

# Создание резервной копии перед обновлением
log "Creating backup before update..."
$BACKUP_SCRIPT
if [ $? -ne 0 ]; then
    log "Backup failed. Aborting update."
    exit 1
fi

# Остановка службы Hermes
log "Stopping Hermes service..."
sudo systemctl stop hermes

# Клонирование репозитория и сборка новой версии
log "Cloning Hermes repository..."
cd $HOME
git clone $HERMES_REPO hermes_update
cd hermes_update
git checkout $HERMES_VERSION

log "Building Hermes $HERMES_VERSION..."
cargo build --release --bin hermes

# Проверка успешности сборки
if [ $? -ne 0 ]; then
    log "Build failed. Reverting to previous version."
    cd $HOME
    rm -rf hermes_update
    sudo systemctl start hermes
    exit 1
fi

# Замена исполняемого файла
log "Installing new Hermes binary..."
sudo cp $HOME/hermes_update/target/release/hermes /usr/local/bin/hermes
sudo chmod +x /usr/local/bin/hermes

# Очистка
cd $HOME
rm -rf hermes_update

# Запуск службы Hermes
log "Starting Hermes service with new version..."
sudo systemctl start hermes

# Проверка статуса
sleep 10
if systemctl is-active --quiet hermes; then
    log "Update completed successfully. Hermes $HERMES_VERSION is now running."
else
    log "Error: Hermes service failed to start after update."
    exit 1
fi

# Проверка версии
hermes_version=$(hermes version)
log "Installed Hermes version: $hermes_version"
EOF

chmod +x $HOME/update_hermes.sh

echo "Мониторинг и оповещения настроены успешно. Grafana доступна по адресу http://YOUR_SERVER_IP:3000 (логин: admin, пароль: admin)"
```

## Шаг 17: Настройка безопасности и оптимизация производительности

```bash
# Настройка файрвола для защиты сервера
sudo apt-get install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Разрешение только необходимых портов
sudo ufw allow ssh
sudo ufw allow 3000/tcp  # Grafana
sudo ufw allow 9090/tcp  # Prometheus
sudo ufw allow 26656/tcp  # Tendermint P2P
sudo ufw allow 26657/tcp  # Tendermint RPC

# Включение файрвола
sudo ufw --force enable

# Настройка fail2ban для защиты от брутфорс-атак
sudo apt-get install -y fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Создание конфигурации для оптимизации системы
cat > /etc/sysctl.d/99-hermes-tuning.conf << EOF
# Увеличение лимитов для сетевых соединений
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65536
net.ipv4.tcp_max_syn_backlog = 65536
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 10

# Оптимизация использования памяти
vm.swappiness = 10
vm.vfs_cache_pressure = 50
EOF

# Применение настроек системы
sudo sysctl -p /etc/sysctl.d/99-hermes-tuning.conf

# Настройка лимитов открытых файлов
cat > /etc/security/limits.d/hermes.conf << EOF
# Увеличение лимитов открытых файлов для пользователя
$USER soft nofile 65536
$USER hard nofile 65536
EOF

# Оптимизация конфигурации Hermes для производительности
cat > $HOME/.hermes/config.toml.optimized << 'EOF'
[global]
log_level = 'info'

# Увеличение тайм-аутов для улучшения стабильности
[rest]
enabled = true
host = '127.0.0.1'
port = 3000

[telemetry]
enabled = true
host = '127.0.0.1'
port = 3001

[mode]
clients = 'enabled'
connections = 'enabled'
channels = 'enabled'
packets = 'enabled'

# Оптимизация параметров для улучшения производительности
[mode.packets.filter]
min_fees = [
    { denom = 'uatom', amount = '100' },
    { denom = 'uosmo', amount = '100' },
]

[chains]
# Конфигурация для Cosmos Hub
[chains.cosmoshub-4]
id = 'cosmoshub-4'
rpc_addr = 'https://rpc.cosmos.network:443'
grpc_addr = 'https://grpc.cosmos.network:443'
websocket_addr = 'wss://rpc.cosmos.network:443/websocket'
rpc_timeout = '30s'
account_prefix = 'cosmos'
key_name = 'cosmos'
store_prefix = 'ibc'
default_gas = 300000
max_gas = 3000000
gas_price = { price = 0.025, denom = 'uatom' }
gas_multiplier = 1.3
max_msg_num = 30
max_tx_size = 180000
clock_drift = '15s'
max_block_time = '30s'
trusting_period = '14days'
trust_threshold = { numerator = '1', denominator = '3' }
memo_prefix = 'Relayed by Hermes'
[chains.cosmoshub-4.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-X'],
]

# Конфигурация для Osmosis
[chains.osmosis-1]
id = 'osmosis-1'
rpc_addr = 'https://rpc.osmosis.zone:443'
grpc_addr = 'https://grpc.osmosis.zone:443'
websocket_addr = 'wss://rpc.osmosis.zone:443/websocket'
rpc_timeout = '30s'
account_prefix = 'osmo'
key_name = 'osmosis'
store_prefix = 'ibc'
default_gas = 300000
max_gas = 3000000
gas_price = { price = 0.025, denom = 'uosmo' }
gas_multiplier = 1.3
max_msg_num = 30
max_tx_size = 180000
clock_drift = '15s'
max_block_time = '30s'
trusting_period = '14days'
trust_threshold = { numerator = '1', denominator = '3' }
memo_prefix = 'Relayed by Hermes'
[chains.osmosis-1.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-Y'],
]
EOF

# Сравнение конфигураций и применение оптимизированной версии
diff -u $HOME/.hermes/config.toml $HOME/.hermes/config.toml.optimized
read -p "Применить оптимизированную конфигурацию? (y/n): " apply_config
if [ "$apply_config" = "y" ]; then
  cp $HOME/.hermes/config.toml.optimized $HOME/.hermes/config.toml
  sudo systemctl restart hermes
  echo "Оптимизированная конфигурация применена"
else
  echo "Оптимизированная конфигурация сохранена как $HOME/.hermes/config.toml.optimized"
fi

# Настройка автоматической очистки логов
cat > /etc/logrotate.d/hermes << EOF
$HOME/hermes.log {
  daily
  rotate 7
  compress
  delaycompress
  missingok
  notifempty
  create 0640 $USER $USER
  postrotate
    systemctl reload hermes >/dev/null 2>&1 || true
  endscript
}
EOF

# Создание скрипта для проверки и восстановления соединений
cat > $HOME/check_connections.sh << 'EOF'
#!/bin/bash

LOG_FILE="$HOME/connection_check.log"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> $LOG_FILE
}

log "Starting connection check..."

# Проверка клиентов
check_client() {
  local chain=$1
  local client=$2
  
  status=$(hermes query client state --chain $chain --client $client 2>&1)
  
  if echo "$status" | grep -q "error"; then
    log "Client issue detected on $chain for $client"
    log "Attempting to update client..."
    hermes update client --host-chain $chain --client $client
    return 1
  fi
  
  return 0
}

# Проверка соединений
check_connection() {
  local chain_a=$1
  local chain_b=$2
  local conn_id=$3
  
  status=$(hermes query connection end --chain $chain_a --connection $conn_id 2>&1)
  
  if echo "$status" | grep -q "error"; then
    log "Connection issue detected on $chain_a for $conn_id"
    return 1
  fi
  
  return 0
}

# Проверка каналов
check_channel() {
  local chain=$1
  local port=$2
  local channel=$3
  
  status=$(hermes query channel end --chain $chain --port $port --channel $channel 2>&1)
  
  if echo "$status" | grep -q "error"; then
    log "Channel issue detected on $chain for $port/$channel"
    return 1
  fi
  
  return 0
}

# Проверка клиентов
client_issues=0
check_client "cosmoshub-4" "07-tendermint-0" || ((client_issues++))
check_client "osmosis-1" "07-tendermint-0" || ((client_issues++))

# Проверка соединений
connection_issues=0
check_connection "cosmoshub-4" "osmosis-1" "connection-0" || ((connection_issues++))
check_connection "osmosis-1" "cosmoshub-4" "connection-0" || ((connection_issues++))

# Проверка каналов
channel_issues=0
check_channel "cosmoshub-4" "transfer" "channel-X" || ((channel_issues++))
check_channel "osmosis-1" "transfer" "channel-Y" || ((channel_issues++))

# Проверка наличия зависших пакетов
pending_packets_cosmos=$(hermes query packet pending --chain cosmoshub-4 --port transfer --channel channel-X 2>/dev/null | grep -c "packet" || echo "0")
pending_packets_osmosis=$(hermes query packet pending --chain osmosis-1 --port transfer --channel channel-Y 2>/dev/null | grep -c "packet" || echo "0")

if [ "$pending_packets_cosmos" -gt 0 ] || [ "$pending_packets_osmosis" -gt 0 ]; then
  log "Pending packets detected: Cosmos Hub ($pending_packets_cosmos), Osmosis ($pending_packets_osmosis)"
  log "Attempting to clear packets..."
  
  hermes clear packets --chain cosmoshub-4 --port transfer --channel channel-X
  hermes clear packets --chain osmosis-1 --port transfer --channel channel-Y
fi

total_issues=$((client_issues + connection_issues + channel_issues))
if [ "$total_issues" -gt 0 ]; then
  log "Total issues found: $total_issues. Restarting Hermes service..."
  sudo systemctl restart hermes
else
  log "No issues found. Hermes is running properly."
fi
EOF

chmod +x $HOME/check_connections.sh

# Настройка регулярной проверки соединений
(crontab -l 2>/dev/null; echo "*/30 * * * * $HOME/check_connections.sh > /dev/null 2>&1") | crontab -

# Создание скрипта для мониторинга использования ресурсов
cat > $HOME/monitor_resources.sh << 'EOF'
#!/bin/bash

LOG_FILE="$HOME/resource_usage.log"
THRESHOLD_CPU=80
THRESHOLD_MEM=80
THRESHOLD_DISK=85

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> $LOG_FILE
}

# Получение использования CPU
cpu_usage=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')
cpu_usage_int=${cpu_usage%.*}

# Получение использования памяти
mem_usage=$(free | grep Mem | awk '{print $3/$2 * 100.0}')
mem_usage_int=${mem_usage%.*}

# Получение использования диска
disk_usage=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')

log "Resource usage - CPU: ${cpu_usage}%, Memory: ${mem_usage}%, Disk: ${disk_usage}%"

# Проверка превышения пороговых значений
if [ "$cpu_usage_int" -gt "$THRESHOLD_CPU" ]; then
  log "WARNING: High CPU usage detected: ${cpu_usage}%"
fi

if [ "$mem_usage_int" -gt "$THRESHOLD_MEM" ]; then
  log "WARNING: High memory usage detected: ${mem_usage}%"
fi

if [ "$disk_usage" -gt "$THRESHOLD_DISK" ]; then
  log "WARNING: High disk usage detected: ${disk_usage}%"
fi

# Проверка использования ресурсов процессом Hermes
hermes_pid=$(pgrep hermes)
if [ -n "$hermes_pid" ]; then
  hermes_cpu=$(ps -p $hermes_pid -o %cpu | tail -n 1 | tr -d ' ')
  hermes_mem=$(ps -p $hermes_pid -o %mem | tail -n 1 | tr -d ' ')
  log "Hermes process - CPU: ${hermes_cpu}%, Memory: ${hermes_mem}%"
  
  # Перезапуск Hermes при чрезмерном использовании ресурсов
  if (( $(echo "$hermes_cpu > 50" | bc -l) )) || (( $(echo "$hermes_mem > 40" | bc -l) )); then
    log "WARNING: Hermes is using excessive resources. Restarting service..."
    sudo systemctl restart hermes
  fi
else
  log "ERROR: Hermes process not found!"
  sudo systemctl start hermes
fi
EOF

chmod +x $HOME/monitor_resources.sh

# Настройка регулярного мониторинга ресурсов
(crontab -l 2>/dev/null; echo "*/15 * * * * $HOME/monitor_resources.sh > /dev/null 2>&1") | crontab -

echo "Настройка безопасности и оптимизация производительности завершены успешно."

# Финальное сообщение
echo "=========================================================="
echo "Установка и настройка Hermes релеера завершена успешно!"
echo "=========================================================="
echo "Важные команды:"
echo "- Проверка статуса: sudo systemctl status hermes"
echo "- Просмотр логов: journalctl -u hermes -f"
echo "- Мониторинг: http://YOUR_SERVER_IP:3000 (Grafana)"
echo "- Резервное копирование: $HOME/backup_hermes.sh"
echo "- Проверка соединений: $HOME/check_connections.sh"
echo "- Мониторинг ресурсов: $HOME/monitor_resources.sh"
echo "- Обновление Hermes: $HOME/update_hermes.sh"
echo ""
echo "Не забудьте сохранить мнемоническую фразу в безопасном месте!"
echo "=========================================================="
