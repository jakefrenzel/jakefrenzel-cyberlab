# 🏗️ 01 - Network Architecture

## Objective
Document the physical and logical network architecture, including devices used, physical connections, and network topology

## Hardware
| Device | Purpose |
|--------|---------|
| Modem | Internet Access |
| Protectli Vault VP2440 | Firewall / Router |
| Netgear GS308EP | Managed Switch |
| Proxmox Host | Virtualization Server |
| Raspberry Pi | Intrusion Detection System (IDS) |
| Dell Laptop | Admin PC |
| Beryl AX | Access Point |

## Physical Connections
| From | To | Notes |
|------|----|-------|
| Modem | Protectli WAN | Internet access |
| Protectli LAN | Switch Port 1 | Trunk / VLAN tagging |
| Proxmox Host | Switch port 2 | VLAN trunk |
| Raspberry Pi | Switch port 3 | Port Mirroring |
| Raspberry Pi | Switch port 4 | IDS |
| Beryl AX | Switch Port 5 | Access port |
| Admin PC | Switch port 6 | Temporary management |

## Logical Network Design

| Network | VLAN | Subnet | Purpose |
|---------|------|--------|---------|
| Management | 10 | 10.27.10.1 | Firewall, Switch, Router, Proxmox, Admin access |
| Workstations | 20 | 10.27.20.1 | Normal users, primary internal network |
| Servers | 30 | 10.27.30.1 | AD, file servers, DNS, internal services |
| Lab | 40 | 10.27.40.1 | Kali, malware analysis, attack VMs |
| DMZ | 50 | 10.27.50.1 | Public-facing services |
| Guest | 60 | 10.27.60.1 | Guest internet access |
| Monitoring | 70 | 10.27.70.1 | IDS, Suricata, monitoring, logging |

## Network Diagram
