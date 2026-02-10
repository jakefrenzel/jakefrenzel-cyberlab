# 🏗️ 01 - Network Architecture

## 🎯 Objective
Plan the physical and logical layout of the cybersecurity lab, including all devices, connections, and network segments. This ensures a clear foundation before assigning IPs or configuring VLANs.

## 📝 Prerequisites
- Basic understanding of networking concepts (switches, routers, VLANs)  
- Access to lab hardware (switches, firewall, Proxmox host, lab machines)  
- Tools for diagramming (draw.io, Lucidchart, or pen and paper)  

## ⚙️ Steps

1. **Identify all devices**  
   - Firewalls  
   - Proxmox host  
   - Managed switches  
   - Lab machines (VMs or physical)  
   - Other peripherals  

2. **Decide physical connections**  
   - Which switch ports connect to which devices  
   - Which firewall interfaces connect to WAN / LAN / lab networks  

3. **Plan network segmentation**  
   - Management network  
   - User network  
   - Lab / test network  
   - Consider VLAN boundaries now to make IP planning easier  

4. **Draw a topology diagram**  
   - Include all devices, interfaces, and network segments  
   - Save in `architecture/` folder as `.png` or `.drawio`  

5. **Document the architecture**  
   - List each device, purpose, and connection  
   - Note VLANs or segments for each port if pre-decided  

## ✅ Verification
- Ensure diagram matches physical connections  
- All devices can be located and identified in the lab  
- Confirm that planned VLANs match physical ports (if pre-assigned)  

## ✍️ Lessons Learned
- Planning the physical and logical network first reduces confusion later  
- Diagramming helps visualize VLANs and traffic flow  
- Early architecture planning makes IP and firewall setup much easier

## ➡️ Next Up

Navigate to next section: [02 - IP Schemes](02-ip_schemes.md)
