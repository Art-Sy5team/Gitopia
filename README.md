
## Tutorial Testnet Gitopia 

- [website](https://gitopia.com/) 
- [Discord](https://discord.gg/EPeQMx45kw) 
- [Explorer](https://gitopia.explorers.guru/)
- [docs ](https://docs.gitopia.com/installation/index.html)      


## Spesifikasi Minimum
| Komponen  | Requirements minimal|
|-----------|---------------------|
|CPU        | 8x Cores             |
|RAM        | 64GB                 |
|Penyimpanan| 1TB (SSD atau NVME)  |
|Koneksi    |10Mbps - 100Mbps      |

## Set up your gitopia fullnode
### Option 1 (otomatis)
install otomatis dan masukan username node!
```
wget -O gitopia.sh https://raw.githubusercontent.com/Art-Sy5team/Gitopia/main/gitopia.sh && chmod +x gitopia.sh && ./gitopia.sh
```

## Post installation

setelah install selesai gunakan perintah
```
source $HOME/.bash_profile
```

check synchronization block status
```
gitopiad status 2>&1 | jq .SyncInfo
```

### Buat wallet
untuk membuat wallet baru gunakan command dibawah. **jangan lupa simpan mnemonic**
```
gitopiad keys add $WALLET
```

(OPTIONAL) jika sudah punya wallet gunakan command dibawah dan masukan seed phrase anda
```
gitopiad keys add $WALLET --recover
```

check list wallet saat ini.
```
gitopiad keys list
```

### Save wallet
simpan address anda di valoper 
```
GITOPIA_WALLET_ADDRESS=$(gitopiad keys show $WALLET -a)
GITOPIA_VALOPER_ADDRESS=$(gitopiad keys show $WALLET --bech val -a)
echo 'export GITOPIA_WALLET_ADDRESS='${GITOPIA_WALLET_ADDRESS} >> $HOME/.bash_profile
echo 'export GITOPIA_VALOPER_ADDRESS='${GITOPIA_VALOPER_ADDRESS} >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### buat validator
Sebelum membuat validator node sudah sinkronkan & pastikan punnya minimal 1 tlore (1 tlore sama denga 1000000 utlore)

Untuk memeriksa saldo di wallet:
```
gitopiad query bank balances $GITOPIA_WALLET_ADDRESS
```
>  Jika dompet Anda tidak menunjukkan saldo apa pun, kemungkinan node Anda masih disinkronkan. Harap tunggu hingga selesai untuk menyinkronkan. 

Untuk membuat validator Anda, jalankan perintah di bawah ini
```
gitopiad tx staking create-validator \
  --amount 1000000utlore \
  --from $WALLET \
  --commission-max-change-rate "0.01" \
  --commission-max-rate "0.2" \
  --commission-rate "0.07" \
  --min-self-delegation "1" \
  --pubkey  $(gitopiad tendermint show-validator) \
  --moniker $NODENAME \
  --chain-id $GITOPIA_CHAIN_ID
```

### Keamana Firewall
Start by checking the status of ufw.
```
sudo ufw status
```

```
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh/tcp
sudo ufw limit ssh/tcp
sudo ufw allow ${GITOPIA_PORT}656,${GITOPIA_PORT}660/tcp
sudo ufw enable
```


## Calculate synchronization time
Skrip ini akan membantu Anda memperkirakan berapa lama waktu yang dibutuhkan untuk menyinkronkan node Anda sepenuhnya\ 104
Ini mengukur blok rata-rata per menit yang disinkronkan selama 5 menit dan kemudian memberi Anda hasil 105
```
wget -O synctime.py https://raw.githubusercontent.com/kj89/testnet_manuals/main/gitopia/tools/synctime.py && python3 ./synctime.py
```

### Check validator key 
```
[[ $(gitopiad q staking validator $GITOPIA_VALOPER_ADDRESS -oj | jq -r .consensus_pubkey.key) = $(gitopiad status | jq -r .ValidatorInfo.PubKey.value) ]] && echo -e "\n\e[1m\e[32mTrue\e[0m\n" || echo -e "\n\e[1m\e[31mFalse\e[0m\n"
```

### check list validators
```
gitopiad q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_BONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl
```

## Dapatkan daftar peer yang saat ini terhubung dengan id
```
curl -sS http://localhost:${GITOPIA_PORT}657/net_info | jq -r '.result.peers[] | "\(.node_info.id)@\(.remote_ip):\(.node_info.listen_addr)"' | awk -F ':' '{print $1":"$(NF)}'
```

## Usefull commands
### Service management
Check logs
```
journalctl -fu gitopiad -o cat
```

Start service
```
sudo systemctl start gitopiad
```

Stop service
```
sudo systemctl stop gitopiad
```

Restart service
```
sudo systemctl restart gitopiad
```

### Node info
Synchronization info
```
gitopiad status 2>&1 | jq .SyncInfo
```

Validator info
```
gitopiad status 2>&1 | jq .ValidatorInfo
```

Node info
```
gitopiad status 2>&1 | jq .NodeInfo
```

Show node id
```
gitopiad tendermint show-node-id
```

### Wallet operations
List of wallets
```
gitopiad keys list
```

Recover wallet
```
gitopiad keys add $WALLET --recover
```

Delete wallet
```
gitopiad keys delete $WALLET
```

cek saldo wallet
```
gitopiad query bank balances $GITOPIA_WALLET_ADDRESS
```

Transfer funds
```
gitopiad tx bank send $GITOPIA_WALLET_ADDRESS <TO_GITOPIA_WALLET_ADDRESS> 10000000utlore
```

### Voting
```
gitopiad tx gov vote 1 yes --from $WALLET --chain-id=$GITOPIA_CHAIN_ID
```

### Staking, Delegation dan Rewards
Delegate stake
```
gitopiad tx staking delegate $GITOPIA_VALOPER_ADDRESS 10000000utlore --from=$WALLET --chain-id=$GITOPIA_CHAIN_ID --gas=auto
```

Redelegate stake dari validator ke validator lainya
```
gitopiad tx staking redelegate <srcValidatorAddress> <destValidatorAddress> 10000000utlore --from=$WALLET --chain-id=$GITOPIA_CHAIN_ID --gas=auto
```

Withdraw semua rewards
```
gitopiad tx distribution withdraw-all-rewards --from=$WALLET --chain-id=$GITOPIA_CHAIN_ID --gas=auto
```

Withdraw rewards dengan Komisi
```
gitopiad tx distribution withdraw-rewards $GITOPIA_VALOPER_ADDRESS --from=$WALLET --commission --chain-id=$GITOPIA_CHAIN_ID
```

### Validator management
Edit validator
```
gitopiad tx staking edit-validator \
  --moniker=$NODENAME \
  --identity=<your_keybase_id> \
  --website="<your_website>" \
  --details="<your_validator_description>" \
  --chain-id=$GITOPIA_CHAIN_ID \
  --from=$WALLET
```

Unjail validator
```
gitopiad tx slashing unjail \
  --broadcast-mode=block \
  --from=$WALLET \
  --chain-id=$GITOPIA_CHAIN_ID \
  --gas=auto
```

### menghapus node
This commands will completely remove node from server. Use at your own risk!
```
sudo systemctl stop gitopiad
sudo systemctl disable gitopiad
sudo rm /etc/systemd/system/gitopia* -rf
sudo rm $(which gitopiad) -rf
sudo rm $HOME/.gitopia* -rf
sudo rm $HOME/gitopia -rf
sed -i '/GITOPIA_/d' ~/.bash_profile
```

### Art-Team INFO
noted: ***art team*** here does not have any specific goals or intentions, they only collect data and share it with everyone.

untuk INFO Testnet lainya Silahkan join Discord 👇

[![twitter](https://img.shields.io/badge/twitter-1DA1F2?style=for-the-badge&logo=twitter&logoColor=white)](https://twitter.com/ArtSy5team)
[![Discord](https://img.shields.io/badge/discord-7289d9?style=for-the-badge&logo=discord&logoColor=white)](https://discord.gg/EAKEdZU6c8)
[![Github](https://img.shields.io/badge/GitHub-171515?style=for-the-badge&logo=GitHub&logoColor=white)](https://github.com/Art-Sy5team)
[![trakteer](https://img.shields.io/badge/trakteer.id-e31e1e?style=for-the-badge&logo=ko-fi&logoColor=white)](https://trakteer.id/Art-Sy5team/tip)

