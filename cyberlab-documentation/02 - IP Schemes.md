# 📐 02 - IP Schemes

## Objectives

Design and document the IP addressing scheme for the home lab, including subnets, VLAN assignments, static IPs, and DHCP ranges.

## IP Addressing Conventions
Every VLAN follows the same layout for the last octet, so any address tells you what kind of device it is

| Range | Use |
|-------|-----|
| .1 | Gateway (pfSense VLAN interface) |
| .2 – .9 | Core infrastructure |
| .10 – .99 | Services and servers |
| .100 – .199 | DHCP pool |
| .200 – .254 | Reserved / testing |

## Network Subnets

| Network | VLAN | Subnet | Gateway | DHCP Range |
|---------|------|--------|---------|------------|
| Management | 10 | 10.27.10.0/24 | 10.27.10.1 | None (static only) |
| Workstations | 20 | 10.27.20.0/24 | 10.27.20.1 | 10.27.20.100 – 10.27.20.199 |
| Servers | 30 | 10.27.30.0/24 | 10.27.30.1 | 10.27.30.100 – 10.27.30.199 |
| Lab | 40 | 10.27.40.0/24 | 10.27.40.1 | 10.27.40.100 – 10.27.40.199 |
| DMZ | 50 | 10.27.50.0/24 | 10.27.50.1 | 10.27.50.100 – 10.27.50.199 |
| Guest | 60 | 10.27.60.0/24 | 10.27.60.1 | 10.27.60.100 – 10.27.60.199 |
| Monitoring | 70 | 10.27.70.0/24 | 10.27.70.1 | 10.27.70.100 – 10.27.70.199 |

> [!NOTE]
> The Management VLAN has no DHCP pool. Every device on it is assigned a static IP, which keeps infrastructure addresses predictable and means an unknown device plugged into a Management port gets no address.

## Static IP Assignments

| Device | VLAN | IP Address | Purpose |
|--------|------|------------|---------|
| pfSense (Protectli VP2440) | 10 | 10.27.10.1 | Gateway / firewall |
| Netgear GS308EP | 10 | 10.27.10.2 | Switch management |
| Proxmox Host | 10 | 10.27.10.3 | Virtualization management (web GUI on port 8006) |
| ELK Stack VM | 10 | 10.27.10.5 | Log ingestion and dashboards |
| Raspberry Pi (management NIC) | 10 | 10.27.10.7 | IDS sensor management and log shipping |

> [!NOTE]
> The Raspberry Pi has two network interfaces. Only the USB management NIC has an IP. The built-in `eth0` is the capture interface on the mirror port and intentionally has **no IP address**. See [04 - Intrusion Detection System](<04 - Intrusion Detection System.md>).

## Planned Static IPs

| Device | VLAN | IP Address | Purpose |
|--------|------|------------|---------|
| Web Server | 30 | 10.27.30.10 | Hosting internal apps |
| Database Server | 30 | 10.27.30.11 | Backend for lab services |
| Metasploitable | 40 | 10.27.40.10 | Exploitation practice |
| Public Web Server | 50 | 10.27.50.10 | Internet-facing app |
| Raspberry Pi (management NIC) | 70 | 10.27.70.10 | IDS sensor (moving from 10.27.10.7) |
| ELK Stack VM | 70 | 10.27.70.20 | Log ingestion and dashboards (moving from 10.27.10.5) |
| Grafana | 70 | 10.27.70.30 | Visualization dashboards |

> [!IMPORTANT]
> The IDS sensor and ELK VM currently sit on the Management VLAN so log shipping needs no firewall rules. They belong on the Monitoring VLAN and should be **moved together**, so Filebeat → Logstash stays an intra-VLAN connection.

## VM Network Assignments

| VM | VLAN | IP Type |
|----|------|---------|
| ELK Stack | 10 | Static |
| Kali Linux | 40 | DHCP |
| Metasploitable | 40 | Static |

## Steps taken
- Defined a /24 subnet per VLAN, with the third octet matching the VLAN ID
- Standardized the last octet layout across all VLANs
- Separated static and DHCP ranges

1. Created a gateway IP on pfSense for each VLAN interface (`.1`)
2. Configured DHCP pools (`.100` – `.199`) for VLANs 20 – 70
3. Left the Management VLAN without a DHCP pool
4. Assigned static IPs to infrastructure devices on the Management VLAN
5. Documented all assignments in this file

## Notes & Observations
1. Matching the third octet to the VLAN ID (VLAN 40 → `10.27.40.0/24`) made it easy to tell which network an address belongs to
2. Keeping the same last octet layout in every VLAN made static and DHCP ranges predictable
3. Documenting IP assignments early prevented conflicts, including two planned servers that were originally given the same address
4. A single wrong octet can break things silently: Filebeat was pointed at `10.27.0.5` instead of `10.27.10.5` and failed without an obvious error. Check addresses against this table when configuring services

## ➡️ Next Up
[**03 - pfSense Firewall:** Install pfSense and configure VLANs, DHCP, DNS, and firewall rules](<03 - pfSense Firewall.md>)
