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

3. Run a quick test to see some traffic: `sudo tcpdump -i eth0 -nn`


