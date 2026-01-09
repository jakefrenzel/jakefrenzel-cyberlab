# 🛡️ VPN Setup — WireGuard Client (NordVPN Example)

This guide walks you through setting up a **WireGuard client** for your VPN service (using **NordVPN** as an example).

---

## ✅ Prerequisites

Before starting, make sure you have:

- Access to your **device’s admin panel**
- A **VPN provider** that supports WireGuard (e.g., NordVPN)
- A **password manager** for storing your access token securely

---

## ⚙️ Setup Instructions

1. **Access the admin panel.**  
   Log into your router’s web interface or admin panel.

2. **Navigate to the VPN section.**

3. **Select _WireGuard Client_** from the dropdown menu.

4. **Choose your VPN provider.**  
   For example: `NordVPN`.

5. **Log in to your VPN provider account** in a new browser tab.

6. **Generate an access token.**  
   - In your NordVPN dashboard, select **NordVPN** from the sidebar.  
   - Scroll down to **Get Access Token** and click it.  
   - Copy the token and save it securely in your password manager.

7. **Return to your admin panel.**  
   Paste the access token into the **Access Token** input field and click **Submit**.

8. **Select your preferred VPN server(s)** from the available list.

9. **Save or apply your settings.**

10. **Enable your VPN connection.**  
    Go to the VPN dashboard and toggle the switch to **ON**.

---

## 🔍 Verification

To confirm your VPN is active, run:

```bash
curl https://ipinfo.io
```

You should see your **VPN server’s IP address**, not your local one.
