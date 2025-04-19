**Manual Installation**

Official Documentation
```
Recommended Hardware: 4 Cores, 32GB RAM, 2TB NVMe
```

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.23.1"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**set vars**
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export BABYLON_CHAIN_ID="bbn-test-5"" >> $HOME/.bash_profile
echo "export BABYLON_PORT="60"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
rm -rf babylon
git clone https://github.com/babylonlabs-io/babylon.git
cd babylon
git checkout v1.0.0-rc.5
make install
```

**config and init app**
```
babylond init $MONIKER --chain-id $BABYLON_CHAIN_ID
sed -i \
-e "s/chain-id = .*/chain-id = \"$BABYLON_CHAIN_ID\"/" \
-e "s/keyring-backend = .*/keyring-backend = \"os\"/" \
-e "s/node = .*/node = \"tcp:\/\/localhost:${BABYLON_PORT}657\"/" $HOME/.babylond/config/client.toml
```

**download genesis and addrbook**
```
wget -O $HOME/.babylond/config/genesis.json https://server-7.itrocket.net/testnet/babylon/genesis.json
wget -O $HOME/.babylond/config/addrbook.json  https://server-7.itrocket.net/testnet/babylon/addrbook.json
```

**set seeds and peers**
```
SEEDS="0c949c3bcd83b81c794af8c3ae026a97d9c4564e@babylon-testnet-seed.itrocket.net:60656"
PEERS="041b2be170e9f5e3f951d1942030447ad1134ec4@babylon-testnet-peer.itrocket.net:60656,e643972e8eb888af2e2a94db13839e71b6d9a5ac@88.198.70.23:16156,0b50fa79f9965c4f02c3a2e2a2ec8d85f4478a22@195.201.197.160:26656,5e83cc27be007d8944b5424e2141bca51dd9121a@46.4.91.76:30556,61d43c6d4f131b6fa07692cd5d0c3bc8537f53f0@180.210.127.139:26656,b5ef770cac2aecd9f0a62bac7fbe262c1c55616b@51.89.40.26:26656,5121a3dc7d45860c6d87f4316b3e1387d9d7d7af@54.172.176.127:2121,c9eb49d2de9c30e27b7ae54deff234e356c97373@45.77.250.145:26656,514391ddf9e16a4f12d54bd3197d0ea045020b44@194.163.168.139:25656,dac672e0f4d02c5c43c0ef62500ac00f212c89b6@51.89.7.79:26656,63111b516fec1fb947ea0a38d05aab4eb82e454b@51.79.99.173:26656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.babylond/config/config.toml
```
**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${BABYLON_PORT}317%g;
s%:8080%:${BABYLON_PORT}080%g;
s%:9090%:${BABYLON_PORT}090%g;
s%:9091%:${BABYLON_PORT}091%g;
s%:8545%:${BABYLON_PORT}545%g;
s%:8546%:${BABYLON_PORT}546%g;
s%:6065%:${BABYLON_PORT}065%g" $HOME/.babylond/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${BABYLON_PORT}658%g;
s%:26657%:${BABYLON_PORT}657%g;
s%:6060%:${BABYLON_PORT}060%g;
s%:26656%:${BABYLON_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${BABYLON_PORT}656\"%;
s%:26660%:${BABYLON_PORT}660%g" $HOME/.babylond/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.babylond/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.babylond/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.babylond/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.002ubbn"|g' $HOME/.babylond/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.babylond/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.babylond/config/config.toml
```

**create service file**
```
sudo tee /etc/systemd/system/babylond.service > /dev/null <<EOF
[Unit]
Description=Babylon node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.babylond
ExecStart=$(which babylond) start --home $HOME/.babylond
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

**reset and download snapshot**
```
babylond tendermint unsafe-reset-all --home $HOME/.babylond
if curl -s --head curl https://server-7.itrocket.net/testnet/babylon/babylon_2025-03-20_547863_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-7.itrocket.net/testnet/babylon/babylon_2025-03-20_547863_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.babylond
    else
  echo "no snapshot found"
fi
```

**enable and start service**
```
sudo systemctl daemon-reload
sudo systemctl enable babylond
sudo systemctl restart babylond && sudo journalctl -u babylond -fo cat
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/babylon/autoinstall/)
```

Create wallet
**to create a new wallet, use the following command. don’t forget to save the mnemonic**
```
babylond keys add $WALLET
```

**to restore exexuting wallet, use the following command**
```
babylond keys add $WALLET --recover
```

**save wallet and validator address**
```
WALLET_ADDRESS=$(babylond keys show $WALLET -a)
VALOPER_ADDRESS=$(babylond keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**check sync status, once your node is fully synced, the output from above will print "false"**
```
babylond status 2>&1 | jq 
```

**before creating a validator, you need to fund your wallet and check balance**
```
babylond query bank balances $WALLET_ADDRESS
```

**Node Sync Status Checker**
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.babylond/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://babylon-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"
  sleep 5
done
```

**Create validator**
```
cd $HOME
```

# Create validator.json file
echo "{\"pubkey\":{\"@type\":\"/cosmos.crypto.ed25519.PubKey\",\"key\":\"$(babylond comet show-validator | grep -Po '\"key\":\s*\"\K[^"]*')\"},
    \"amount\": \"1000000ubbn\",
    \"moniker\": \"test\",
    \"identity\": \"\",
    \"website\": \"\",
    \"security\": \"\",
    \"details\": \"I love blockchain ❤️\",
    \"commission-rate\": \"0.1\",
    \"commission-max-rate\": \"0.2\",
    \"commission-max-change-rate\": \"0.01\",
    \"min-self-delegation\": \"1\"
}" > validator.json
# Create a validator using the JSON configuration
babylond tx staking create-validator validator.json \
    --from $WALLET \
    --chain-id bbn-test-5 \
	--gas auto --gas-adjustment 1.5
	
Monitoring
If you want to have set up a monitoring and alert system use our cosmos nodes monitoring guide with tenderduty

Security
To protect you keys please don`t share your privkey, mnemonic and follow basic security rules

Set up ssh keys for authentication
You can use this guide to configure ssh authentication and disable password authentication on your server

Firewall security
Set the default to allow outgoing connections, deny all incoming, allow ssh and node p2p port

sudo ufw default allow outgoing 
sudo ufw default deny incoming 
sudo ufw allow ssh/tcp 
sudo ufw allow ${BABYLON_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop babylond
sudo systemctl disable babylond
sudo rm -rf /etc/systemd/system/babylond.service
sudo rm $(which babylond)
sudo rm -rf $HOME/.babylond
sed -i "/BABYLON_/d" $HOME/.bash_profile
