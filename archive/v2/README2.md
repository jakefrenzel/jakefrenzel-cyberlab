> [!IMPORTANT]
> This repository is a work in progess, started February 2026.

# 💻 Jake’s Cybersecurity & IT Networking Lab
This repository documents the design and implementation of a personal cybersecurity home lab built to develop hands-on experience with network security, virtualization, firewall configuration, and secure infrastructure design. The lab is designed to simulate an enterprise-grade network and is intentionally documented to be reproducible and educational, allowing others to follow the same process and learn from both the design decisions and lessons learned.

---

## 🎯 Objectives
- Design and implement a segmented network using industry-relevant tools
- Apply security best practices such as least privilege and network segmentation
- Gain hands-on experience with firewall configuration and traffic control
- Practice documenting technical systems clearly and accurately

---

## 🛠 Technologies Used
- pfSense (Firewall & routing)
- Proxmox (Virtualization)
- Kali Linux (Offensive security testing)
- Vulnerable lab services (e.g., OWASP Juice Shop)
- Managed network switch

---

## 💡 Skills Demonstrated
- Network design and IP addressing
- VLAN configuration and segmentation
- Firewall rule design and validation
- Virtualization and lab orchestration
- Security-focused documentation and analysis

---

## 🗂 Repository Structure
```text
/
├── README.md
├── logs.md
├── config/
│   └── router.md
├── hardware/
│   ├── inventory.md
│   └── topology-diagram.md
├── setup/
│   ├── vpn.md
│   ├── network.md
│   ├── owasp-juice-shop.md
│   ├── proxmox.md
│   ├── virtualization.md
│   └── wazuh.md
└── assets/
    └── images/
        └── topology.png
```

| Folder | Description |
|--------|-------------|
| `architecture/` | Network diagrams, topology, and planning documentation |
| `hardware/` | Hardware inventory, specifications, and rationale for choices |
| `setup/` | Step-by-step guides for installing and configuring systems and services |
| `security/` | Firewall rules, hardening procedures, and monitoring setup |
| `labs/` | Individual lab exercises, including offensive and defensive security tests |
| `lessons-learned/` | Reflections, project iteration notes, and best practices learned |
| `resources/` | Reference materials, external tools, and useful links |

---

## 🔄 Project Iteration

This lab has undergone multiple iterations. The current version reflects lessons learned from an initial deployment and emphasizes improved planning, documentation, and security posture.

Details can be found in the `lessons-learned/` directory.

---

## 🏗 How to Use This Repository

This repository is intended for:
- Cybersecurity students building a homelab
- Individuals learning network security concepts
- Anyone interested in practical firewall and segmentation design

Readers are encouraged to follow the setup guides in order and review the lessons learned to avoid common pitfalls.

---

## ⚠️ Disclaimer

This lab is for educational purposes only. All vulnerable services are intentionally deployed in isolated environments.

---
