# 🔒 Firewall Setup — Implementation (pfSense)

This document outlines the actual firewall configuration implemented for
CyberLab v2 using **pfSense**, including interface assignments, VLAN routing,
and baseline security rules.

The goal of this setup is to provide strong network segmentation while
maintaining controlled access for management and monitoring.

---

## ⚙️ Initial pfSense Setup

### Access pfSense Web GUI

1. Open a browser and go to the default IP address of the pfSense firewall: ```192.168.1.1```

2. Sign in with default credentials:
    - User: admin
    - Pass: pfsense
 
3. Go to step 2 in the pfSense setup and add DNS primary DNS server: ```1.1.1.1```

4. Add a secondary DNS:
```8.8.8.8```

5. Uncheck overide DNS

<br>
Optional: Change the hostname and domain to fit your network

---

## DHCP

### Enable DHCP

```pfsense
Services -> DHCP Server
```

1. ☑ Enable DHCP server on LAN interface

### Primary Address Pool

1. Set address pool range from
```10.27.10.100``` to ```10.27.10.199``` 
