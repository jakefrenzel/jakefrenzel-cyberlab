## OWASP Juice Shop
### Setting Up The Host Virtual Machine - Ubuntu Server
1. Download the latest version: Ubuntu 24.04.3 LTS from [ubuntu.com/download/server](https://ubuntu.com/download/server/)
2. Navigate to your virtualization platform
3. Open the Ubuntu ISO image and upload
> &nbsp;&nbsp;&nbsp;The goal is to have the Ubuntu Server ready to boot up, I use proxmox which I will explain in the next section

### Proxmox Setup - Ubuntu Server
1. Press the create VM in the top right
2. Give it a name: **owasp-juice-shop**
3. Next go to **OS**
4. Select the proper storage: **mydata**
5. Select the ISO Image you uploaded: **Ubuntu Server**
6. Click next to the **System** tab
7. Toggle **Qemu Agent:** Enable
8. Click next to **Disks**
9. Change the disk size here if you need, the reccomendation is: **20GB**.
10. Click next to **CPU**
11. Change the amount of CPU cores to use, I will select **2 Cores**
12. Click next to **Memory**
13. Change the amount of memory to **4096**
14. Click **Next** and **Finish**

### Boot & Configure Ubuntu Server
1. Click the new VM and press **Start**
2. Press the **Console** option to display the VM's screen
3. Keep all the default settings (Click next)
4. Enter your name and server name: **owaspjs**
5. Pick a username: **jake**
6. Type in a secure password
7. Click **Next** through the rest
8. Remove the ISO by clicking **Hardware**, double clicking **CD/DVD Drive**, and then select **Do not use any media**
9. Reboot
