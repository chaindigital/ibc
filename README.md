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
