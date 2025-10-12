## Ubuntu Server - Virtual Machine
### Setup
#1. Download the latest version; Ubuntu 24.04.3 LTS from https://ubuntu.com/download/server<br/>
#2. Log in to the Proxmox web GUI and click on your drive in the explorer on the left side<br/>
#3. Click ISO Images and upload<br/>
#4. Select the ISO that you downloaded and press upload<br/>

### Create VM
#1. Press the create VM in the top right<br/>
#2. Give it a name (UbuntuServer)<br/>
#3. Next go to OS<br/>
#4. Select the proper storage (mydata)<br/>
#5. Select the ISO Image you uploaded<br/>
#6. Click next to system<br/>
#7. Toggle Qemu Agent<br/>
#8. Click next to disks<br/>
#9. Change the disk size here if you need, I am leaving mine at 32GB for this one.<br/>
#10. Click next to CPU<br/>
#11. Change the amount of CPU cores to use, I will select 2<br/>
#12. Click next to memory<br/>
#13. Change the amount of memory to 4096<br/>
#14. Click next and finish<br/>

### Boot Ubuntu Server
#1. Click the new VM in the left pane and press start<br/>
#2. Press the 'console' option to display the VM<br/>
#3. Keep all the default settings and click next through them<br/>
#4. Enter your name and server name (ubuntuserver)<br/>
#5. Pick a username (jake)<br/>
#6. Type in a password<br/>
#7. Click through the rest<br/>
