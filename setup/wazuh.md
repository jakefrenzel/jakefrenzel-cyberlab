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
#9.
