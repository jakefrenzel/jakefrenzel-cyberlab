# Netgear Switch Configuration

## Quick Access

[Return to Network Architecture Implementation](../setup/implementation/01-network_architecture_implementation.md)

## Access Switch Management Dashboard

1. Access the web GUI from the default IP

```netgear-web-gui
192.168.0.239
```

## Set Secure Admin Password

1. Log in with the default password: `password`

2. Create a strong password up to 20 characters long (I recommend to use all 20 characters)

3. Log in with admin credentials

## Update Firmware
```netgear-web-gui
Settings -> Firmware -> Check for updates
```

My Firmware was an entire major version behind

1. Download latest firmware

2. Unzip the download

3. Update

## Enable Advanced 802.1Q VLAN
```netgear-web-gui
Switching -> VLAN
```
Advanced 802.1Q VLAN
1. Activate mode: `Advanced 802.1Q VLAN`

2. Create a new VLAN: `Add VLAN`

3. Name it corresponding to the pfSense VLANs: `Management`

4. Assign the matching VLAN ID: `10`

5. Select whether each VLAN should be tagged, untagged, or excluded

6. Repeat for all VLANs

| VLAN | PVID | Port | Membership |
|------|------|------|------------|
| Management | 10 | 1,2,3,4,5,6,7,8 | Tagged |
| 

## Edit PVID Table

1. Click view PVID table

2. For each port select the PVID for each port. (e.g. port 2 proxmox server has PVID of 30, the servers VLAN)
