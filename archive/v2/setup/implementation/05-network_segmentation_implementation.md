# 🌐 Network Segmentation — Implementation

This document details how network segmentation was applied in CyberLab v2
using VLANs, managed switches, and firewall policies. Segmentation helps
reduce lateral movement, protect critical resources, and improve monitoring
visibility.

---

## Segmentation Overview

The network is divided into VLANs, each representing a separate security zone:

| VLAN | Network | Purpose | Trust Level |
|------|--------|--------|-------------|
| 10 | 10.27.10.0/24 | Management | Highly Trusted |
| 20 | 10.27.20.0/24 | Workstations | Semi-Trusted |
| 30 | 10.27.30.0/24 | Servers | Restricted |
| 40 | 10.27.40.0/24 | Lab | Untrusted |
| 50 | 10.27.50.0/24 | DMZ | Public-Facing |
| 60 | 10.27.60.0/24 | Guest | Untrusted |
| 70 | 10.27.70.0/24 | Monitoring | Restricted |

Each VLAN is a separate Layer 2 broadcast domain, with traffic routed
through pfSense for Layer 3 enforcement.

---

## 🔌 Switch Configuration

The managed switch enforces VLAN separation:

- VLAN tagging enabled on all networks
- Access ports assigned to specific VLANs
- Unused ports disabled to prevent unauthorized access

<br>

📝 Configuration instructions available: [Switch Configuration](../../configuration/managed-switch.md)

---

## 🔥 Firewall Enforcement

pfSense handles inter-VLAN routing with the following policies:

- Default-deny between all VLANs
- Explicit allow rules for required services only
- Lab and Guest VLANs cannot access Management or Server VLANs
- DMZ isolated from internal networks
- Monitoring VLAN can observe traffic passively but cannot initiate connections

---
