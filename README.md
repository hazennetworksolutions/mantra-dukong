<div align="center">

# 🔴 MANTRA Dukong Testnet Full Node & Validator Setup Guide

**A complete guide to running a MANTRA Dukong testnet full node and registering as a validator**  
*System preparation, binary installation, Cosmovisor setup, and validator creation — step by step.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04+-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![MANTRA](https://img.shields.io/badge/MANTRA-Dukong%20Testnet-E63946?style=flat-square)](https://mantrachain.io)
[![Version](https://img.shields.io/badge/Node%20Version-v8.1.0-brightgreen?style=flat-square)](https://github.com/MANTRA-Chain/mantrachain/releases)
[![Chain ID](https://img.shields.io/badge/Chain%20ID-mantra--dukong--1-blue?style=flat-square)](https://docs.mantrachain.io)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Network:** MANTRA Dukong Testnet (Chain ID: mantra-dukong-1)  
> **Version:** v8.1.0  
> **Last Updated:** May 2026

---

## Table of Contents

- [Hardware Requirements](#hardware-requirements)
- [Network Endpoints](#network-endpoints)
- [Step 1 — System Verification](#step-1--system-verification)
- [Step 2 — System Update and Dependencies](#step-2--system-update-and-dependencies)
- [Step 3 — Install Go](#step-3--install-go)
- [Step 4 — Download Binary](#step-4--download-binary)
- [Step 5 — Install Cosmovisor](#step-5--install-cosmovisor)
- [Step 6 — Create Systemd Service](#step-6--create-systemd-service)
- [Step 7 — Initialize the Node](#step-7--initialize-the-node)
- [Step 8 — Download Genesis and Addrbook](#step-8--download-genesis-and-addrbook)
- [Step 9 — Configure Ports, Gas Prices and Pruning](#step-9--configure-ports-gas-prices-and-pruning)
- [Step 10 — Configure Seeds and Peers](#step-10--configure-seeds-and-peers)
- [Step 11 — Start the Node](#step-11--start-the-node)
- [Step 12 — Create a Wallet](#step-12--create-a-wallet)
- [Step 13 — Register as a Validator](#step-13--register-as-a-validator)
- [Monitoring the Node](#monitoring-the-node)
- [Useful Commands](#useful-commands)
- [Staying Updated](#staying-updated)

---

## Hardware Requirements

| Component | Minimum | Recommended |
|---|---|---|
| Operating System | Ubuntu 22.04+ | Ubuntu 24.04 |
| CPU | 4 cores | 8 cores |
| RAM | 16 GB | 32 GB |
| Disk | 500 GB NVMe SSD | 2 TB NVMe SSD |
| Network | 100 Mbps | 1 Gbps |

> ⚠️ Disk usage grows over time. Plan for long-term storage growth if running an archival node.

---

## Network Endpoints

| Type | Endpoint |
|---|---|
| RPC | https://rpc.dukong.mantrachain.io |
| REST API | https://api.dukong.mantrachain.io |
| gRPC | grpc.dukong.mantrachain.io:443 |
| EVM RPC | https://evm.dukong.mantrachain.io |
| Explorer | https://mantrascan.io/dukong |
| Faucet | https://faucet.dukong.mantrachain.io |
| EVM Chain ID | 5887 |

---

## Step 1 — System Verification

After SSH-ing into your server, verify the system meets requirements:

```bash
lsb_release -a          # Should be Ubuntu 22.04 or higher
uname -r                # Kernel version
lscpu | grep -E "Model name|CPU\(s\)|Thread|Socket|Core"
free -h                 # Minimum 16 GB RAM
df -h                   # Minimum 500 GB free disk
```

---

## Step 2 — System Update and Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git wget htop tmux build-essential jq make lz4 gcc unzip
```

---

## Step 3 — Install Go

MANTRA Chain v8.1.0 requires **Go 1.23+**:

```bash
cd $HOME
VER="1.23.0"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"

echo "export PATH=\$PATH:/usr/local/go/bin:\$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

Verify the installation:

```bash
go version
```

Expected output: `go version go1.23.0 linux/amd64`

---

## Step 4 — Download Binary

```bash
mkdir -p $HOME/.mantrachain/cosmovisor/genesis/bin

wget -O /tmp/mantrachaind.tar.gz \
  https://github.com/MANTRA-Chain/mantrachain/releases/download/v8.1.0/mantrachaind-8.1.0-linux-amd64.tar.gz

tar -xzf /tmp/mantrachaind.tar.gz -C $HOME/.mantrachain/cosmovisor/genesis/bin/
chmod +x $HOME/.mantrachain/cosmovisor/genesis/bin/mantrachaind
rm /tmp/mantrachaind.tar.gz

ln -s $HOME/.mantrachain/cosmovisor/genesis $HOME/.mantrachain/cosmovisor/current -f
sudo ln -s $HOME/.mantrachain/cosmovisor/current/bin/mantrachaind /usr/local/bin/mantrachaind -f
```

Verify:

```bash
mantrachaind version
```

Expected output: `v8.1.0`

> **Alternatively**, build from source:
> ```bash
> git clone https://github.com/MANTRA-Chain/mantrachain
> cd mantrachain && git checkout v8.1.0
> make install
> mkdir -p $HOME/.mantrachain/cosmovisor/genesis/bin
> cp $(which mantrachaind) $HOME/.mantrachain/cosmovisor/genesis/bin/
> ln -s $HOME/.mantrachain/cosmovisor/genesis $HOME/.mantrachain/cosmovisor/current -f
> sudo ln -s $HOME/.mantrachain/cosmovisor/current/bin/mantrachaind /usr/local/bin/mantrachaind -f
> ```

---

## Step 5 — Install Cosmovisor

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0
```

Verify:

```bash
cosmovisor version
```

---

## Step 6 — Create Systemd Service

Set your moniker and port prefix before creating the service:

```bash
MONIKER="YOUR_MONIKER"
PORT="26"   # Default is 26. Change to avoid conflicts (e.g. 27, 28...)
```

Create the service file:

```bash
sudo tee /etc/systemd/system/mantrachaind.service > /dev/null << EOF
[Unit]
Description=MANTRA Chain Node Service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.mantrachain"
Environment="DAEMON_NAME=mantrachaind"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.mantrachain/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable mantrachaind
```

---

## Step 7 — Initialize the Node

```bash
mantrachaind config set client chain-id mantra-dukong-1
mantrachaind config set client keyring-backend test
mantrachaind config set client node tcp://localhost:${PORT}657

mantrachaind init $MONIKER --chain-id mantra-dukong-1
```

Set environment variables permanently:

```bash
echo "export MONIKER=$MONIKER" >> $HOME/.bash_profile
echo "export MANTRA_CHAIN_ID=\"mantra-dukong-1\"" >> $HOME/.bash_profile
echo "export MANTRA_PORT=$PORT" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

---

## Step 8 — Download Genesis and Addrbook

```bash
wget -O $HOME/.mantrachain/config/genesis.json \
  https://raw.githubusercontent.com/hazennetworksolutions/mantra-dukong/refs/heads/main/genesis.json

wget -O $HOME/.mantrachain/config/addrbook.json \
  https://raw.githubusercontent.com/hazennetworksolutions/mantra-dukong/refs/heads/main/addrbook.json
```

Verify genesis checksum:

```bash
sha256sum $HOME/.mantrachain/config/genesis.json
```

---

## Step 9 — Configure Ports, Gas Prices and Pruning

### Custom Ports

If you changed the default port prefix, apply it here:

```bash
sed -i.bak -e "s%:1317%:${MANTRA_PORT}317%g;
s%:8080%:${MANTRA_PORT}080%g;
s%:9090%:${MANTRA_PORT}090%g;
s%:9091%:${MANTRA_PORT}091%g;
s%:8545%:${MANTRA_PORT}545%g;
s%:8546%:${MANTRA_PORT}546%g" $HOME/.mantrachain/config/app.toml

sed -i.bak -e "s%:26658%:${MANTRA_PORT}658%g;
s%:26657%:${MANTRA_PORT}657%g;
s%:6060%:${MANTRA_PORT}060%g;
s%:26656%:${MANTRA_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${MANTRA_PORT}656\"%;
s%:26660%:${MANTRA_PORT}660%g" $HOME/.mantrachain/config/config.toml
```

### Gas Prices

```bash
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.0002uom"|g' \
  $HOME/.mantrachain/config/app.toml
```

### Pruning

```bash
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.mantrachain/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.mantrachain/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.mantrachain/config/app.toml
```

### Enable Prometheus (optional)

```bash
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.mantrachain/config/config.toml
```

### Disable Indexer (saves disk space)

```bash
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.mantrachain/config/config.toml
```

---

## Step 10 — Configure Seeds and Peers

```bash
SEEDS="a9a71700397ce950a9396421877196ac19e7cde0@mantra-testnet-seed.itrocket.net:22656"
PEERS="1a46b1db53d1ff3dbec56ec93269f6a0d15faeb4@mantra-testnet-peer.itrocket.net:22656,e726816f42831689eab9378d5d577f1d06d25716@169.155.171.102:36656,9cde11013022782043c8cebad09504fff4c9f5ce@212.126.35.133:26667,da54ab1be9e88bdfa2845269ef24f975bb1c90c1@34.130.170.97:26656,0b9d7d7daefacc1903cecef392c6f10528839fa6@34.92.64.233:26656,593824b79d719e764832efca9e5137f29d1c49ca@88.218.226.175:26656,ae751ab62cffb7dda53a52e324d76cadf300ed9e@35.214.250.61:26656,1f402acf2e4ad59aca187e84c64ef973b2b3c8bb@35.220.173.152:26656,d93d460d030779b93cb2cf874f64af37a263e4e0@34.96.191.10:26656,0bfc4c0e18f09c50f26a7ac38b1541c3024edb87@34.96.208.200:26656,7e061edecef73a700b699c785f61a44ca981ff7f@34.150.103.79:26656"

sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" \
       $HOME/.mantrachain/config/config.toml
```

---

## Step 11 — Start the Node

```bash
sudo systemctl start mantrachaind
sudo journalctl -u mantrachaind -f --no-pager -o cat
```

Verify the service is running:

```bash
sudo systemctl status mantrachaind --no-pager
```

The service should show `active (running)`.

Check sync status:

```bash
mantrachaind status 2>&1 | jq .SyncInfo
```

Wait until `catching_up` is `false` before proceeding to validator registration.

---

## Step 12 — Create a Wallet

```bash
mantrachaind keys add wallet --keyring-backend test
```

> ⚠️ **CRITICAL:** Save your mnemonic phrase in a secure location. Without it, you cannot recover your wallet.

To recover an existing wallet:

```bash
mantrachaind keys add wallet --recover --keyring-backend test
```

Get testnet tokens from the faucet:

> 🚰 **Faucet:** [faucet.dukong.mantrachain.io](https://faucet.dukong.mantrachain.io)

Check your balance:

```bash
mantrachaind query bank balances $(mantrachaind keys show wallet -a --keyring-backend test)
```

---

## Step 13 — Register as a Validator

> The node must be **fully synced** before creating a validator.

### Get your pubkey:

```bash
mantrachaind comet show-validator
```

### Create validator JSON:

```bash
cat > $HOME/validator.json << EOF
{
  "pubkey": $(mantrachaind comet show-validator),
  "amount": "1000000uom",
  "moniker": "YOUR_MONIKER",
  "identity": "",
  "website": "",
  "security": "",
  "details": "",
  "commission-rate": "0.05",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
EOF
```

### Submit the transaction:

```bash
mantrachaind tx staking create-validator $HOME/validator.json \
  --from wallet \
  --chain-id mantra-dukong-1 \
  --keyring-backend test \
  --gas auto \
  --gas-adjustment 1.4 \
  --fees 500uom \
  -y
```

### Verify your validator:

```bash
mantrachaind query staking validator \
  $(mantrachaind keys show wallet --bech val -a --keyring-backend test)
```

---

## Monitoring the Node

### Watch live block commits:

```bash
sudo journalctl -u mantrachaind -f --no-pager | grep "committed block"
```

### Full logs:

```bash
sudo journalctl -u mantrachaind -f --no-pager
```

### Sync status:

```bash
mantrachaind status 2>&1 | jq .SyncInfo
```

### Service management:

```bash
# Restart service
sudo systemctl restart mantrachaind

# Stop service
sudo systemctl stop mantrachaind

# Check status
sudo systemctl status mantrachaind
```

---

## Useful Commands

### Wallet

```bash
# List wallets
mantrachaind keys list --keyring-backend test

# Show wallet address
mantrachaind keys show wallet -a --keyring-backend test

# Check balance
mantrachaind query bank balances $(mantrachaind keys show wallet -a --keyring-backend test)
```

### Staking

```bash
# Delegate tokens
mantrachaind tx staking delegate \
  $(mantrachaind keys show wallet --bech val -a --keyring-backend test) 1000000uom \
  --from wallet --chain-id mantra-dukong-1 --keyring-backend test \
  --gas auto --gas-adjustment 1.4 --fees 500uom -y

# Redelegate tokens
mantrachaind tx staking redelegate \
  $(mantrachaind keys show wallet --bech val -a --keyring-backend test) <NEW_VALOPER> 1000000uom \
  --from wallet --chain-id mantra-dukong-1 --keyring-backend test \
  --gas auto --gas-adjustment 1.4 --fees 500uom -y

# Undelegate tokens
mantrachaind tx staking unbond \
  $(mantrachaind keys show wallet --bech val -a --keyring-backend test) 1000000uom \
  --from wallet --chain-id mantra-dukong-1 --keyring-backend test \
  --gas auto --gas-adjustment 1.4 --fees 500uom -y
```

### Rewards

```bash
# Withdraw all rewards
mantrachaind tx distribution withdraw-all-rewards \
  --from wallet --chain-id mantra-dukong-1 --keyring-backend test \
  --gas auto --gas-adjustment 1.4 --fees 500uom -y

# Withdraw commission
mantrachaind tx distribution withdraw-rewards \
  $(mantrachaind keys show wallet --bech val -a --keyring-backend test) --commission \
  --from wallet --chain-id mantra-dukong-1 --keyring-backend test \
  --gas auto --gas-adjustment 1.4 --fees 500uom -y
```

### Governance

```bash
# List proposals
mantrachaind query gov proposals

# Vote on a proposal
mantrachaind tx gov vote 1 yes \
  --from wallet --chain-id mantra-dukong-1 --keyring-backend test \
  --gas auto --gas-adjustment 1.4 --fees 500uom -y
```

### Validator Operations

```bash
# Edit validator
mantrachaind tx staking edit-validator \
  --new-moniker "NEW_MONIKER" \
  --identity "" \
  --from wallet --chain-id mantra-dukong-1 --keyring-backend test \
  --gas auto --gas-adjustment 1.4 --fees 500uom -y

# Unjail validator
mantrachaind tx slashing unjail \
  --from wallet --chain-id mantra-dukong-1 --keyring-backend test \
  --gas auto --gas-adjustment 1.4 --fees 500uom -y

# Check validator signing info
mantrachaind query slashing signing-info $(mantrachaind comet show-validator)
```

---

## Staying Updated

Follow these channels to stay informed about upgrades and announcements:

- Discord: [MANTRA Chain Discord](https://discord.gg/mantrachain)
- GitHub Releases: [MANTRA-Chain/mantrachain Releases](https://github.com/MANTRA-Chain/mantrachain/releases)
- Official Docs: [docs.mantrachain.io](https://docs.mantrachain.io/operators/node-setup-and-deployment/validator-nodes/running-a-node)

### Upgrade with Cosmovisor (example: v9.0.0)

```bash
mkdir -p $HOME/.mantrachain/cosmovisor/upgrades/v9.0.0/bin

wget -O /tmp/mantrachaind.tar.gz \
  https://github.com/MANTRA-Chain/mantrachain/releases/download/v9.0.0/mantrachaind-9.0.0-linux-amd64.tar.gz

tar -xzf /tmp/mantrachaind.tar.gz -C $HOME/.mantrachain/cosmovisor/upgrades/v9.0.0/bin/
chmod +x $HOME/.mantrachain/cosmovisor/upgrades/v9.0.0/bin/mantrachaind
rm /tmp/mantrachaind.tar.gz
```

Cosmovisor will automatically switch to the new binary at the upgrade block height.

---

*Guide maintained by [HazenNetworkSolutions](https://hazennetworksolutions.com)*
