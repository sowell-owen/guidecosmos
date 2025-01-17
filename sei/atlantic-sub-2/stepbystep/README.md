# Step by step guide for Sei node atlantic-sub-2 chain

### Delete ur previous Sei node, if u use same server
```
cd $HOME
sudo systemctl stop seid && sudo systemctl disable seid
rm -rf $HOME/sei-chain $HOME/.sei $(which seid)
```
### Install node
#### Update APT
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
```
#### Install GO
```
rm -r /usr/local/go
rm -r /usr/lib/go-1.13
wget https://golang.org/dl/go1.18.3.linux-amd64.tar.gz; \
rm -rv /usr/local/go; \
tar -C /usr/local -xzf go1.18.3.linux-amd64.tar.gz && \
rm -v go1.18.3.linux-amd64.tar.gz && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile && \
source ~/.bash_profile && \
go version
```
#### Build binary
```
cd $HOME
git clone https://github.com/sei-protocol/sei-chain
cd $HOME/sei-chain
git checkout master && git pull
git checkout 1.1.2beta-internal
make install
mv $HOME/go/bin/seid /usr/bin/
```
#### Add variables (change "YOURNODENAME" and "YOURWALLETNAME" without "<>")
```
NODENAME=<YOURNODENAME>
WALLETNAME=<YOURWALLETNAME>
echo export NODENAME=${NODENAME} >> $HOME/.bash_profile
echo export WALLETNAME=${WALLETNAME} >> $HOME/.bash_profile
echo export CHAIN_ID=atlantic-sub-2 >> $HOME/.bash_profile
source ~/.bash_profile
```
#### make init
```
seid init $NODENAME --chain-id $CHAIN_ID
```
#### Add chain-id in config
```
seid config chain-id $CHAIN_ID
```
#### Create wallet
```
seid keys add $WALLETNAME
```
or restore previous wallet
```
seid keys add $WALLETNAME --recover
```
### Download genesis
```
seid tendermint unsafe-reset-all --home $HOME/.sei
rm $HOME/.sei/config/genesis.json
wget -O $HOME/.sei/config/genesis.json "https://raw.githubusercontent.com/sei-protocol/testnet/main/atlantic-subchains/atlantic-sub-2/genesis.json"
```
### Turn on pruning and turn off indexing
```
indexer="null"
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="10"

sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.sei/config/config.toml
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.sei/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.sei/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.sei/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.sei/config/app.toml
```
### Add peers and seeds
```
SEEDS=""
PEERS=""; \
sed -i.bak -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.sei/config/config.toml
```
### Create service
```
tee /etc/systemd/system/seid.service > /dev/null <<EOF
[Unit]
Description=SEI
After=network.target
[Service]
Type=simple
User=$USER
ExecStart=$(which seid) start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```
### Start service
```
sudo systemctl daemon-reload
sudo systemctl enable seid
sudo systemctl restart seid && journalctl -u seid -f -o cat
```
### Wait sync and create validator (if u is not in genesis file), backup $HOME/.sei/config/priv_validator_key.json file.
```
seid tx staking create-validator \
  --amount 1000000usei \
  --from $WALLETNAME \
  --commission-max-change-rate "0.05" \
  --commission-max-rate "0.20" \
  --commission-rate "0.05" \
  --min-self-delegation "1" \
  --pubkey $(seid tendermint show-validator) \
  --moniker $NODENAME \
  --chain-id $CHAIN_ID \
  --gas 300000 \
  -y
```
#### check sync status and block number
```
curl localhost:26657/status
```
#### check balance
```
seid q bank balances $(seid keys show $WALLETNAME -a)
```
