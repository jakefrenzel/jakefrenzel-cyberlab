# 🌐 05 - Network Segmentation

## 🎯 Objective
Plan the segmentation of your lab network using VLANs and subnets to improve security, traffic management, and troubleshooting. Define which networks can communicate and which should be isolated.

## 📝 Prerequisites
- Completed Network Architecture (`01 - Network Architecture.md`)  
- Planned IP Schemes (`02 - IP Schemes.md`)  
- Firewall rules plan (`03 - Firewall Setup.md`)  
- Virtual Machine plan (`04 - Virtual Machines.md`)  

## ⚙️ Steps

1. **Identify network segments to isolate**  
   - Management VLAN: firewall, Proxmox management, monitoring servers  
   - User VLAN: optional client access  
   - Lab VLAN: VMs for testing and vulnerable apps  

2. **Map VLAN IDs to each segment**  
   Example:  
   | Network | VLAN ID | Purpose |
   |---------|---------|---------|
   | Management | 10 | Admin / monitoring |
   | User | 20 | Optional client access |
   | Lab | 30 | Pen testing / vulnerable apps |

3. **Plan switch port assignments**  
   - Assign physical ports to VLANs where applicable  
   - Plan trunk ports for VLAN tagging to hosts or Proxmox  

4. **Define inter-VLAN access rules**  
   - Which VLANs can communicate with each other?  
   - Example: Lab VLAN cannot access Management VLAN  
   - Document exceptions if needed  

5. **Document segmentation plan**  
   - Use a table or diagram for clarity  
   - Reference planned firewall rules to enforce isolation  

6. **Plan verification steps**  
   - Devices in the same VLAN can communicate  
   - Devices in isolated VLANs cannot reach restricted networks  
   - Trunked connections pass VLAN tags correctly  

## ✅ Verification
- Check that each VLAN is assigned correctly on switches and hosts  
- Confirm isolation between VLANs matches your planned policy  
- Test connectivity using ping or small traffic tests between networks  

## ➡️ Next Up

Navigate to next section: [06 - Security Hardening](06-security_hardening.md)
## ✍️ Lessons Learned
- Early VLAN planning reduces misconfigurations and troubleshooting later  
- Clear documentation of allowed and restricted traffic is critical  
- Mapping VLANs to firewall rules ensures consistency and security
