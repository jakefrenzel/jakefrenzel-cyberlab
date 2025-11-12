# 🧃 OWASP Juice Shop Setup (Ubuntu Server + Docker)

This guide explains how to deploy **OWASP Juice Shop** on an **Ubuntu Server** virtual machine using **Docker**.  
If you already have an Ubuntu Server running, skip to the **Docker Installation & Startup** section.

---

## 🧭 Overview

- [Setting Up the Host Virtual Machine (Ubuntu Server)](#setting-up-the-host-virtual-machine---ubuntu-server)
- [Proxmox Setup — Ubuntu Server](#proxmox-setup---ubuntu-server)
- [Boot & Configure Ubuntu Server](#boot--configure-ubuntu-server)
- [Installation & Startup (Docker)](#installation--startup)
- [Optional: Auto Startup Configuration](#optional-auto-startup-docker)

---
### Setting Up The Host Virtual Machine - Ubuntu Server
1. Download the latest version: Ubuntu 24.04.3 LTS from [ubuntu.com/download/server](https://ubuntu.com/download/server/)
   
3. Navigate to your virtualization platform
   
5. Open the Ubuntu ISO image and upload
   
> &nbsp;&nbsp;&nbsp;The goal is to have the Ubuntu Server ready to boot up, I use proxmox which I will explain in the next section
---
### Proxmox Setup - Ubuntu Server
1. Press the create VM in the top right
   
3. Give it a name: **owasp-juice-shop**
   
5. Next go to **OS**
   
7. Select the proper storage: **mydata**
   
9. Select the ISO Image you uploaded: **Ubuntu Server**
    
11. Click next to the **System** tab
    
13. Toggle **Qemu Agent:** Enable
    
15. Click next to **Disks**
    
16. Change the disk size here if you need, the reccomendation is: **20GB**.
    
18. Click next to **CPU**
    
20. Change the amount of CPU cores to use, I will select **2 Cores**
    
22. Click next to **Memory**
    
24. Change the amount of memory to **4096**
    
26. Click **Next** and **Finish**
---
### Boot & Configure Ubuntu Server
1. Click the new VM and press **Start**
   
3. Press the **Console** option to display the VM's screen
   
5. Keep all the default settings (Click next)
   
7. Enter your name and server name: **owaspjs**
   
9. Pick a username: **jake**
    
11. Type in a secure password
    
13. Click **Next** through the rest
    
15. Remove the ISO by clicking **Hardware**, double clicking **CD/DVD Drive**, and then select **Do not use any media**
    
17. Reboot
---
### Installation & Start Up
1. Log into the Ubuntu Server
   
3. Enter the command (make sure the dir exists so no errors occur)
```text
sudo mkdir -p /etc/apt/keyrings
```
3. Then update
```text
sudo apt update
```
4. Install Docker for the Ubuntu server
```text
sudo apt install docker.io -y
```
5. Enable and start up Docker
```text
sudo systemctl enable docker
sudo systemctl start docker
```
6. To verify the installation and startup was successful
```text
sudo docker run hello-world
```
OPTIONAL: Enter the following command to use docker without having to type sudo repeatedly
```text
sudo usermod -aG docker $USER
newgrp docker
```
7. Test with the following command:
```text
docker ps
```
8. Run the OWASP Juice Shop!
```text
docker pull bkimminich/juice-shop
docker run -d -p 3000:3000 --name juice-shop bkimminich/juice-shop
```
9. Access OWASP Juice Shop in the browser
```text
http://<your_vm_ip>:3000
```
### Optional: Auto Startup Docker
1. Remove the previous container so we can create an auto-restart container
```text
docker stop juice-shop
docker rm juice-shop
```
2. Create the new container with the restart flag
```text
docker run -d \
  --name juice-shop \
  --restart unless-stopped \
  -p 3000:3000 \
  bkimminich/juice-shop
```
This will allow the container to start up each time the Ubuntu Server is turned on or rebooted
