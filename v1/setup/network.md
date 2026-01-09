# 🌐 Network Setup

This guide outlines how to configure your **Beryl AX Router** and **NETGEAR Switch**, then assign LAN IP addresses for your local network.

---

## 🛰 Beryl AX Router Setup

1. Connect to the **Beryl AX Router** LAN port using an Ethernet cable.
   
2. Access the admin panel via [http://192.168.8.1/](http://192.168.8.1/) in your browser.
   
3. Set a **strong and secure password** for the router’s admin panel.
   
4. Set **secure passwords** for both the **2.4 GHz** and **5 GHz** Wi-Fi networks.
   
5. Log back into the admin panel using the password you just created.

---

## 🖧 NETGEAR Switch Setup

1. Open a browser and navigate to [http://192.168.8.249/](http://192.168.8.249/).  
   > 💡 **Note:** If your switch is not using the default IP, open your router’s admin panel, click on **Clients**, and find your switch (model **GS308EP**) to locate its assigned IP address.

2. Enter the **default password**: `password`
   
3. When prompted, set a **new secure admin password**.
   
4. Log back into the Netgear admin panel using your new password.

---

## ⚙️ Configure LAN IP Addresses

### 🧩 Switch Configuration

1. Log in to the switch admin panel.
   
2. From the home page, open the **IP Address** dropdown.
   
3. **Disable DHCP.**
   
4. Change the **IP Address** to `192.168.10.2`.
   
5. Change the **Gateway** to `192.168.10.1` (this will be your router’s IP).
    
6. The admin panel will become temporarily inaccessible until the router is reconfigured.

---

### 📡 Router Configuration

1. Log in to the router admin panel.
   
2. In the sidebar, open the **Network** dropdown.
   
3. Select the **LAN** option.
   
4. Change the **Router IP Address** to `192.168.10.1`.
   
5. Apply the settings — the router will reboot automatically.
    
6. Navigate to **DHCP Server Settings** (scroll down on the same page).
    
7. Set the **DHCP IP Range**:
   - **Start:** `192.168.10.100`
   - **End:** `192.168.10.200`

---

## ✅ Summary

| Device        | IP Address     | Gateway        | DHCP |
|----------------|----------------|----------------|-------|
| Router (Beryl AX) | `192.168.10.1` | —              | Enabled |
| Switch (NETGEAR)  | `192.168.10.2` | `192.168.10.1` | Disabled |

---

## 💡 Tips

- Keep your router and switch admin credentials stored securely.
- Avoid using easily guessed IPs or default passwords.
- After configuration, label your devices with their assigned IPs for easier troubleshooting.
