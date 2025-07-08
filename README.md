# XOSD Full Auto Install Script
# System Requirements: Ubuntu 20.04 LTS / 6+ Core / 32GB RAM / 4TB NVMe
# Chain ID: xos_1267-1 / Binary: v0.5.2 / Port Prefix: 33
#

# 
sudo systemctl stop xosd
sudo systemctl disable xosd
sudo rm /etc/systemd/system/xosd.service
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
rm -rf ~/.xosd ~/.xosd/config ~/.xosd/data ~/.xosd/keyring-test ~/go/bin/xosd /usr/local/bin/xosd
#

# Update & install dependencies
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y

# Install Golang
cd $HOME
GO_VERSION="1.23.4"
wget "https://golang.org/dl/go${GO_VERSION}.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go${GO_VERSION}.linux-amd64.tar.gz"
rm "go${GO_VERSION}.linux-amd64.tar.gz"

# Setup environment
cat <<EOF > ~/.bash_profile
export PATH=\$PATH:/usr/local/go/bin:\$HOME/go/bin
export WALLET="wallet"
export MONIKER="MaouamNodelab"
export XOSD_CHAIN_ID="xos_1267-1"
export XOSD_PORT="33"
EOF

source ~/.bash_profile
mkdir -p ~/go/bin

# Install xosd
cd $HOME
wget https://github.com/xos-labs/node/releases/download/v0.5.2/node_0.5.2_Linux_amd64.tar.gz
tar -xzf node_0.5.2_Linux_amd64.tar.gz
sudo mv ./bin/xosd /usr/local/bin/
chmod +x /usr/local/bin/xosd
rm -rf node_0.5.2_Linux_amd64.tar.gz bin

# Init config
xosd config set node tcp://localhost:${XOSD_PORT}657
xosd config keyring-backend os
xosd config chain-id $XOSD_CHAIN_ID
xosd init "$MONIKER" --chain-id $XOSD_CHAIN_ID

# Download genesis
wget https://raw.githubusercontent.com/xos-labs/networks/refs/heads/main/testnet/genesis.json -O ~/.xosd/config/genesis.json

# Add peers
PEERS="c8297e8fcff832fbe2c2c5a53709480c11240332@199.85.209.4:26656,32badb9649620b3fc87b469ed124551dd0d7ec9d@199.85.208.177:26656,6835f9864136b7dc1e21e4e50c89516a112722d7@203.161.32.223:26656"
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" ~/.xosd/config/config.toml

# Port dan config ganti
sed -i.bak -e "s%:1317%:${XOSD_PORT}317%g; s%:8080%:${XOSD_PORT}080%g; s%:9090%:${XOSD_PORT}090%g; s%:9091%:${XOSD_PORT}091%g; s%:8545%:${XOSD_PORT}545%g; s%:8546%:${XOSD_PORT}546%g; s%:6065%:${XOSD_PORT}065%g" ~/.xosd/config/app.toml
sed -i.bak -e "s%:26658%:${XOSD_PORT}658%g; s%:26657%:${XOSD_PORT}657%g; s%:6060%:${XOSD_PORT}060%g; s%:26656%:${XOSD_PORT}656%g; s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${XOSD_PORT}656\"%; s%:26660%:${XOSD_PORT}660%g" ~/.xosd/config/config.toml

# Pruning & gas & prometheus
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" ~/.xosd/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" ~/.xosd/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" ~/.xosd/config/app.toml
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "7000000.000000000000000000axos"|g' ~/.xosd/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" ~/.xosd/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" ~/.xosd/config/config.toml

# Systemd service
sudo tee /etc/systemd/system/xosd.service > /dev/null <<EOF
[Unit]
Description=XOSD Node
After=network-online.target

[Service]
User=$USER
WorkingDirectory=$HOME/.xosd
ExecStart=/usr/local/bin/xosd start --home /root/.xosd --chain-id=${XOSD_CHAIN_ID}
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable xosd
sudo systemctl restart xosd

# Log
sudo systemctl restart xosd && sudo journalctl -u xosd -fo cat

# Check status
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.xosd/config/config.toml" | cut -d ':' -f 3)
xosd status --node tcp://127.0.0.1:$rpc_port

# Check Block
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.xosd/config/config.toml" | cut -d ':' -f 3)
xosd status --node tcp://127.0.0.1:$rpc_port | jq '.sync_info.latest_block_height'

# Chaching
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.xosd/config/config.toml" | cut -d ':' -f 3)
xosd status --node tcp://127.0.0.1:$rpc_port | jq '.sync_info.catching_up'

# Add a new wallet
xosd keys add wallet --keyring-backend os

# Restore wallet from mnemonic phrase
xosd keys add wallet --recover --keyring-backend os

# List all wallets
xosd keys list --keyring-backend os

# Export private key (handle with care!)
xosd keys export wallet --keyring-backend os

# Show wallet info in validator bech32 format
xosd keys show wallet --bech val --keyring-backend os

# check balance
xosd query bank balances $(xosd keys show $WALLET -a --keyring-backend os)

# Display your validator public key
xosd tendermint show-validator


# create validator.json
cat <<EOF > /root/.xosd/validator.json
{
  "pubkey": {
    "@type": "/cosmos.crypto.ed25519.PubKey",
    "key": "$(xosd tendermint show-validator | grep -Po '\"key\":\s*\"\K[^"]*')"
  },
  "amount": "1000000axos",
  "moniker": "${MONIKER}",
  "identity": "",
  "website": "",
  "security": "",
  "details": "XOS Validator Node",
  "commission-rate": "0.10",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
EOF

# create validator
xosd tx staking create-validator /root/.xosd/validator.json \
  --from=wallet \
  --chain-id=$XOSD_CHAIN_ID \
  --gas=auto \
  --gas-prices=80000000000axos \
  --gas-adjustment=1.4 \
  --keyring-backend=os \
  --home ~/.xosd \
  -y

# delegate to own 
xosd tx staking delegate $(xosd keys show $WALLET --bech val -a --keyring-backend os) 1000000axos \
  --from $WALLET \
  --chain-id $XOSD_CHAIN_ID \
  --keyring-backend os \
  --gas=auto \
  --gas-adjustment=1.3 \
  --gas-prices=7000000axos \
  -y

# Query validator information
xosd query staking validator $(xosd keys show wallet --bech val --keyring-backend os)

# Check slashing (signing info)
xosd query slashing signing-info $(xosd tendermint show-validator) --chain-id xos_1267-1

# Unjail 
xosd tx slashing unjail \
  --from wallet \
  --chain-id xos_1267-1 \
  --keyring-backend os \
  --home ~/.xosd \
  -y
