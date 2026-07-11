# 🖥️ 04 - Virtual Machines

## 🎯 Objective
Plan the virtual machines (VMs) for your lab, including their purpose, network segment, and resources. This ensures a structured and reproducible lab environment before deployment.

## 📝 Prerequisites
- Completed Network Architecture (`01 - Network Architecture.md`)  
- Planned IP Schemes (`02 - IP Schemes.md`)  
- Firewall rules plan (`03 - Firewall Setup.md`)  
- Access to virtualization platform documentation (Proxmox)  

## ⚙️ Steps

1. **Identify required VMs**  
   Examples:  
   - Offensive testing: Kali Linux  
   - Vulnerable applications: OWASP Juice Shop  
   - Monitoring/logging: ELK stack, Wazuh  
   - Optional: Windows/Linux clients for testing  

2. **Assign network segments**  
   - Map each VM to its VLAN or subnet (Lab VLAN, Management VLAN if needed)  
   - Decide if any VM needs dual interfaces for testing routing or firewall rules  

3. **Plan VM resources**  
   - CPU cores, RAM, storage per VM  
   - Consider performance vs host limitations  

4. **Plan host networking**  
   - Map Proxmox virtual NICs to physical interfaces and VLANs  
   - Decide whether to use bridged or NAT networking per VM  

5. **Document VM inventory**  
   | VM Name | Purpose | VLAN / Network | CPU | RAM | Storage |
   |---------|--------|----------------|-----|-----|--------|
   | Kali Linux | Pen testing | Lab VLAN 30 | 2 cores | 4 GB | 40 GB |
   | OWASP Juice Shop | Vulnerable app | Lab VLAN 30 | 2 cores | 2 GB | 20 GB |
   | Wazuh Server | Monitoring | Management VLAN 10 | 2 cores | 4 GB | 50 GB |

6. **Plan verification steps**  
   - VM should receive an IP from the planned subnet  
   - VM can reach intended networks based on firewall rules  
   - Resource allocation matches host capacity  

## ✅ Verification
- Confirm all planned VMs fit within host hardware resources  
- Check that network assignments align with IP scheme and VLAN plan  
- Validate that each VM’s purpose is clear and matches your lab objectives  

## ✍️ Lessons Learned
- Planning VM roles early prevents overprovisioning or misplacement  
- Assigning network segments in advance avoids misconfigurations later  
- Documenting resources and roles ensures reproducibility and easier scaling

## ➡️ Next Up

Navigate to next section: [05 - Network Segmentation](05-network_segmentation.md)
