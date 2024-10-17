![image](https://github.com/user-attachments/assets/b1662ef1-0c15-48fb-9210-bb1361a9084f)

# Setup-Node-Story-Iliad network
Story Blockchain Validator Node Setup Guide for test net - Iliad network
# Installation
## System Specs
Hardware	Requirement
CPU	4 Cores
RAM	8 GB
Disk	200 GB
Bandwidth	10 MBit/s
https://docs.story.foundation/docs/node-setup

## Install dependencies
```
sudo apt update
sudo apt-get update
sudo apt install curl git make jq build-essential gcc unzip wget lz4 aria2 -y
```
## Download Story-Geth binary v0.9.4
```
cd $HOME
wget https://github.com/piplabs/story-geth/releases/download/v0.9.4/geth-linux-amd64
[ ! -d "$HOME/go/bin" ] && mkdir -p $HOME/go/bin
if ! grep -q "$HOME/go/bin" $HOME/.bash_profile; then
  echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
fi
chmod +x geth-linux-amd64
mv $HOME/geth-linux-amd64 $HOME/go/bin/story-geth
source $HOME/.bash_profile
story-geth version
```
## Download Story binary v0.11.0
```
cd $HOME
wget https://story-geth-binaries.s3.us-west-1.amazonaws.com/story-public/story-linux-amd64-0.11.0-aac4bfe.tar.gz
tar -xzvf story-linux-amd64-0.11.0-aac4bfe.tar.gz
[ ! -d "$HOME/go/bin" ] && mkdir -p $HOME/go/bin
if ! grep -q "$HOME/go/bin" $HOME/.bash_profile; then
  echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
fi
sudo cp $HOME/story-linux-amd64-0.11.0-aac4bfe/story $HOME/go/bin
source $HOME/.bash_profile
story version
```
## Init Iliad node
```
story init --network iliad --moniker "Your_moniker_name"
```
## Create story-geth service file
```
sudo tee /etc/systemd/system/story-geth.service > /dev/null <<EOF
[Unit]
Description=Story Geth Client
After=network.target

[Service]
User=root
ExecStart=/root/go/bin/story-geth --iliad --syncmode full
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
```
## Create story service file
```
sudo tee /etc/systemd/system/story.service > /dev/null <<EOF
[Unit]
Description=Story Consensus Client
After=network.target

[Service]
User=root
ExecStart=/root/go/bin/story run
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
```
# Use snapshot
### Install tool
```
sudo apt-get install wget lz4 aria2 pv -y
```
### Stop node
```
sudo systemctl stop story
sudo systemctl stop story-geth
```
### Download snapshot (Prune snapshot:  1473790 )
- Story data: 94Gb
```
cd $HOME
rm -f Story_snapshot.lz4
aria2c -x 16 -s 16 -k 1M https://story.josephtran.co/Story_snapshot.lz4
```
- Geth data: 36Gb
```
cd $HOME
rm -f Geth_snapshot.lz4
aria2c -x 16 -s 16 -k 1M https://story.josephtran.co/Geth_snapshot.lz4
```
### Backup ```priv_validator_state.json```
```
cp ~/.story/story/data/priv_validator_state.json ~/.story/priv_validator_state.json.backup
```
### Remove old data
```
rm -rf ~/.story/story/data
rm -rf ~/.story/geth/iliad/geth/chaindata
```
### Decompress snapshot
```
sudo mkdir -p /root/.story/story/data
lz4 -d -c Story_snapshot.lz4 | pv | sudo tar xv -C ~/.story/story/ > /dev/null
```
```
sudo mkdir -p /root/.story/geth/iliad/geth/chaindata
lz4 -d -c Geth_snapshot.lz4 | pv | sudo tar xv -C ~/.story/geth/iliad/geth/ > /dev/null
```
### Move ```priv_validator_state.json``` back
```
cp ~/.story/priv_validator_state.json.backup ~/.story/story/data/priv_validator_state.json
```
### Restart node
```
sudo systemctl start story
sudo systemctl start story-geth
```
# Live peers
```
PEERS=$(curl -s -X POST https://rpc-story.josephtran.xyz -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","method":"net_info","params":[],"id":1}' | jq -r '.result.peers[] | select(.connection_status.SendMonitor.Active == true) | "\(.node_info.id)@\(if .node_info.listen_addr | contains("0.0.0.0") then .remote_ip + ":" + (.node_info.listen_addr | sub("tcp://0.0.0.0:"; "")) else .node_info.listen_addr | sub("tcp://"; "") end)"' | tr '\n' ',' | sed 's/,$//' | awk '{print "\"" $0 "\""}')
sed -i "s/^persistent_peers *=.*/persistent_peers = $PEERS/" "$HOME/.story/story/config/config.toml"
    if [ $? -eq 0 ]; then
echo -e "Configuration file updated successfully with new peers"
    else
echo "Failed to update configuration file."
fi
```

## Reload and start story-geth
```
sudo systemctl daemon-reload && \
sudo systemctl start story-geth && \
sudo systemctl enable story-geth && \
sudo systemctl status story-geth
```
## Reload and start story
```
sudo systemctl daemon-reload && \
sudo systemctl start story && \
sudo systemctl enable story && \
sudo systemctl status story
```
## Check logs
```
sudo journalctl -u story-geth -f -o cat
```
Wait a minute for connect peers

```
sudo journalctl -u story -f -o cat
```
## Check sync status
```
curl localhost:26657/status | jq
```
## Check block sync left:
```
while true; do
    local_height=$(curl -s localhost:26657/status | jq -r '.result.sync_info.latest_block_height');
    network_height=$(curl -s https://rpc-story.josephtran.xyz/status | jq -r '.result.sync_info.latest_block_height');
    blocks_left=$((network_height - local_height));
    echo -e "\033[1;38mYour node height:\033[0m \033[1;34m$local_height\033[0m | \033[1;35mNetwork height:\033[0m \033[1;36m$network_height\033[0m | \033[1;29mBlocks left:\033[0m \033[1;31m$blocks_left\033[0m";
    sleep 5;
done
```
# Create validator
```
story validator export
```
In addition, if you want to export the derived EVM private key of your validator into the default data config directory, please run the following:
```
story validator export --export-evm-key
```
Note that to participate in consensus, at least 1 IP must be staked (equivalent to 1000000000000000000 wei)!
Faucet link: https://faucet.story.foundation/
# Create validator
```
story validator create --stake 1000000000000000000 --private-key "your_private_key"
```
- Caution: Backup the validator key:
File location: /root/.story/story/config/priv_validator_key.json
Copy this file to your local machine. Store it carefully; this is the most crucial key for your validator.

## Validator Staking
```
story validator stake \
   --validator-pubkey "VALIDATOR_PUB_KEY_IN_BASE64" \
   --stake 1000000000000000000 \
   --private-key xxxxxxxxxxxxxx
```
- Replace VALIDATOR_PUB_KEY_IN_BASE64 Amount: 1000000000000000000=1 IP Token
### Check your Validator on Explorer
```
curl -s localhost:26657/status | jq -r '.result.validator_info'
```
# Delete node
```
sudo systemctl stop story-geth
sudo systemctl stop story
sudo systemctl disable story-geth
sudo systemctl disable story
sudo rm /etc/systemd/system/story-geth.service
sudo rm /etc/systemd/system/story.service
sudo systemctl daemon-reload
sudo rm -rf $HOME/.story
sudo rm $HOME/go/bin/story-geth
sudo rm $HOME/go/bin/story
```
- Caution: Backup your data, private key, validator key before remove node

































