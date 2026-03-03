# Intrusion Detection System

## Raspberry Pi Ubunutu Server Setup

1. Install Ubunutu 24.04 LTS server

2. Update the Ubunutu server
```shell
sudo apt update && sudo apt upgrade -y
```

## Switch Configuration - Port Mirroring

1. Log in to Netgear web GUI

2. Navigate to port mirroring section
```netgear-webgui
Diagnostics -> Port Mirroring
```

3. Enable port mirroring

4. Source port: `all ports besides the Raspberry Pi's port`

5. Destination port: `the Raspberry Pi's port`

6. Apply changes

## Suricata Setup

1. Install using: `sudo apt install -y suricata`

2. Verify version: `suricata --build-info | grep "version"`

### Disable NIC Offloading

1. Allow Suricata to see packets by disabling offloading: `sudo ethtool -K eth0 gro off lro off tso off gso off`

2. Verify that most of the offloads are set to off using: `ethtool -k eth0 | grep offload`

### Enable Promiscuous Mode

1. Allow the Ubunutu Server to capture traffic not destined to just itself: `sudo ip link set eth0 promisc on`

2. Verify that you see the **PROMISC** flag: `ip link show eth0`

### Verify Port Mirroring + NIC Offloading + Promiscuous

1. Run a quick test to see some traffic using tcpdump: `sudo tcpdump -i eth0 -nn`

2. Ping another LAN device or run an nmap scan and look for ICMP requests and replies

3. If you see them from the Raspberry Pi, it is working

## Suricata Configuration

1. Verify the config file exists
```shell
sudo cat /etc/suricata/suricata.yaml
```

### Install Rules
1. Install IDS rules
```shell
sudo suricata-update
```

## Test Suricata
1. Test run Suricata
```shell
sudo suricata -i eth0 -v
```

2. Verify engine is running:
`suricata: This is Suricata version 7.0.3 RELEASE running in SYSTEM mode`

3. Verify rules are processed
`detect: 48746 signatures processed`

4. Ignore errors for `hugepages` and `af-packet`

5. From another device run an aggressive nmap scan on a LAN device such as the pfsense firewall: `nmap -A 10.27.0.1`

6. Ctrl + C to stop Suricata

7. View logs and search for `ET SCAN` alert
```shell
sudo cat /var/log/suricata/fast.log
```

## Enable Suricata as a Service
Testing has been complete and now we can deploy Suricata

### Start Suricata on Boot

1. Enable Suricata serivce
```shell
sudo systemctl enable suricata
```

2. Start Suricata
```shell
sudo systemctl start suricata
```

### Create Boot Script to Make Offload and Promisc Persistent

1. Create a boot script
```shell
sudo nano /usr/local/bin/ids-interface-tuning.sh
```

2. Paste this inside
```shell
#!/bin/bash

# Disable NIC offloading
/usr/sbin/ethtool -K eth0 gro off
/usr/sbin/ethtool -K eth0 gso off
/usr/sbin/ethtool -K eth0 tso off
/usr/sbin/ethtool -K eth0 lro off

# Enable promiscuous mode
/usr/sbin/ip link set eth0 promisc on

exit 0
```

3. Save and exit

4. Make it executable
```shell
sudo chmod +x /usr/local/bin/ids-interface-tuning.sh
```

5. Create a systemd service
```shell
sudo nano /etc/systemd/system/ids-interface-tuning.service
```

6. Paste this inside
```shell
[Unit]
Description=IDS Interface Tuning (Disable Offloading + Enable Promisc)
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/ids-interface-tuning.sh
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```

7. Save and exit

### Enable Boot Script

1. Enable it
```shell
sudo systemctl daemon-reload
```
```shell
sudo systemctl enable ids-interface-tuning.service
```
```shell
sudo systemctl start ids-interface-tuning.service
```

2. Verify it works
```shell
sudo reboot
```

3. Confirm gro, lro, tso, gso are OFF
```shell
sudo ethtool -k eth0
```

4. Verify PROMISC flag is present
```shell
ip link show eth0
```
