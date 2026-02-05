# 📐 02 - IP Schemes (Implementation)

## 🎯 Objective
Document the actual IP addressing scheme implemented in CyberLab v2, including subnets, VLAN assignments, static IPs, and DHCP ranges.

---

## 📏 IP Addressing Conventions
- .1        Network gateway
- .2 – .9     Core infrastructure
- .10 – .99   Services
- .100 – .199 DHCP range
- .200+     Reserved / testing

---

## 🌐 Implemented Network Subnets

| Network | VLAN | Subnet | Gateway |
|---------|------|--------|---------|
| Management | 10 | 10.27.10.0/24 | 10.27.10.1 |
| Workstations | 20 | 10.27.20.0/24 | 10.27.20.1 |
| Servers | 30 | 10.27.30.0/24 | 10.27.30.1 |
| Lab | 40 | 10.27.40.0/24 | 10.27.40.1 |
| DMZ | 50 | 10.27.50.0/24 | 10.27.50.1 |
| Guest | 60 | 10.27.60.0/24 | 10.27.60.1 |
| Monitoring | 70 | 10.27.70.0/24 | 10.27.70.1 |

---

## 🧱 Static IP Assignments

| Device | Interface | VLAN | IP Address | Purpose |
|--------|-----------|------|------------|---------|
| Firewall (pfSense) | LAN | 10 | 10.27.10.1 | Gateway |
| Proxmox Host | vmbr0 | 10 | 10.27.10.2 | VM management |
| Switch | Management | 10 | 10.27.10.3 | Switch admin |
| OWASP Juice Shop | ens18 | 40 | 10.27.40.10 | Web app testing |
| Wazuh Server | ens18 | 70 | 10.27.70.40 | Monitoring |

---

### Planned Static IPs

| Device | Interface | VLAN | IP Address | Purpose |
|--------|-----------|------|------------|---------|
| Web Server | ens18 | 30 | 10.27.30.10 | Hosting internal apps |
| Database Server | ens18 | 30 | 10.27.30.10 | Backend for lab services |
| Metasploitable | ens18 | 40 | 10.27.40.10 | Exploitation practice |
| Public Web Server | ens18 | 50 | 10.27.50.10 | Internet-facing app |
| IDS Suricata | ens18 | 70 | 10.27.70.10 | Monitoring |
| Elk Stack | ens18 | 70 | 10.27.70.20 | Log ingestion & analysis |
| Grafana | ens18 | 70 | 10.27.70.30 | Visualization dashboards |

---

## 📡 DHCP Configuration

| Network | VLAN | DHCP Range |
|---------|------|------------|
| User | 20 | 192.168.20.100 – 192.168.20.200 |
| Lab | 30 | 192.168.30.100 – 192.168.30.200 |

> **Note:** Management VLAN uses static IPs only to reduce attack surface and
> ensure predictable access to infrastructure devices.

---

## 🖥️ VM Network Assignments

| VM | VLAN | IP Type |
|----|------|---------|
| Kali Linux | 40 | DHCP |
| Test Linux VM | 40 | DHCP |
| OWASP Juice Shop | 40 | Static |
| Metasploitable | 40 | Static |
| Monitoring VM | 10 | Static |

---

## 🛠️ Steps Taken

1. Defined gateway IPs on pfSense for each VLAN  
2. Configured DHCP scopes for User and Lab networks  
3. Assigned static IPs to infrastructure devices  
4. Verified VLAN tagging and IP assignment on Proxmox  
5. Documented IP assignments in this repository  

---

## ✍️ Notes & Observations

- Keeping management infrastructure on static IPs simplifies access  
- Documenting IP assignments early prevented conflicts  
- Clear separation of static and DHCP ranges improves maintainability  
