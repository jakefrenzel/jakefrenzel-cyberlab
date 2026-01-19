# 🏗️ 01 - Network Architecture (Implementation)

## Objective
Document the physical and logical network architecture implemented in Cyberlab v2, including devices used, physical connections, and network topology.

---

## Hardware

| Device | Purpose |
|--------|---------|
| Protectli Vault VP2440 | Firewall / Router |
| Netgear GS308EP | Managed Switch |
| Beryl AX | Router (AP Mode) |
| Proxmox Host | Virtualization Server |
| Dell Laptop | Admin PC |
| Modem | Internet Access |

---

## Physical Network Connections

| From | To | Notes |
|------|----|-------|
| Modem | Protectli WAN | Internet access |
| Protectli LAN | Switch Port 1 | Trunk / VLAN tagging |
| Beryl AX | Switch Port 2 | Access port |
| Proxmox Host | Switch port 3 | VLAN trunk |
| Admin PC | Switch port 4 | Temporary management |

---

## Logical Network Design
| Network | VLAN | Subnet | Purpose |
|---------|------|--------|---------|
| Management | 10 | 10.27.10.0 | Firewall, Switch, Router, Proxmox, Admin access |
| Workstations | 20 | 10.27.20.0 | Normal users, primary internal network |
| Servers | 30 | 10.27.30.0 | AD, file servers, DNS, internal services |
| Lab | 40 | 10.27.40.0 | Kali, malware analysis, attack VMs |
| DMZ | 50 | 10.27.50.0 | Public-facing services |
| Guest | 60 | 10.27.60.0 | Guest internet access |
| Monitoring | 70 | 10.27.70.0 | IDS, Suricata, monitoring, logging |

---

## Network Diagram

![CyberLab v2 Network Topology](../architecture/v2-topology.png)
