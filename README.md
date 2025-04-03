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

```bash
# Настройка параметров системы для оптимальной производительности
cat > /etc/sysctl.d/99-relayer-tuning.conf << EOF
# Увеличение лимитов сетевых буферов
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Оптимизация TCP
net.ipv4.tcp_fastopen = 3
net
