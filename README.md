> [!IMPORTANT]
> This repository is a work in progess, started February 2026.

> [!NOTE]  
> All IP addresses, hostnames, usernames, and other identifying network information shown in this documentation are examples or sanitized values. They are not the actual values used in my home lab, but they follow the same structure and format for documentation purposes.

# 💻 Jake’s Cybersecurity & IT Networking Lab
This repository documents the design and implementation of a personal cybersecurity home lab built to gain hands-on experience with network security, virtualization, firewall configuration, and secure infrastructure design. The lab is designed to simulate an enterprise-grade network and is intentionally documented to be reproducible and educational, allowing others to follow the same process and learn from both the design decisions and lessons learned.

---

## 🚀 Get started
[**01 - Network Architecture:** Plan physical and logical network layout](<cyberlab-documentation/01 - Network Architecture.md>)

[**02 - IP Schemes:** Plan IP ranges and VLANs](<cyberlab-documentation/02 - IP Schemes.md>)

[**03 - pfSense Firewall:** Install pfSense and configure VLANs, DHCP, DNS, and firewall rules](<cyberlab-documentation/03 - pfSense Firewall.md>)

[**04 - Intrusion Detection System:** Deploy Suricata and Zeek on mirrored traffic](<cyberlab-documentation/04 - Intrusion Detection System.md>)

[**05 - Kubernetes Lab Node:** Proxmox, Terraform, and Talos Linux as code](<cyberlab-documentation/05 - Kubernetes Lab Node.md>)

---

## 🎯 Objectives
- [x] Gain experience with firewall configuration and traffic control
- [ ] Practice documenting technical systems clearly and accurately
- [x] Design and implement a segmented network with VLANs
- [ ] Define firewall rules following the principle of least privilege
- [x] Implement intrusion detection and preventions sytems (IDPS)
- [ ] Implement a VPN for secure remote connections over the public internet
- [ ] Set up advanced monitoring and visibility with a SIEM
- [ ] Apply security hardening across the network
- [ ] Create and experiment with offensive and defensive cybersecurity exercises 

---

## 🛠 Technologies Used
- pfSense — Firewall, routing, and network management
- Proxmox VE — Virtualization and VM management
- VLANs — Network segmentation and isolation
- Suricata — Network intrusion detection and prevention (IDS/IPS)
- Zeek — Network security monitoring and traffic analysis
- ELK Stack — Log collection, analysis, and visualization
- Tailscale — Secure remote access and private networking
- Terraform — Infrastructure as code for Proxmox VMs

---

## 💡 Skills Demonstrated
- Network design and IP addressing
- VLAN configuration and segmentation
- Firewall rule design and validation
- Virtualization and lab orchestration
- Security-focused documentation and analysis

---


> [!WARNING]
> This lab is for educational purposes only. All vulnerable services are intentionally deployed in isolated environments.

---
