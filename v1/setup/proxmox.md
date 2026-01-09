# 💻 Proxmox Virtualization Setup

This guide walks through installing and configuring **Proxmox VE** on a server, setting a static IP, updating repositories, and disabling the subscription popup.

---

## ⚙️ Installation Steps
1. Download the latest **Proxmox ISO** from [proxmox.com](https://www.proxmox.com/en/downloads).
   
2. Use a tool such as **Rufus** to write the ISO to a **bootable USB drive**.
   
3. Insert the USB drive into the server or device that will run Proxmox.
   
4. Enter the BIOS and **boot from the USB drive**.
   
5. Install Proxmox and complete the setup wizard.
    
6. Set a **static IP** for the Proxmox server (e.g., `192.168.10.20`).
    
7. From another device, open a browser and navigate to the Proxmox web GUI using the IP address set above (e.g., `https://192.168.10.20:8006`).
    
8. In the left panel, click your Proxmox server.
    
9. Open the **Updates** dropdown and select **Repositories**.
    
10. Disable any repositories labeled **Enterprise** (no subscription key available).
    
11. Add a new repository and select the **No-Subscription** option.
    
12. Go to **Updates**, press **Refresh**, wait for it to complete, then press **Upgrade**.

---

## 🚫 Disable Subscription Popup

To remove the “No valid subscription” popup in the Proxmox web GUI:

1. Open the **Proxmox shell**.
   
2. Backup the JavaScript file and open it for editing:
 ```bash
 cd /usr/share/javascript/proxmox-widget-toolkit
 cp proxmoxlib.js proxmoxlib.js.bak
 nano proxmoxlib.js
 ```

3. Press Ctrl + F
   
4. Search for `no valid sub`
   
5. Locate the line that says
`let res = response.result;`

6. Delete it as well as the next 16 lines or more
    
7. The next remaining line should be orig_cmd(); (you will have deleted one of them)
    
8. Also delete the curly bracket on the next line to fix the code
    
9. Exit with Ctrl + X and make sure to press 'Y' to save
    
10. Restart with the following command:
`systemctl restart pveproxy.service`

## 💡 Tips

- Always backup files before editing system scripts.
- Use No-Subscription repositories if you do not have an enterprise license.
- After disabling the popup, refresh the Proxmox web GUI to verify changes.
- Keep your Proxmox server IP noted for future access and configurations.
