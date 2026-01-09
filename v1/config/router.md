# Router Configurations


## Setting up a Firewall
### Prep
1. Access the router's admin panel
2. In the left pane, select the **System** dropdown
3. Click **Advanced Settings**
---
### Install LuCI
1. Press the blue button **Install LuCI**
> &nbsp;&nbsp;&nbsp;Well that was easy
---
### LuCI Setup
1. Click **Go to LuCI**
> &nbsp;&nbsp;&nbsp;You should now be in a new window
2. Log in to the LuCI admin panel
> &nbsp;&nbsp;&nbsp;User is **root** and password is the same as your router's password
---
### Baseline Firewall Config
1. In the top menu, click the **Network** dropdown
2. Click **Firewall**
3. Enable **SYN-flood protection**
4. Enable **Drop invalid packets**
> &nbsp;&nbsp;&nbsp;This helps block malformed or suspicious packets that don't follow the proper TCP/IP rules, often used in scans or attacks
5. Scroll down to **Zones**
6. In the LAN -> forwarding zone set all three to accept

&nbsp;&nbsp;&nbsp;`Input = Accept`

&nbsp;&nbsp;&nbsp;`Output = Accept`

&nbsp;&nbsp;&nbsp;`Forward = Accept`

7. In the WAN -> forwarding zone set

&nbsp;&nbsp;&nbsp;`Input = Drop`

&nbsp;&nbsp;&nbsp;`Output = Accept`

&nbsp;&nbsp;&nbsp;`Forward = Drop`
