# Netmaker Host Network

Version: 0.16.3

With Netmaker configured using Host Network, it is easier to reverse proxy clients network for example your home network to the public internet, without secondary vps as reverse proxy node.

# Installation

## 0. Preqrequisites

- Virtual Machine with Ubuntu 22.04 (Prefered) from (aws, gcp, azure, digital ocean, etc)
- Minimum 1GB RAM 1 CPU
- A publicly owned domain name

## 1. Prepare DNS

Create a wildcard A record pointing to the public IP of your VM. As an example, \*.netmaker.example.com. Alternatively, create records for these specific subdomains:

- dashboard.domain
- api.domain
- broker.domain

## 2. Install Dependencies

```bash
sudo apt update
sudo apt install -y docker.io docker-compose wireguard
```

## 3. Open Firewall Ports

Make sure firewall settings are set for Netmaker both on the VM and with your cloud security groups (AWS, GCP, etc).

```bash
sudo ufw allow proto tcp from any to any port 443 && sudo ufw allow 51821:51830/udp && sudo ufw allow ssh
sudo ufw enable
```

It is also important to make sure the server does not block forwarding traffic (it will do this by default on some providers). To ensure traffic will be forwarded:

```bash
iptables --policy FORWARD ACCEPT
```

## 4. Install Netmaker

Create Netmaker directory.

```bash
sudo mkdir /etc/netmaker
```

Get Netmaker binary.

```bash
sudo wget -o /etc/netmaker/netmaker https://github.com/gravitl/netmaker/releases/download/v0.16.3/netmaker
```

Copy Netmaker binary to /usr/sbin.

```bash
sudo cp /etc/netmaker/netmaker /usr/sbin/netmaker
```

## 5. Configure Netmaker

Create systemd service file. in /etc/systemd/system/netmaker.service with the following configuration:

```bash
[Unit]
Description=Netmaker Server
After=network.target

[Service]
Type=simple
Restart=on-failure

ExecStart=/usr/sbin/netmaker -c /etc/netmaker/netmaker.yaml

[Install]
WantedBy=multi-user.target
```

Generate Master Key for Netmaker.

```bash
tr -dc A-Za-z0-9 </dev/urandom | head -c 30 ; echo ''
```

Replace manually NETMAKER_BASE_DOMAIN, NETMAKER_MASTER_KEY, MQ_ADMIN_PASSWORD with your domain, generated master key, and mosquitto admin password inside netmaker.yaml, or using the sed command below:

```bash
sed -i 's/NETMAKER_BASE_DOMAIN/<your base domain>/g' netmaker.yaml
sed -i 's/NETMAKER_MASTER_KEY/<your master key>/g' netmaker.yaml
sed -i 's/MQ_ADMIN_PASSWORD/<your mq password>/g' netmaker.yaml
```

Update docker-compose.yml NETMAKER_BASE_DOMAIN placeholder with your domain.

```bash
sed -i 's/NETMAKER_BASE_DOMAIN/<your base domain>/g' docker-compose.yml
```

## 6. Perpare Mosquitto

Retrieve Mosquitto configuration file.

```bash
wget -O mosquitto.conf https://raw.githubusercontent.com/gravitl/netmaker/v0.16.3/docker/mosquitto.conf
```

Retrieve Mosquitto start script.

```bash
wget -O wait.sh https://raw.githubusercontent.com/gravitl/netmaker/v0.16.3/docker/wait.sh
chmod +x wait.sh
```

## 7. Perpare CoreDNS

Disable systemd-resolved, for CoreDNS to be able to bind to port 53.

```bash
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
rm /etc/resolv.conf
```

## 8. Install And Configure Caddy Reverse Proxy

Copy caddy-l4 binary included in this repository to /usr/bin, or build it with xcaddy or download it from with l4 extra feaure [here](https://caddyserver.com/download)

```bash
sudo cp caddy/caddy-l4 /usr/bin/caddy
```

Create caddy directory & copy caddy configuration file.

```bash
sudo mkdir /etc/caddy && sudo cp caddy/config.json /etc/caddy/config.json
```

Replace NETMAKER_BASE_DOMAIN and YOUR_EMAIL placeholder inside caddy configuration file.

```bash
sudo sed -i 's/NETMAKER_BASE_DOMAIN/<your base domain>/g' /etc/caddy/config.json
sudo sed -i 's/YOUR_EMAIL/<your email>/g' /etc/caddy/config.json
```

Create systemd service file. in /etc/systemd/system/caddy.service with the following configuration:

```bash
# caddy.service
#
# For using Caddy with a config file.
#
# Make sure the ExecStart and ExecReload commands are correct
# for your installation.
#
# See https://caddyserver.com/docs/install for instructions.
#
# WARNING: This service does not use the --resume flag, so if you
# use the API to make changes, they will be overwritten by the
# Caddyfile next time the service is restarted. If you intend to
# use Caddy's API to configure it, add the --resume flag to the
# `caddy run` command or use the caddy-api.service file instead.

[Unit]
Description=Caddy
Documentation=https://caddyserver.com/docs/
After=network.target network-online.target
Requires=network-online.target

[Service]
Type=notify
User=caddy
Group=caddy
ExecStart=/usr/bin/caddy run --environ --config /etc/caddy/config.json
ExecReload=/usr/bin/caddy reload --config /etc/caddy/config.json --force
TimeoutStopSec=5s
LimitNOFILE=1048576
LimitNPROC=512
PrivateTmp=true
ProtectSystem=full
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

Create caddy user.

```bash
sudo useradd caddy
sudo mkdir /home/caddy
sudo chown caddy:caddy /home/caddy
```

## 9. Start Everything

Reload systemd daemon.

```bash
sudo systemctl daemon-reload
```

Star Netmaker.

```bash
sudo systemctl enable netmaker
sudo systemctl start netmaker
```

Start Caddy.

```bash
sudo systemctl enable caddy
sudo systemctl start caddy
```

Start Mosquitto & Netmaker UI.

```bash
docker-compose up -d
```

Wait 1-2 minutes for caddy to resolve certs for the first time, and then you should be able to access the dashboard at https://dashboard.yourdomain.com.

# Troubleshooting

Some troubleshooting can be found [here](/netmaker/TROUBLESHOOTING.md)

# Credits

Netmaker https://github.com/gravitl/netmaker.
CoreDNS https://github.com/coredns/coredns.
Caddy Server https://github.com/caddyserver/caddy.
