# 🔒 03 - Firewall Setup

## 🎯 Objective
Plan the firewall configuration for your lab, including interfaces, VLANs, routing, and rules. The goal is to ensure secure, segmented traffic flows and controlled access between networks.

## 📝 Prerequisites
- Completed Network Architecture (`01 - Network Architecture.md`)  
- Planned IP Schemes and VLANs (`02 - IP Schemes.md`)  
- Understanding of basic firewall concepts and rule order  
- Access to your firewall model documentation (e.g., pfSense)  

## ⚙️ Steps

1. **Define firewall interfaces**  
   - Assign interfaces to each VLAN or network segment  
   - Label interfaces clearly (e.g., WAN, LAN, Management, Lab)  

2. **Plan default firewall policies**  
   - Decide whether default is allow or deny per interface  
   - Consider least privilege principle  

3. **Design specific rules per interface**  
   Example for LAN / Lab networks:  
   - Management → can reach all networks  
   - Lab → can reach lab network only, no access to management VLAN  
   - User → can access the internet but limited to internal resources if needed  

4. **Plan NAT (Network Address Translation) rules**  
   - Define which networks need outbound NAT  
   - Consider whether inbound NAT or port forwarding is needed for lab VMs  

5. **Plan logging and monitoring**  
   - Decide which traffic should be logged  
   - Plan alerts for suspicious traffic (optional in planning stage)  

6. **Document your firewall plan**  
   - Use a table to summarize rules:  
     | Source VLAN | Destination VLAN | Protocol / Port | Action |
     |-------------|----------------|----------------|--------|
     | Lab VLAN 30 | Internet | Any | Allow |
     | Lab VLAN 30 | Management VLAN 10 | Any | Deny |
     | User VLAN 20 | Internet | Any | Allow |
     | User VLAN 20 | Management VLAN 10 | Any | Deny |

## ✅ Verification
- Ensure each planned rule aligns with the network segmentation plan  
- Verify that no unintended traffic is allowed between VLANs  
- Make sure NAT rules cover required outbound traffic  
- Document any potential conflicts or exceptions  

## ✍️ Lessons Learned
- Planning firewall rules early prevents accidental exposure of sensitive networks  
- Visualizing source and destination VLANs helps avoid misconfigurations  
- Logging and monitoring plans allow for easier troubleshooting once implemented

## ➡️ Next Up

Navigate to next section: [04 - Virtual Machines](04-virtual_machines.md)
