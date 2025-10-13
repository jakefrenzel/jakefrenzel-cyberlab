### Wazuh Setup
#1. Log in to your proxmox web GUI
#2. Open the shell
#3. Enter the command:
`cd /var/lib/vz`
#4. Make a new directory called wazuh and change directory into it
`mkdir wazuh && cd wazuh`
#5. Go to installation alternatives in the Wazuh documentation on their website (https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html)
#6. Copy the link to the download for the lastest version (https://packages.wazuh.com/4.x/vm/wazuh-4.13.1.ova)
#7. Execute the command in the Proxmox shell
`wget https://packages.wazuh.com/4.x/vm/wazuh-4.13.1.ova`
#8. When it completes, extract it
`tar -xvf wazuh-4.13.1.ova`

### Setting up the Wazuh VM
#1. In the top right, click the blue 'Create VM' button
#2. Set the name to wazuh and the VM ID to 200 (or anything else but remember this)
#3. Click next to the OS tab
#4. Select do not use any media
#5. Click next to the System tab
#6. Change the machine option to use q35
#7. Toggle Qemu Agent on
#8. Click next to the Disks tab
#9. Delete the default disk
#10. Click next to CPU
#11. Use the recommended amount of cores according to the Wazuh docs (4 cores)
#12. Use recommended amount of RAM as well (8gb of ram or 8192 MiB)
#13. Click next and finish

### Importing the disk to the VM
#1. Navigate to the Proxmox shell
#2. Change directory into the location of the ova we extracted
`cd /var/lib/vz/wazuh`
#3. We then need to import the disk (200 indicates the VM ID)
`qm importdisk 200 wazuh-4.13.1-disk-1.vmdk local-lvm --format raw`
#4. Click on the wazuh VM in the left pane
#5.  Click hardware
#6. Double click on the unused disk near the bottom
#7. Click add to add it to the VM

### Update the boot order
#1. Click on the wazuh VM in the left pane
#2. Navigate to the 'Options' tab
#3. Double click on boot order
#4. Uncheck all the default options
#5. Select the third option with our vm disk
#6. Press ok

### Boot into the VM
#1. Go to console and start up the VM
#2. Enter the credentials to access the Wazuh server
#3. The default user is 'wazuh-user' and the pass is 'wazuh'
#4. Find out what IP address the server is running on with the command:
`ip a`
#5. Look for the Inet IPv4 address and use that IP to access the dashboard in the browser
#6. Now log in to the Wazuh admin portal with the default credentials (user: admin) (pass: admin)
#7. You are now in!

### Configure the Wazuh Server to a Static IP
#1. Enter the Wazuh server's console
#2. Enter this command to change the network settings
`cd /etc/sysconfig/network-scripts/`
#3. Edit the file
`sudo nano ifcfg-eth0`
#4. Change 'BOOTPROTO' to static rather than DHCP
`BOOTPROTO=static`
#5. Add a new configuration line for IP address
#6. Add whatever IP address you want to use
`IPADDR=192.168.10.30`
#7. Add a new line for the subnet mask
`NETMASK=255.255.255.0`
#8. Add a gateway (our router)
`GATEWAY=192.168.10.1`
#9. Save it with Ctrl + X
#10. Press 'Y' to save the config and press enter
#11. Restart the service
`sudo systemctl restart network.service`
#12. You can now access the admin dashboard from the new IP address

### The most important wazuh setting
#1. Log into the Wazuh admin dashboard
#2. Click the menu icon in the top left
#3. Navigate to the 'Dashboard Management' dropdown
#4. Click 'Dashboard Management'
#5. Click advanced settings in the left pane
#6. Scroll to the appearance section
#7. Toggle dark mode ON
#8. Save changes and reload page
