# Stratos Tropos-4 Kurulum Rehberi 
![image](https://user-images.githubusercontent.com/102043225/181925323-87e8650a-b88e-4d04-adec-a294e37a69f0.png)

##  Sistem Gereksinimleri
* 4vCPU
* 8GB RAM
* 200GB SSD

## Root Yetkisi Alma Ve Root Dizinine Geçme
```shell
sudo su
cd /root
```

## Sistemi Güncelleme
```shell
apt update && apt upgrade -y
```

## Gerekli Kütüphanelerin Kurulması
```shell
apt install make clang pkg-config libssl-dev build-essential git jq ncdu bsdmainutils htop screen -y < "/dev/null"
```

## Değişkenleri Yükleme
aşağıda değiştirmeniz gereken yerleri yazıyorum.
* $NODENAME validator adınız
* $WALLET cüzdan adınız
```shell
echo "export NODENAME=$NODENAME"  >> $HOME/.bash_profile
echo "export WALLET=$WALLET" >> $HOME/.bash_profile
echo "export CHAIN_ID=tropos-4" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

Yani düzenlediğinizde aşağıdaki gibi olacak.
```shell
echo "export NODENAME=Bilge"  >> $HOME/.bash_profile
echo "export WALLET=Bilge" >> $HOME/.bash_profile
echo "export CHAIN_ID=tropos-4" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

## stcahind Binary Dosyalarını Yükleme
```shell
cd $HOME
wget https://github.com/stratosnet/stratos-chain/releases/download/v0.8.0/stchaind
```

## stcahind Ayrıntılarını Kontrol Etme
```shell
md5sum stchain*
```

Aşağıdaki gibi bir çıktı aldıysanız sorun yoktur.

```shell
834c713f15752e9f68489a43bac6a180 stchaind
```

## İndirilen Binary Dosyasına Yürütme İzni Verme
```shell
chmod +x stchaind
```

## Go Kurulumu
```shell
ver="1.18.4"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
rm -rf /usr/local/go
tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm -rf "go$ver.linux-amd64.tar.gz"
echo 'export GOROOT=/usr/local/go' >> $HOME/.bash_profile
echo 'export GOPATH=$HOME/go' >> $HOME/.bash_profile
echo 'export GO111MODULE=on' >> $HOME/.bash_profile
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
go mod tidy
```

## Stratos Cahin Kurulumu
```shell
cd $HOME
git clone https://github.com/stratosnet/stratos-chain.git
cd stratos-chain
git checkout v0.8.0
make build
```

## make build Kodundan Sonra Hata Alırsanız
```shell
go mod tidy
apt update
make build
```

## Binary Dosyalarını $GOPATH/bin Dizinine Yükleme
```shell
mv build/stchaind ./
make install
```

## Uygulamayı Başlatma
```shell
./stchaind init $NODENAME
```

## Genesis, Config ve Addrbook Dosyalarının İndirilmesi
```shell
curl https://raw.githubusercontent.com/stratosnet/stratos-chain-testnet/main/genesis.json > ~/.stchaind/config/genesis.json
curl https://raw.githubusercontent.com/stratosnet/stratos-chain-testnet/main/config.toml > ~/.stchaind/config/config.toml
curl https://github.com/mmc6185/node-testnets/blob/main/stratos/stratos-tropos-4/addrbook.json > ~/.stchaind/config/addrbook.json
```

## Servis Dosyası Oluşturma
```shell
tee <<EOF >/dev/null /etc/systemd/system/stratosd.service
[Unit]
Description=Stratos Node
After=network.target
[Service]
User=$USER
Type=simple
ExecStart=$(which stchaind) start
Restart=on-failure
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

## Journald Dosyası Oluşturma
```shell
tee <<EOF >/dev/null /etc/systemd/journald.conf
Storage=persistent
EOF
```

## Servisi Başlatma
```shell
systemctl restart systemd-journald
systemctl daemon-reload
systemctl enable stratosd
systemctl restart stratosd
```

## Logları Kontrol Etme
```shell
journalctl -u stratosd -f -o cat
```  

## Cüzdan Oluşturma

### Yeni Cüzdan Oluşturma
`$WALLET` bölümünü değiştirmiyoruz kurulumun başında cüzdanımıza isim belirledik.
```shell 
stchaind keys add --hd-path "m/44'/606'/0'/0/0" --keyring-backend test $WALLET
```  

### Var Olan Cüzdanı İçeri Aktarma
```shell
stchaind keys add --hd-path "m/44'/606'/0'/0/0" --keyring-backend test $WALLET --recover
```

* BU AŞAMADAN SONRA NODE'UMUZUN EŞLEŞMESİNİ BEKLİYORUZ.
* EŞLEŞMEYİ BEKLERKEN TOKEN ALALIM

## Faucet / Token Alma
```shell
curl --header "Content-Type: application/json" --request POST --data '{"denom":"ustos","address":"CUZDAN_ADRESINIZ"} ' https://faucet-tropos.thestratos.org/credit
```


## Cüzdan Bakiyesini Kontrol Etme
```shell
stchaind query bank balances CUZDAN_ADRESINIZ 
```  

## Senkronizasyonu Kontrol Etme
`false` çıktısı almaldıkça bir sonraki yani validator oluşturma adımına geçmiyoruz.
```shell
stchaind status 2>&1 | jq .SyncInfo
```

## Validator Oluşturma
 Aşağıdaki komutta aşağıda berlittiğim yerler dışında bir değişikli yapmanız gerekmez;
   'identity'  buraya `httpskeybase.io` sitesine üye olarak size verilen kimlik numaranızı yazıyorsunuz.
   'details'  kendiniz hakkında bilgiler verebilir ya da `Rues Community Supporter` yazabilirsiniz.
   'website'  Varsa bir siteniz yazabilirsiniz ya da `httpsforum.rues.info` olarak bırakabilirsiniz.
   'security-contact'  E-posta adresiniz.
```shell 
stchaind tx staking create-validator \
 --commission-max-change-rate=0.01 \
 --commission-max-rate=0.20 \
 --commission-rate=0.10 \
 --amount 14500000000ustos \
 --pubkey=$(stchaind tendermint show-validator) \
 --moniker=$NODENAME \
 --chain-id=$CHAIN_ID \
 --details=Rues Community Supporter \
 --security-contact=E-POSTANIZ \
 --website=httpsforum.rues.info \
 --identity=XXXX1111XXXX1111 \
 --min-self-delegation=1 \
 --from=$WALLET \
 --gas=auto -y
 ```  


## Validator Kontrol
[Explorer](https://explorer-tropos.thestratos.org/)

## DAHA FAZLA SORUNUZ VARSA STRATOS TÜRKİYE TELEGRAM GRUBU

[Stratos Türkiye Telegram Sayfası](https://t.me/StratosTurkish)

## FAYDALI KOMUTLAR

### Logları Kontrol Etme 
```shell
journalctl -fu stchaind -o cat
```

### Sistemi Başlatma
```shell
systemctl start stchaind
```

### Sistemi Durdurma
```shell
systemctl stop stchaind
```

### Sistemi Yeniden Başlatma
```shell
systemctl restart stchaind
```

### Node Senkronizasyon Durumu
```shell
stchaind status 2>&1 | jq .SyncInfo
```

### Validator Bilgileri
```shell
stchaind status 2>&1 | jq .ValidatorInfo
```

### Node Bilgileri
```shell
stchaind status 2>&1 | jq .NodeInfo
```

### Node ID Öğrenme
```shell
stchaind tendermint show-node-id
```

### Node IP Adresini Öğrenme
```shell
curl icanhazip.com
```

### Peer Adresinizi Öğrenme
```shell
echo $(stchaind tendermint show-node-id)@$(curl ifconfig.me)16656
```

### Cüzdanların Listesine Bakma
```shell
stchaind keys list
```

### Cüzdanı İçeri Aktarma
```shell
stchaind keys add $WALLET --recover
```

### Cüzdanı Silme
```shell
stchaind keys delete CUZDAN_ADI
```

### Cüzdan Bakiyesine Bakma
```shell
stchaind query bank balances CUZDAN_ADRESI
```

### Bir Cüzdandan Diğer Bir Cüzdana Transfer Yapma
```shell
stchaind tx bank send CUZDAN_ADRESI GONDERILECEK_CUZDAN_ADRESI 100000000ustrd
```

### Proposal Oylamasına Katılma
```shell
stchaind tx gov vote 1 yes --from $WALLET --chain-id=CHAIN_ID 
```

### Validatore Stake Etme  Delegate Etme
```shell
stchaind tx staking delegate $VALOPER_ADDRESS 100000000utoi --from=$WALLET --chain-id=C$HAIN_ID  --gas=auto
```

### Mevcut Validatorden Diğer Validatore Stake Etme  Redelegate Etme
```shell
stchaind tx staking redelegate MevcutValidatorAdresi StakeEdilecekYeniValidatorAdresi 100000000ustrd --from=WALLET --chain-id=CHAIN_ID  --gas=auto
```

### Ödülleri Çekme
```shell
stchaind tx distribution withdraw-all-rewards --from=$WALLET --chain-id=CHAIN_ID  --gas=auto
```

### Komisyon Ödüllerini Çekme
```shell
stchaind tx distribution withdraw-rewards VALIDATOR_ADRESI --from=$WALLET --commission --chain-id=CHAIN_ID 
```

### Validator İsmini Değiştirme
```shell
stchaind tx staking edit-validator 
--moniker=YENI_NODE_ADI 
--chain-id=$CHAIN_ID  
--from=$WALLET
```

### Validatoru Jail Durumundan Kurtarma 
```shell
stchaind tx slashing unjail 
  --broadcast-mode=block 
  --from=$WALLET 
  --chain-id=$CHAIN_ID  
  --gas=auto
```

### Node'u Tamamen Silme 
```shell
sudo systemctl stop stchaind && 
sudo systemctl disable stchaind && 
rm etc/systemd/system/stride.service && 
systemctl daemon-reload && 
cd $HOME && 
rm -rf .stride stride && 
rm -rf $(which stchaind)
```

### Hesaplar

[Linktree](https://linktr.ee.mehmetkoltigin)

[Twitter](https://twitter.com.mehmetkoltigin)

### Komunite 
[Forum Rues](https://forum.rues.infoindex.php)

[Telegram Rues Announcement](https://t.meRuesAnnouncement)

[Telegram Rues Chat](https://t.meRuesChat)

[Telegram Rues Node](https://t.meRuesNode)

[Telegram Rues Node Chat](https://t.meRuesNodeChat)
