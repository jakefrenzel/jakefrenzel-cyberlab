## Proxmox Virtualization Setup
#1. Download proxmox iso<br/>
#2. Use a tool such as Rufus to load the iso into a bootable removable USB drive<br/>
#3. Insert the drive to your server / device that will be used for the proxmox server (I am using a laptop)<br/>
#4. Enter the laptop bios and boot from the removable USB<br/>
#5. Install proxmox and go through the setup<br/>
#6. Set the static IP for the proxmox server (I am setting mine to 192.168.10.20)<br/>
#7. Open a browser on another device and access the web gui via the IP address set in the last step (192.168.10.20)<br/>
#8. In the explorer on the left, click on your proxmox server<br/>
#9. Open the dropdown for 'Updates' and select the 'Repositories' option.<br/>
#10. Select the repositories that say 'Enterprise' as we do not want these since we do not have a key<br/>
#11. Add a new repository, select the 'No-Subscription' option<br/>
#12. Click 'Updates', press the 'Refresh' button, wait for it to say OK, and then press the 'Upgrade' button<br/>

### Disable Subscription Popup
#1. Open proxmox shell
#2. Type in the following commands to create a backup
`cd /usr/share/javascript/proxmox-widget-toolkit`
`cp proxmoxlib.js proxmoxlib.js.bak`
`nano proxmoxlib.js`
#3. Press Ctrl + W
#4. Search for 'no valid sub'
#5. Find the line that says
`let res = response.result;`
#6. Delete it as well as the next 16 lines
#7. The next remaining line should be orig_cmd(); (you will have deleted one of them)
#8. Also delete the curly bracket on the next line to fix the code
