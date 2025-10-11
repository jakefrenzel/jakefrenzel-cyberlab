# Network Setup
## Beryl AX Router Setup
#1. Connect to the Beryl AX Router LAN port with laptop via ethernet cable<br/>
#2. Access the admin panel via http://192.168.8.1/ in the broswer<br/>
#3. Set a very secure password for the router's admin panel<br/>
#4. Set very secure passwords for the 2.4 GHz and 5 GHz Wi-Fi channels<br/>
#5. Access the admin panel using the password set previously<br/>

## NETGEAR Switch Setup
#1. Open a browser and connect to http://192.168.8.249/
> It is possible your Netgear switch did not set to the default. In this case, open your router's admin panel, click on 'clients' in the sidebar, and look for the Netgear switch (GS308EP) and use the IP that it displays.

#2. Type in the default password which is 'password'<br/>
#3. It will then prompt you to enter a new admin password, also set a very secure password for this admin<br/>
#4. Log in to Netgear admin with the new secure password<br/>

## Configure LAN IP Addresses
### Switch
#1. Log in to the admin panel<br/>
#2. Click on the IP address dropdown on the home page<br/>
#3. Disable DHCP<br/>
#4. Change the IP address to 192.168.10.2<br/>
#5. Change the gateway to 192.168.10.1 (this is what our router's address will be)<br/>
#6. The admin panel will now fail to load until we set up our router<br/>


### Router
#1. Log in to the admin panel<br/>
#2. Open the network dropdown in the sidebar<br/>
#3. Select the option labeled 'LAN'<br/>
#4. Change the router IP address to 198.168.10.1<br/>
#5. Apply settings and the will reboot<br/>
#6. Navigate to the DHCP server settings (scroll down in the same page as before)<br/>
#7. Change the DHCP server IP range to start at 192.168.10.100 and end at 192.168.10.200<br/>
