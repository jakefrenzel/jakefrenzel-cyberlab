# 🖥️ Ubuntu Server — Virtual Machine Setup

This guide walks through downloading Ubuntu Server, creating a VM in Proxmox, and booting the server.

---

## ⚙️ Setup

1. Download the latest **Ubuntu Server 24.04.3 LTS** from [ubuntu.com/download/server](https://ubuntu.com/download/server).

2. Log in to the **Proxmox web GUI** and click on your storage drive in the left explorer panel.

3. Click **ISO Images** and upload the Ubuntu ISO.

4. Select the uploaded ISO and press **Upload**.

---

## 🛠️ Create VM

1. Click **Create VM** (top right corner).

2. Name the VM: `UbuntuServer`.

3. Go to the **OS** tab.

4. Select the proper storage (e.g., `mydata`).

5. Choose the **ISO Image** you uploaded.

6. Go to the **System** tab and toggle **Qemu Agent**.

7. Move to the **Disks** tab and set the disk size (e.g., **32GB**).

8. Go to the **CPU** tab and allocate **2 Cores**.

9. Go to the **Memory** tab and set **4096 MB**.

10. Click **Next** and then **Finish** to create the VM.

---

## 🚀 Boot Ubuntu Server

1. Select the new VM in the left pane and click **Start**.

2. Click the **Console** option to access the VM display.

3. Keep all default settings and click **Next** through the installation prompts.

4. Enter your **Name** and **Server Name**: `ubuntuserver`.

5. Choose a **Username**: `jake`.

6. Enter a secure **Password**.

7. Complete the remaining installation prompts.

8. Remove the ISO:  
   - Go to **Hardware** → double-click **CD/DVD Drive** → select **Do not use any media**.
  
  
10. **Reboot** the VM.

---

## 💡 Tips

- Keep track of the username and password for SSH access later.  
- Double-check the disk and memory allocation to match your system resources.  
- After reboot, verify network connectivity before proceeding with further setup.
