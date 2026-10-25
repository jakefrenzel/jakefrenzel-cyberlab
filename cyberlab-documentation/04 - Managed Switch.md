# 🔀 04 - Managed Switch

## Objective
Configure the Netgear switch to carry all seven VLANs between pfSense and Proxmox, give each device the right access port, mirror the pfSense trunk to the IDS sensor, and keep the switch's own management on VLAN 10.

## Hardware
| Device | Purpose |
|--------|---------|
| Netgear GS308EP | 8-port managed (Smart Managed Plus) PoE switch |
| Dell Laptop | Admin PC, on port 8 |

## Software
| Item | Value |
|------|-------|
| Firmware | V2.0.0.11 |
| Management IP | `10.27.10.2` / `24`, gateway `10.27.10.1` (static) |
| Management VLAN | 10 |
| VLAN mode | Advanced 802.1Q VLAN |

## Port Map
| Port | Name | Connected to | Mode | VLANs | PVID |
|------|------|--------------|------|-------|------|
| 1 | `pfsense` | pfSense `igc1` | Trunk | Tagged 10 – 70, untagged 700 | 700 |
| 2 | `proxmox` | Proxmox host | Trunk | Tagged 10 – 70, untagged 700 | 700 |
| 3 | `sensor-capture` | Pi `eth0` (capture) | Mirror destination | Untagged 10 | 10 |
| 4 | `sensor-mgmt` | Pi USB NIC (management) | Access | Untagged 10 | 10 |
| 5 – 7 | — | Unused | Parked | Untagged 1 | 1 |
| 8 | `admin` | Admin PC | Access | Untagged 10 | 10 |

## VLAN Membership
`T` = tagged, `U` = untagged, `–` = not a member

| VLAN | Name | P1 | P2 | P3 | P4 | P5 | P6 | P7 | P8 |
|------|------|----|----|----|----|----|----|----|----|
| 1 | Default | – | – | – | – | U | U | U | – |
| 10 | Management | T | T | U | U | – | – | – | U |
| 20 | Workstations | T | T | – | – | – | – | – | – |
| 30 | Servers | T | T | – | – | – | – | – | – |
| 40 | Lab | T | T | – | – | – | – | – | – |
| 50 | DMZ | T | T | – | – | – | – | – | – |
| 60 | Guest | T | T | – | – | – | – | – | – |
| 70 | Monitoring | T | T | – | – | – | – | – | – |
| 700 | void | U | U | – | – | – | – | – | – |

> [!NOTE]
> **VLAN 700 is a black hole.** It is the native (untagged) VLAN on both trunks and exists nowhere else, so any untagged frame on a trunk goes nowhere. Real traffic is always tagged. This keeps a misconfigured device, or the pfSense bootstrap LAN, from landing in a real network. Unused ports 5 – 7 are parked in VLAN 1 for the same reason: VLAN 1 is not carried to pfSense, so a device plugged into them reaches nothing.

---

## Steps taken

### First login
1. Connected the switch to pfSense (port 1) while the pfSense bootstrap LAN was still active, so the switch received a DHCP address. Without DHCP, the switch falls back to `192.168.0.239`
2. Found the switch's address in pfSense under `Status -> DHCP Leases`, and browsed to it from the Admin PC
3. Logged in with the default password printed on the label under the switch
4. Set a long, unique admin password:
```netgear-web-gui
Settings -> Change Password
```

### Update the firmware
```netgear-web-gui
Settings -> Firmware
```

1. Checked for and installed the latest firmware before configuring anything else

### Enable Advanced 802.1Q VLANs
```netgear-web-gui
Switching -> VLAN
```

