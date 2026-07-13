# 🧪 CyberLab Documentation

This is the second iteration of my cybersecurity homelab, rebuilt to improve network design, documentation, and security posture based on lessons learned from v1.

## 📖 Summary
This iteration focuses on improved IP planning, network segmentation, and step-by-step reproducibility for learning purposes.

## 🗺️ Roadmap / Recommended Build Order

Follow these steps to replicate the lab:

1. **🏗️ 01 - Network Architecture**: Plan physical and logical network layout
2. **📐 02 - IP Schemes**: Plan IP ranges and VLANs 
3. **🔒 03 - Firewall Setup**: Configure pfSense with rules and routing  
4. **🖥️ 04 - Virtual Machines**: Deploy lab VMs on Proxmox  
5. **🌐 05 - Network Segmentation**: Apply VLANs and isolation  
6. **🛡️ 06 - Security Hardening**: Apply firewall hardening and monitoring  
7. **🧪 07 - Labs / Exercises**: Deploy vulnerable apps, capture traffic, analyze

---

## 📝 Notes
- Each step has its own markdown file in `setup/` for detailed instructions.  
- Verification and testing steps are included in each setup file.  
- See the folder table below for a quick reference to all sections.

---

## 🗂 Repository Structure

| Folder | Description |
|--------|-------------|
| 📐 `architecture/` | Network diagrams, topology, and planning documentation |
| 🖥️ `hardware/` | Hardware inventory, specifications, and rationale for choices |
| ⚙️ `setup/` | Step-by-step guides for installing and configuring systems and services |
| 🔒 `security/` | Firewall rules, hardening procedures, and monitoring setup |
| 🧪 `labs/` | Individual lab exercises, including offensive and defensive security tests |
| ✍️ `lessons-learned/` | Reflections, project iteration notes, and best practices learned |
| 📚 `resources/` | Reference materials, external tools, and useful links |
