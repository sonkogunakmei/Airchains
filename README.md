**Manual Installation**

Official Documentation
```
Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)
```

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.21.6"
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
echo "export AIRCHAIN_CHAIN_ID="junction"" >> $HOME/.bash_profile
echo "export AIRCHAIN_PORT="19"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
wget -O junctiond https://github.com/airchains-network/junction/releases/download/v0.2.0/junctiond-linux-amd64
chmod +x junctiond
mv junctiond $HOME/go/bin/
```

**config and init app**
```
junctiond init $MONIKER --chain-id $AIRCHAIN_CHAIN_ID 
sed -i -e "s|^node *=.*|node = \"tcp://localhost:${AIRCHAIN_PORT}657\"|" $HOME/.junction/config/client.toml
```

# download genesis and addrbook
wget -O $HOME/.junction/config/genesis.json https://server-4.itrocket.net/testnet/airchains/genesis.json
wget -O $HOME/.junction/config/addrbook.json  https://server-4.itrocket.net/testnet/airchains/addrbook.json

# set seeds and peers
SEEDS="04e2fdd6ec8f23729f24245171eaceae5219aa91@airchains-testnet-seed.itrocket.net:19656"
PEERS="e929f77cfe4cd7d51433f438d3b764937e799313@airchains-testnet-peer.itrocket.net:19656,976a0fe0a0fa205478beb66addaae3842907c3f6@37.27.48.77:32656,5880ddf4518b061c111ae6bf07b1ef76ef2a42af@158.220.100.154:26656,a9ee14aac08cd57715086fd6759371cc434b4bcd@[2a01:4f8:171:325::2]:26656,8b2a63f074a37bbfebd82cb78a4893936e1dfd61@37.27.132.57:19656,84230c0e2f9a1e0dbd96dea52b9b90209be0478b@65.109.92.163:1020,859485b13c2d8ab3888ffc11d1c506d78f681317@5.9.116.21:26756,0305205b9c2c76557381ed71ac23244558a51099@162.55.65.162:26656,d0cbeeeb6d16d82aa36e0f3936efc0f0918f8956@51.91.80.192:26656,aaf57c42eb1a53b487443088db025a5d05f78159@[2a01:4f9:3051:19c2::2]:13756,5717aadf2f21e223012a1ab27e21307f510037c3@[2a01:4f8:262:121c::2]:13756"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.junction/config/config.toml

# set custom ports in app.toml
sed -i.bak -e "s%:1317%:${AIRCHAIN_PORT}317%g;
s%:8080%:${AIRCHAIN_PORT}080%g;
s%:9090%:${AIRCHAIN_PORT}090%g;
s%:9091%:${AIRCHAIN_PORT}091%g;
s%:8545%:${AIRCHAIN_PORT}545%g;
s%:8546%:${AIRCHAIN_PORT}546%g;
s%:6065%:${AIRCHAIN_PORT}065%g" $HOME/.junction/config/app.toml

# set custom ports in config.toml file
sed -i.bak -e "s%:26658%:${AIRCHAIN_PORT}658%g;
s%:26657%:${AIRCHAIN_PORT}657%g;
s%:6060%:${AIRCHAIN_PORT}060%g;
s%:26656%:${AIRCHAIN_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${AIRCHAIN_PORT}656\"%;
s%:26660%:${AIRCHAIN_PORT}660%g" $HOME/.junction/config/config.toml

# config pruning
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.junction/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.junction/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.junction/config/app.toml

# set minimum gas price, enable prometheus and disable indexing
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.001amf"|g' $HOME/.junction/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.junction/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.junction/config/config.toml

# create service file
sudo tee /etc/systemd/system/junctiond.service > /dev/null <<EOF
[Unit]
Description=Airchains node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.junction
ExecStart=$(which junctiond) start --home $HOME/.junction
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

# reset and download snapshot
junctiond tendermint unsafe-reset-all --home $HOME/.junction
if curl -s --head curl https://server-4.itrocket.net/testnet/airchains/airchains_2025-02-07_4036307_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-4.itrocket.net/testnet/airchains/airchains_2025-02-07_4036307_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.junction
    else
  echo "no snapshot found"
fi

# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable junctiond
sudo systemctl restart junctiond && sudo journalctl -u junctiond -fo cat
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/airchains/autoinstall/)
Create wallet
# to create a new wallet, use the following command. don’t forget to save the mnemonic
junctiond keys add $WALLET

# to restore exexuting wallet, use the following command
junctiond keys add $WALLET --recover

# save wallet and validator address
WALLET_ADDRESS=$(junctiond keys show $WALLET -a)
VALOPER_ADDRESS=$(junctiond keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile

# check sync status, once your node is fully synced, the output from above will print "false"
junctiond status 2>&1 | jq 

# before creating a validator, you need to fund your wallet and check balance
junctiond query bank balances $WALLET_ADDRESS 
Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.junction/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://airchains-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"
  sleep 5
done
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, amf
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
cd $HOME
# Create validator.json file
echo "{\"pubkey\":{\"@type\":\"/cosmos.crypto.ed25519.PubKey\",\"key\":\"$(junctiond comet show-validator | grep -Po '\"key\":\s*\"\K[^"]*')\"},
    \"amount\": \"1000000amf\",
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
junctiond tx staking create-validator validator.json \
    --from $WALLET \
    --chain-id junction \
	--fees 200amf
	
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
sudo ufw allow ${AIRCHAIN_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop junctiond
sudo systemctl disable junctiond
sudo rm -rf /etc/systemd/system/junctiond.service
sudo rm $(which junctiond)
sudo rm -rf $HOME/.junction
sed -i "/AIRCHAIN_/d" $HOME/.bash_profile