1. Clicked **Activate Mode** under **Advanced 802.1Q VLAN**
2. Clicked **Add VLAN** for each network, using the VLAN IDs and names from [02 - IP Schemes](<02 - IP Schemes.md>): `10` Management through `70` Monitoring
3. Edited each VLAN and set every port to `T`, `U`, or `E` (exclude) to match the [VLAN Membership](#vlan-membership) table

> [!IMPORTANT]
> Don't build VLAN 700 or change the PVIDs on ports 1 and 2 yet. The Admin PC is still reaching the switch through the untagged bootstrap LAN, and that path ends the moment the trunks get their black-hole PVID. Move management to VLAN 10 first.

### Move management to VLAN 10
Order matters here. Every step keeps a working path to the switch until the next one exists.

1. Set **port 8**'s PVID to `10` (open the **PVID Table** link on the VLAN page and edit port 8)
2. Moved the Admin PC to **port 8** and gave it a static IP: `10.27.10.200`, mask `255.255.255.0`, gateway `10.27.10.1`. Management has no DHCP
3. Set the switch's own IP address on the **Home** page, under **IP Address**:

| Setting | Value |
|---------|-------|
| DHCP | Disabled |
| IP Address | `10.27.10.2` |
| Subnet Mask | `255.255.255.0` |
| Gateway | `10.27.10.1` |

4. Browsed to `http://10.27.10.2` from the Admin PC on port 8
5. Set **Management VLAN ID** to `10` at the bottom of the VLAN page and applied

> [!WARNING]
> The **gateway** is easy to skip because the switch still works from the same subnet without it. Without it, the switch cannot answer anything routed to it, such as an admin connecting over Tailscale, and looks dead from everywhere except VLAN 10.

### Make the trunks black-hole untagged traffic
1. Added VLAN `700` named `void`, with ports 1 and 2 set to `U` and every other port `E`
2. In the **PVID Table**, set the PVID of ports 1 and 2 to `700`
3. Edited VLAN 1 and excluded ports 1, 2, 3, 4, and 8, leaving only the unused ports 5 – 7

> [!NOTE]
> A port must be an untagged member of a VLAN before it can use that VLAN as its PVID. Add the port to the VLAN first, then change the PVID.

Port 8 must not stay an untagged member of VLAN 1. If it is untagged in both VLAN 1 and VLAN 10, frames from the parked ports 5 – 7 are sent out of port 8 into the Management network.

### Name the ports
On the **Home** page, under **Port Status**, clicked **Edit** on each port and set the names from the [Port Map](#port-map). Clear names matter most on ports 3 and 4, which both connect to the Pi but do very different jobs.

### Port mirroring
```netgear-web-gui
Diagnostics -> Port Mirroring
```

| Setting | Value |
|---------|-------|
| Port Mirroring | On |
| Source port | `1` (pfSense trunk) |
| Destination port | `3` (Pi capture NIC) |

Mirroring only port 1 covers all inter-VLAN and internet traffic without duplicates. The reasoning, and how to verify the mirror from the sensor, are in [05 - Intrusion Detection System](<05 - Intrusion Detection System.md>).

### Security and time settings
| Setting | Location | Value |
|---------|----------|-------|
| DoS Prevention | `Settings -> DoS Prevention` | On |
| Switch Discovery (UPnP) | `Settings -> Switch Discovery` | **Off**, no reason to advertise the switch on the network |
| Power Saving (EEE) | `Settings -> Power Saving` | Off, avoids link problems with some network cards |
| Time | `Settings -> Time -> SNTP Configuration` | SNTP server `10.27.10.1` (pfSense) |

> [!NOTE]
> The switch's web interface is **HTTP only**, so the password crosses the network unencrypted. Only reach it from VLAN 10 or over Tailscale. pfSense already blocks every other VLAN from the Management network ([03 - pfSense Firewall](<03 - pfSense Firewall.md>)).

### Back up the configuration
```netgear-web-gui
Settings -> Configuration File
```

1. Saved the configuration file to the Admin PC, and keep it with the pfSense backup. Never commit it to this repository

### Remove the pfSense bootstrap LAN
With management on VLAN 10 and the trunks black-holing untagged traffic, nothing uses the untagged bootstrap LAN anymore. Remove it from pfSense as described in [Remove the bootstrap LAN](<03 - pfSense Firewall.md#remove-the-bootstrap-lan>).

---

## ✅ Verification
| Test | Expected |
|------|----------|
| `http://10.27.10.2` from the Admin PC on port 8 | Login page loads |
| `ping 10.27.10.2` from a device on Tailscale | Replies (proves the gateway is set) |
| Kali (VM on VLAN 40) runs `ip -br addr` | Address in `10.27.40.100` – `.199` (proves VLAN 40 crosses ports 2 → 1) |
| `Diagnostics -> Port Statistics` | Port 3 shows **0 bytes received** and a growing **bytes sent** (mirror working, capture NIC silent) |
| Laptop plugged into port 5, 6, or 7 | No address and no connectivity |
| `Diagnostics -> Port Statistics` | **CRC Error Packets** stays at 0 on every port |

---

## Troubleshooting
| Symptom | Cause | Fix |
|---------|-------|-----|
| Switch GUI lost after changing the management VLAN | Admin PC not on a VLAN 10 port | Connect to port 8 with a static `10.27.10.x` address |
| Switch answers from VLAN 10 but not over Tailscale | Gateway missing on the switch | Set gateway `10.27.10.1` on the Home page |
| VM gets no DHCP address | VLAN not tagged on port 2, or wrong VLAN tag on the VM | Check the VLAN Membership table and the VM's network device in Proxmox |
| Can't set a port's PVID | Port is not an untagged member of that VLAN | Add the port to the VLAN as `U` first |
| Everything behind the switch goes dark, but switch lights are normal | Usually the pfSense side of port 1, not the switch | In pfSense, check `Status -> Interfaces -> LAN`. If **In packets** stops increasing while the link is up, reboot pfSense |
| Sensor sees no traffic | Mirroring off, or wrong source/destination | Source `1`, destination `3`, then check from the sensor with `tcpdump` |

---

## Notes & Observations
1. The port names on the switch had ports 3 and 4 swapped compared to the real wiring. The mirror configuration was the source of truth. Labels that don't match reality are worse than no labels
2. A black-hole native VLAN (700) means a forgotten untagged network can't leak into a real one. It also made removing the pfSense bootstrap LAN safe, because nothing could be reaching it through the trunks
3. Moving management to a VLAN is the riskiest step on a remote-managed switch. Always have the next path working before removing the current one
4. When the whole lab went dark one evening, power-cycling the switch did nothing. The fault was pfSense's LAN port no longer receiving, which `Status -> Interfaces` showed immediately. The switch's own Port Statistics (no CRC errors) helped rule out the cable

## ➡️ Next Up
[**05 - Intrusion Detection System:** Deploy Suricata and Zeek on mirrored traffic](<05 - Intrusion Detection System.md>)
