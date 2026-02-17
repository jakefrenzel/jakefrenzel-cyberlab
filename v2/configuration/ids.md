# Intrusion Detection System

## Ubunutu Server Setup
`sudo apt update` and `sudo apt upgrade`

### Disable NIC Offloading

1. Allow Suricata to see packets by disabling offloading: `sudo ethtool -K eth0 gro off lro off tso off gso off`

2. Verify that most of the offloads are set to off using: `ethtool -k eth0 | grep offload`

### Enable Promiscuous Mode

1. Allow the Ubunutu Server to capture traffic not destined to just itself: `sudo ip link set eth0 promisc on`

2. Verify that you see the **PROMISC** flag: `ip link show eth0`

3. Run a quick test to see some traffic: `sudo tcpdump -i eth0 -nn`

## Suricata

### Install Suricata

1. Install using: `sudo apt install -y suricata`

2. Verify version: `suricata --build-info | grep "Version"`
