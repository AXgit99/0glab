Manual Installation
Official Documentation
Recommended Hardware: 8 Cores, 64GB RAM, 1 TB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.21.3"
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
echo "export OG_CHAIN_ID="zgtendermint_16600-2"" >> $HOME/.bash_profile
echo "export OG_PORT="47"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
rm -rf 0g-chain
git clone -b v0.2.5 https://github.com/0glabs/0g-chain.git
cd 0g-chain
make install
```

**config and init app**
```
0gchaind config node tcp://localhost:${OG_PORT}657
0gchaind config keyring-backend os
0gchaind config chain-id zgtendermint_16600-2
0gchaind init "test" --chain-id zgtendermint_16600-2
```

**download genesis and addrbook**
```
wget -O $HOME/.0gchain/config/genesis.json https://server-5.itrocket.net/testnet/og/genesis.json
wget -O $HOME/.0gchain/config/addrbook.json  https://server-5.itrocket.net/testnet/og/addrbook.json
```

**set seeds and peers**
```
SEEDS="8f21742ea5487da6e0697ba7d7b36961d3599567@og-testnet-seed.itrocket.net:47656"
PEERS="80fa309afab4a35323018ac70a40a446d3ae9caf@og-testnet-peer.itrocket.net:11656,103cc8035bb265a1aad865ec81dca69897646a01@38.242.215.44:12656,d8cbe10f1a82952a328e6c5be1251380225810b1@157.173.109.56:26656,0ba76ede1cde81cd242eb7cc7c3630c15265e4e8@217.76.49.214:12656,703bb272138ad988e34c63bc87de5cd36d28aeec@185.185.80.39:12656,e764eb09c843d5ef6fcfce8d5894a93cc55da14e@109.199.101.174:12656,b94cd15ff22b41e8965755bb45b87e64aeb6c7e3@144.91.80.34:12656,0f33e8d20b46e7e3aeb13f514a44aaa9735346d1@161.97.115.147:12656,025cc5990c4e92b172e3f54e74aad2734496490f@109.199.115.26:12656,7536c8a546919fba6743fec8da5e5dd281351ac3@116.203.88.24:26656,f0cab502a3749047738df99cc0b79e8faed604c8@65.108.123.126:12656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.0gchain/config/config.toml
```


**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${OG_PORT}317%g;
s%:8080%:${OG_PORT}080%g;
s%:9090%:${OG_PORT}090%g;
s%:9091%:${OG_PORT}091%g;
s%:8545%:${OG_PORT}545%g;
s%:8546%:${OG_PORT}546%g;
s%:6065%:${OG_PORT}065%g" $HOME/.0gchain/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${OG_PORT}658%g;
s%:26657%:${OG_PORT}657%g;
s%:6060%:${OG_PORT}060%g;
s%:26656%:${OG_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${OG_PORT}656\"%;
s%:26660%:${OG_PORT}660%g" $HOME/.0gchain/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.0gchain/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.0gchain/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.0gchain/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0ua0gi"|g' $HOME/.0gchain/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.0gchain/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.0gchain/config/config.toml
```

**create service file**
```
sudo tee /etc/systemd/system/0gchaind.service > /dev/null <<EOF
[Unit]
Description=0G node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.0gchain
ExecStart=$(which 0gchaind) start --home $HOME/.0gchain --log_output_console
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

**reset and download snapshot**
```
0gchaind tendermint unsafe-reset-all --home $HOME/.0gchain
if curl -s --head curl https://server-5.itrocket.net/testnet/og/og_2024-10-01_1283922_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-5.itrocket.net/testnet/og/og_2024-10-01_1283922_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.0gchain
    else
  echo "no snapshot founded"
fi
```

# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable 0gchaind
sudo systemctl restart 0gchaind && sudo journalctl -u 0gchaind -f
Automatic Installation
pruning: custom: 100/0/50 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/og/autoinstall/)
Create wallet
# to create a new wallet, use the following command. don’t forget to save the mnemonic
0gchaind keys add $WALLET

# to restore exexuting wallet, use the following command
0gchaind keys add $WALLET --recover

# save wallet and validator address
WALLET_ADDRESS=$(0gchaind keys show $WALLET -a)
VALOPER_ADDRESS=$(0gchaind keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile

# check sync status, once your node is fully synced, the output from above will print "false"
0gchaind status 2>&1 | jq 

# before creating a validator, you need to fund your wallet and check balance
0gchaind query bank balances $WALLET_ADDRESS 
Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.0gchain/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://og-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  if [ "$blocks_left" -lt 0 ]; then
    blocks_left=0
  fi

  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"

  sleep 5
done
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, ua0gi
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
0gchaind tx staking create-validator \
--amount 1000000ua0gi \
--from $WALLET \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(0gchaind tendermint show-validator) \
--moniker "test" \
--identity "" \
--website "" \
--details "I love blockchain ❤️" \
--chain-id zgtendermint_16600-2 \
--gas=auto --gas-adjustment=1.6 \
-y
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
sudo ufw allow ${OG_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop 0gchaind
sudo systemctl disable 0gchaind
sudo rm -rf /etc/systemd/system/0gchaind.service
sudo rm $(which 0gchaind)
sudo rm -rf $HOME/.0gchain
sed -i "/OG_/d" $HOME/.bash_profile
