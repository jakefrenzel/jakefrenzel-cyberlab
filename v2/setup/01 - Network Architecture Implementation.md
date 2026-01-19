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
| Management | 10 | 10.27.10.0 | Administration and maintenance |
| LAN | 20 | 10.27.20.0 | Firewall, Switch, Router, Proxmox |
