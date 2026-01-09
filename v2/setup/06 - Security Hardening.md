# 🛡️ 06 - Security Hardening

## 🎯 Objective
Plan the security measures for your lab to protect your network and systems. This includes firewall hardening, access controls, monitoring, and logging.

## 📝 Prerequisites
- Completed Network Architecture (`01 - Network Architecture.md`)  
- Planned IP Schemes (`02 - IP Schemes.md`)  
- Firewall Setup plan (`03 - Firewall Setup.md`)  
- Virtual Machines plan (`04 - Virtual Machines.md`)  
- Network Segmentation plan (`05 - Network Segmentation.md`)  

## ⚙️ Steps

1. **Plan firewall hardening**  
   - Restrict unused interfaces and ports  
   - Disable unnecessary services  
   - Ensure default deny rules for inbound traffic  

2. **Plan monitoring and logging**  
   - Decide which devices and VLANs need logging  
   - Identify tools to capture logs (e.g., Wazuh, syslog, SNMP monitoring)  
   - Plan alerting for suspicious events  

3. **Plan access control**  
   - Define admin accounts for firewall, switches, and Proxmox  
   - Plan password policies (length, complexity, rotation)  
   - Consider SSH key usage for management access  

4. **Plan host and VM hardening**  
   - Disable unnecessary services on lab VMs  
   - Apply updates/patches (planning step: note the procedure)  
   - Decide on baseline configurations for security  

5. **Document hardening plan**  
   - Include all firewall rules, accounts, and monitoring configurations in tables or notes  
   - Reference prior planning steps to ensure alignment with VLANs and IP schemes  

6. **Plan verification steps**  
   - Test firewall rules in simulation or planning diagrams  
   - Confirm access control policies cover all administrative interfaces  
   - Ensure monitoring plan can capture and alert on relevant events  

## ✅ Verification
- Confirm planned hardening steps align with network design  
- Ensure no critical access is blocked for management tasks  
- Validate that monitoring and logging cover intended networks and devices  

## ✍️ Lessons Learned
- Planning hardening early avoids security gaps when implementing  
- Mapping security controls to VLANs and IPs improves network safety  
- Documenting procedures makes implementation easier and more reproducible
