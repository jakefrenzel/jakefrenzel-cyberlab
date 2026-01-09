# 🧪 07 - Labs & Exercises

## 🎯 Objective
Plan the hands-on cybersecurity exercises you want to run in this lab, including vulnerable applications, attack paths, detection methods, and learning goals.

## 📝 Prerequisites
- Completed network design and segmentation planning  
- Firewall rules and access restrictions planned  
- VM roles and network placement defined  
- Hardening and monitoring strategy planned  

## ⚙️ Steps

1. **Define learning goals**
   - Practice offensive techniques (e.g., web exploitation, privilege escalation)
   - Practice defensive techniques (e.g., log analysis, detecting attacks)
   - Understand traffic flow across VLANs
   - Build repeatable lab workflows

2. **Identify target systems & vulnerable apps**
   - OWASP Juice Shop (web app exploitation)
   - Metasploitable2 or DVWA (optional)
   - Custom Linux VM for SSH/priv-esc practice
   - Windows VM for forensics / malware analysis (optional)

3. **Map each lab to a network segment**
   - Place vulnerable apps in **Lab VLAN**
   - Place attacker VMs (Kali) in **Attacker VLAN**
   - Ensure management VLAN remains isolated

4. **Plan attack paths**
   Example:
   - Kali → Lab VLAN → OWASP Juice Shop  
   - Kali → Metasploitable → Escalate Privileges  
   - Kali → Scan → Identify misconfigurations  
   - Observe logs/security alerts based on your monitoring setup  

5. **Plan defensive exercises**
   - Log review in Wazuh or syslog  
   - Packet capture in Proxmox, Kali, or a monitoring VM  
   - Firewall rule testing  
   - Alert tuning (FAILED logins, port scans, suspicious traffic)

6. **Plan documentation for each exercise**
   Every lab exercise should include:
   - **Objective**  
   - **Tools used**  
   - **Step-by-step walkthrough**  
   - **Expected outcomes**  
   - **Screenshots or diagrams**  
   - **What you learned**

7. **Plan verification**
   - Confirm attacker VM can access only what it should  
   - Confirm monitoring detects your tests  
   - Validate vulnerable apps function as intended  
   - Ensure segmentation prevents cross-VLAN pivoting unless intentionally allowed  

## ✅ Verification
- All planned labs align with network segmentation  
- Attack paths match firewall rules and intended access  
- Monitoring plan covers expected log sources  
- Exercises are reproducible for others reading your repo

## ✍️ Lessons Learned
- Planning labs ahead ensures every VM and network segment serves a purpose  
- Knowing attack paths in advance improves firewall and logging decisions  
- Documenting labs makes the homelab valuable for your resume and GitHub
