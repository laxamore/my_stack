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

Copy caddy-l4 binary included in this repository to /usr/sbin, or build it with xcaddy or download it from with l4 extra feaure [here](https://caddyserver.com/download)

```bash
sudo cp caddy/caddy-l4 /usr/sbin/caddy
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
[Unit]
Description=Caddy Reverse Proxy
After=network.target

[Service]
Type=simple
Restart=on-failure

ExecStart=/usr/sbin/caddy --config /etc/caddy/config.json

[Install]
WantedBy=multi-user.target
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

# Credits

Netmaker https://github.com/gravitl/netmaker.
CoreDNS https://github.com/coredns/coredns.
Caddy Server https://github.com/caddyserver/caddy.
