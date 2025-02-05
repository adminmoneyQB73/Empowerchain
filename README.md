Hardware Requirements

Minimum

3CPU 4RAM 80GB
Recommended

4CPU 8RAM 160GB
Rent On Hetzner | Rent On OVH
Dependencies Installation

**Install dependencies for building from source**
```
sudo apt update
sudo apt install -y curl git jq lz4 build-essential
```

**Install Go**
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
```
Node Installation

Node Name

Your Node Name
Port prefix

174
**Clone project repository**
```
cd && rm -rf empowerchain
git clone https://github.com/EmpowerPlastic/empowerchain
cd empowerchain
git checkout v2.0.0
```

**Build binary**
```
make install
```

**Prepare cosmovisor directories**
```
mkdir -p $HOME/.empowerchain/cosmovisor/genesis/bin
ln -s $HOME/.empowerchain/cosmovisor/genesis $HOME/.empowerchain/cosmovisor/current -f
```

**Copy binary to cosmovisor directory**
```
cp $(which empowerd) $HOME/.empowerchain/cosmovisor/genesis/bin
```

**Set node CLI configuration**
```
empowerd config chain-id empowerchain-1
empowerd config keyring-backend file
empowerd config node tcp://localhost:17457
```

**Initialize the node**
```
empowerd init "Your Node Name" --chain-id empowerchain-1
```

**Download genesis and addrbook files**
```
curl -L https://snapshots.nodejumper.io/empower/genesis.json > $HOME/.empowerchain/config/genesis.json
curl -L https://snapshots.nodejumper.io/empower/addrbook.json > $HOME/.empowerchain/config/addrbook.json
```

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "f2ed98cf518b501b6d1c10c4a16d0dfbc4a9cc98@tenderseed.ccvalidators.com:27001,e16668ddd526f4e114ebb6c4714f0c18c0add8f8@empower-seed.zenscape.one:26656,6740fa259552a628266a85de8c2a3dee7702b8f9@empower-mainnet-seed.itrocket.net:14656,ebc272824924ea1a27ea3183dd0b9ba713494f83@empowerchain-mainnet-seed.autostake.com:27326,a1427b456513ab70967a2a5c618d347bc89e8848@seed.empowerchain.io:26656,9aa8a73ea9364aa3cf7806d4dd25b6aed88d8152@empowerchain.seed.mzonder.com:12156,8542cd7e6bf9d260fef543bc49e59be5a3fa9074@seed.publicnode.com:26656,b85358e035343a3b15e77e1102857dcdaf70053b@seeds.bluestake.net:21956,ebc272824924ea1a27ea3183dd0b9ba713494f83@empowerchain-mainnet-peer.autostake.com:27326,178718a993161cc20f9d0de2bbef9a3aec5c1d3d@rpc.empower.indonode.net:52656"|' $HOME/.empowerchain/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.025umpwr"|' $HOME/.empowerchain/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.empowerchain/config/app.toml

# Enable prometheus
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.empowerchain/config/config.toml

# Change ports
sed -i -e "s%:1317%:17417%; s%:8080%:17480%; s%:9090%:17490%; s%:9091%:17491%; s%:8545%:17445%; s%:8546%:17446%; s%:6065%:17465%" $HOME/.empowerchain/config/app.toml
sed -i -e "s%:26658%:17458%; s%:26657%:17457%; s%:6060%:17460%; s%:26656%:17456%; s%:26660%:17461%" $HOME/.empowerchain/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots.nodejumper.io/empower/empower_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.empowerchain"

# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.0

# Create a service
sudo tee /etc/systemd/system/empower.service > /dev/null << EOF
[Unit]
Description=EmpowerChain node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.empowerchain
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.empowerchain"
Environment="DAEMON_NAME=empowerd"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable empower.service

# Start the service and check the logs
sudo systemctl start empower.service
sudo journalctl -u empower.service -f --no-hostname -o cat
Secure Server Setup (Optional)

# generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE
ssh-keygen -t rsa

# save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY
cat ~/.ssh/id_rsa.pub
# upgrade system packages
sudo apt update
sudo apt upgrade -y

# add new admin user
sudo adduser admin --disabled-password -q

# upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# disable root login, disable password authentication, use ssh keys only
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd

# install fail2ban
sudo apt install -y fail2ban

# install and configure firewall
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656

# make sure you expose ALL necessary ports, only after that enable firewall
sudo ufw enable

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP
