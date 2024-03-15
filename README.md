# Warden Protocol
<img src="https://github.com/hakandemirdev/Warden-Protocol/blob/ddc41a8575952abed29ddfdda98c121c8a5b526a/warden-preview.png" width="auto">

[Warden Web Sitesi](https://wardenprotocol.org) 

[Warden Twitter](https://twitter.com/wardenprotocol)

[Warden Discord](https://discord.com/invite/wardenprotocol)

Öncelikle işletim sistemimizi güncelliyoruz.
```
sudo apt update && sudo apt upgrade -y
```
Temel paketleri yüklüyoruz.
```
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```
Go kurulumunu yapıyoruz.
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
Warden Protocol reposunu klonluyor ve kuruyoruz.
```
cd $HOME
mkdir -p $HOME/.warden/cosmovisor/genesis/bin
git clone --depth 1 --branch v0.1.0 https://github.com/warden-protocol/wardenprotocol/
cd  wardenprotocol/warden/cmd/wardend
go build
```
Gerekli yapılandırmalar.
```
mv wardend $HOME/.warden/cosmovisor/genesis/bin/
sudo ln -s $HOME/.warden/cosmovisor/genesis $HOME/.warden/cosmovisor/current -f
sudo ln -s $HOME/.warden/cosmovisor/current/bin/wardend /usr/local/bin/wardend -f
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0

```
Servis dosyası oluşturalım.

```
sudo tee /etc/systemd/system/wardend.service > /dev/null << EOF
[Unit]
Description=warden node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.warden"
Environment="DAEMON_NAME=wardend"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.warden/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF
```
Systemctl aktifleştirelim.
```
sudo systemctl daemon-reload
sudo systemctl enable wardend
```
Moniker adımızı belirleyelim.
```
wardend init "Moniker_Adiniz" --chain-id alfama
```
Genesis ve Addrbook dosyalarını indirelim.

```
wget -O $HOME/.warden/config/genesis.json "http://37.120.189.81/warden_testnet/genesis.json"
wget -O $HOME/.warden/config/addrbook.json "http://37.120.189.81/warden_testnet/addrbook.json"

```
Gas ayarını yapalım.
```
sed -i.bak -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.0025uward\"/;" ~/.warden/config/app.toml

```
Peer ve Seedleri ayarlayalım.
```
peers="6a8de92a3bb422c10f764fe8b0ab32e1e334d0bd@sentry-1.alfama.wardenprotocol.org:26656,7560460b016ee0867cae5642adace5d011c6c0ae@sentry-2.alfama.wardenprotocol.org:26656,24ad598e2f3fc82630554d98418d26cc3edf28b9@sentry-3.alfama.wardenprotocol.org:26656"
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.warden/config/config.toml

seeds=""
sed -i.bak -e "s/^seeds =.*/seeds = \"$seeds\"/" $HOME/.warden/config/config.toml

```

Hızlı seknronizasyon işlemi
```
cd $HOME
apt install lz4
sudo systemctl stop wardend
cp $HOME/.warden/data/priv_validator_state.json $HOME/.warden/priv_validator_state.json.backup
rm -rf $HOME/.warden/data
curl -o - -L http://37.120.189.81/warden_testnet/warden_snap.tar.lz4 | lz4 -c -d - | tar -x -C $HOME/.warden
mv $HOME/.warden/priv_validator_state.json.backup $HOME/.warden/data/priv_validator_state.json
```
Port ayarlarını yapıyoruz.
```
CUSTOM_PORT=111
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:${CUSTOM_PORT}58\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:${CUSTOM_PORT}57\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:${CUSTOM_PORT}60\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:${CUSTOM_PORT}56\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":${CUSTOM_PORT}66\"%" $HOME/.warden/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:${CUSTOM_PORT}17\"%; s%^address = \":8080\"%address = \":${CUSTOM_PORT}80\"%; s%^address = \"locaklhost:9090\"%address = \"0.0.0.0:${CUSTOM_PORT}90\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:${CUSTOM_PORT}91\"%" $HOME/.warden/config/app.toml

```
Warden Node'umuzu başlatalım.
```
sudo systemctl restart wardend
journalctl -fu wardend -o cat

```
Cüzdan oluşturuyoruz. Mnemonic ve cüzdan adresimi not alalım.
```
wardend keys add cüzdan-adi
```
Daha önce kullandığınız bir cüzdanı import etmek için aşağıdaki komutu kullanabilirsiniz.
```
wardend keys add cüzdan-adi --recover
```
