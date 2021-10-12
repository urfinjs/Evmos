apt update && apt upgrade -y

# setup variables
echo 'export EVMOS_NODENAME=`put_here_your_validator_moniker`' >> $HOME/.bash_profile && source $HOME/.bash_profile && echo $EVMOS_NODENAME

echo 'export EVMOS_WALLET=`put_here_your_prefered_wallet_name`' >> $HOME/.bash_profile && source $HOME/.bash_profile && echo $EVMOS_WALLET

echo 'export EVMOS_CHAIN=evmos_9000-1' >> $HOME/.bash_profile && source $HOME/.bash_profile && echo $EVMOS_CHAIN
###


# check ports: output should be empty
ss -tulpen | grep '26656\|26657\|26658\|26660\|6060'
###


# install dependencies
apt install make clang pkg-config libssl-dev build-essential git jq ncdu -y < "/dev/null"
###


# install go
target_go_binary="go1.17.2.linux-amd64.tar.gz"

wget -O $target_go_binary https://golang.org/dl/$target_go_binary

rm -rf /usr/local/go && tar -C /usr/local -xzf $target_go_binary && rm $target_go_binary

echo 'export GOROOT=/usr/local/go' >> $HOME/.bash_profile \
echo 'export GOPATH=$HOME/go' >> $HOME/.bash_profile \
echo 'export GO111MODULE=on' >> $HOME/.bash_profile \
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile && . $HOME/.bash_profile

go version
###


# install evmos
git clone https://github.com/tharsis/evmos.git && cd evmos && make install

evmosd version
###


# init node
evmosd init ${EVMOS_NODENAME} --chain-id=${EVMOS_CHAIN}

wget -O $HOME/.evmosd/config/genesis.json "https://raw.githubusercontent.com/tharsis/testnets/main/arsia_mons/genesis.json"

### command below should approve your genesis file
evmosd validate-genesis

### edit config files
sed -i '/\[grpc\]/{:a;n;/enabled/s/false/true/;Ta};/\[api\]/{:a;n;/enable/s/false/true/;Ta;}' $HOME/.evmosd/config/app.toml

external_address=\`curl -s ifconfig.me\` && echo $external_address \
seeds="c36cec90ded95d162b85f8ecd00ecd7c8849ca75@arsiamons.seed.evmos.org:26656" && echo $peers \
sed -i.bak -e "s/^external_address = \"\"/external_address = \"$external_address:26656\"/; s/^seeds *=.*/seeds = \"$seeds\"/" $HOME/.evmosd/config/config.toml
###


# create a wallet
source $HOME/.bash_profile \
evmosd keys add $EVMOS_WALLET
### import key below to the metamask
evmosd keys unsafe-export-eth-key $EVMOS_WALLET
###


# setup a service file and create your validator
```
echo "[Unit]
Description=Evmos Node
After=network-online.target
[Service]
User=$USER
ExecStart=$(which evmosd) start
Restart=always
RestartSec=3
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target" > /etc/systemd/system/evmosdd.service && cat /etc/systemd/system/evmosdd.service
```

systemctl daemon-reload && systemctl enable evmosdd && systemctl restart evmosdd && journalctl -u evmosdd -f

```
evmosd tx staking create-validator \
  --amount=1000000000000aphoton \
  --pubkey=$(evmosd tendermint show-validator) \
  --moniker=$EVMOS_NODENAME \
  --chain-id=$EVMOS_CHAIN \
  --commission-rate="0.10" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1000000" \
  --gas=200000 \
  --gas-prices="0.025aphoton" \
  --from=$EVMOS_WALLET
  ```

## check tx from validator creating above
evmosd q tx {tx hash from above}
###


# commands
### check your current block height and sync status in a loop
while true; do evmosd status 2>&1 | jq -r '(.SyncInfo.latest_block_height|tostring) + " catching up: " + (.SyncInfo.catching_up|tostring)'; sleep 30; done

### below same as: evmosd q staking validator $(evmosd keys show $EVMOS_WALLET --bech val -a) -o json | jq
evmosd q staking validator `your_evmosvaloper` -o json | jq

### below same as: curl -s localhost:26657/status | jq
evmosd status 2>&1 | jq

### check current validators
evmosd q staking validators -o json | jq -r '.validators[] | .description.moniker' | sort

### hodl some tokens for fees
evmosd q bank balances `your_wallet_adddress_not_name`

### delegate more tokens
evmosd tx staking delegate `evmos_valoper_to_delegate` 900000000000000000aphoton --from=${EVMOS_WALLET} --gas=200000 --chain-id=${EVMOS_CHAIN} -y
