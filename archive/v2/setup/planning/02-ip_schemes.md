# 📐 02 - IP Schemes

## 🎯 Objective
Plan IP addressing and subnetting for each network segment in your lab. This ensures a clean, organized, and manageable network before implementation.

## 📝 Prerequisites
- Completed Network Architecture planning (`01 - Network Architecture.md`)  
- Basic understanding of IP addressing and subnetting  
- Access to network diagrams or topology references  

## ⚙️ Steps

1. **Identify network segments**  
   - Management network  
   - User network  
   - Lab / test network  

2. **Assign VLAN IDs**  
   Example:  
   | Network | VLAN |
   |---------|-----|
   | Management | 10 |
   | User | 20 |
   | Lab | 30 |

3. **Plan IP ranges for each segment**  
   Example subnets:  
   | Network | VLAN | IP Range |
   |---------|-----|----------|
   | Management | 10 | 192.168.10.0/24 |
   | User | 20 | 192.168.20.0/24 |
   | Lab | 30 | 192.168.30.0/24 |

4. **Decide device IP assignments**  
   - Which devices will be static (firewall, Proxmox host, switches)  
   - Dynamic IPs / DHCP ranges for lab devices  

5. **Document IPs in a table**  
   - Include device name, VLAN, assigned IP, and purpose  

6. **Plan connectivity verification**  
   - Define which devices should be able to communicate  
   - Plan tests for intra-VLAN and inter-VLAN connectivity  

## ✅ Verification
- Confirm that planned subnets do not overlap  
- Check that VLAN IDs match intended network segments  
- Ensure there is a clear mapping of which devices get which IPs  
- Document verification steps in your notes so you can follow them during implementation  

## ✍️ Lessons Learned
- Planning IPs and VLANs early prevents conflicts and confusion later  
- Keeping a table of IP assignments helps with troubleshooting  
- Aligning IP schemes with the network architecture ensures consistency  

## ➡️ Next Up

Navigate to next section: [03 - Firewall Setup](03-firewall_setup.md)
