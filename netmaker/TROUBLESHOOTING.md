# Netmaker Troubleshooting

## 1. Client Node Not Reconnecting After Reboot

#### 1.1 Check netclient service status

Check if netclient service / daemon is already enabled and running.

```bash
sudo systemctl status netclient
```

If not, enable and start the service.

```bash
sudo systemctl enable netclient
sudo systemctl start netclient
```

#### 1.2 Create custom systemd service and script

If netclient daemon is already running, but still not reconnecting, create a custom systemd service and script to run netclient disconnect and connect commands.

Create a file named `connect-netclient.service` in `/etc/systemd/system/` directory with the following content:

```unit
[Unit]
Description=Connect Netclient
After=network-online.target
Wants=network-online.target

[Service]
User=root
Type=simple
ExecStart=/sbin/connect-netclient.sh
Restart=on-failure
RestartSec=15s

[Install]
WantedBy=multi-user.target
```

Create a file named `connect-netclient.sh` in `/sbin/` directory with the following content:

```bash
#!/bin/bash
CONNECTED=0

while [[ $CONNECTED -eq 0 ]]; do
    netclient disconnect -n <NETWORK_NAME> > /dev/null 2>&1
    netclient connect -n <NETWORK_NAME> > /dev/null 2>&1
    if [[ $? -eq 0 ]]; then
        CONNECTED=1
    fi
done

netclient pull > /dev/null 2>&1
```

Make the script executable:

```bash
sudo chmod +x /sbin/connect-netclient.sh
```

Enable and start the service:

```bash
sudo systemctl enable connect-netclient
sudo systemctl start connect-netclient
```
