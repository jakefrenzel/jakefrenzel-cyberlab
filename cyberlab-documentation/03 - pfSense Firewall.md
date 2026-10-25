# 🔥 03 - pfSense Firewall

## Objective
Take the Protectli Vault from a blank box to the lab's router and firewall: all seven VLANs routed, DHCP and DNS served on each one, and firewall rules that keep the untrusted networks away from Management and from each other.

## Hardware
| Device | Purpose |
|--------|---------|
| Protectli Vault VP2440 | pfSense firewall / router (`core-fw`) |
| Dell Laptop | Admin PC for the initial setup |
| USB drive | pfSense installer |

### Ports
| Port | Interface | Role |
|------|-----------|------|
| `igc0` | WAN | Uplink to the ISP router |
| `igc1` | LAN | 802.1Q trunk to switch port 1, carries VLANs 10 – 70 |
| `ixl0`, `ixl1` | Unused | 10G SFP+ ports, spare |

## Software
| Item | Value |
|------|-------|
| pfSense | Community Edition 2.8.1 |
| Hostname | `core-fw` |
| Domain | `cyberlab.arpa` |
| DHCP server | Kea (see [DHCP](#dhcp)) |
| Packages | Tailscale |

> [!NOTE]
> The WAN gets a private `192.168.0.x` address from the ISP router, so the lab sits behind a second NAT. This keeps the lab isolated from the home network, and nothing in this guide needs inbound access from the internet.

---

## Steps taken

### Install pfSense
1. Downloaded the pfSense CE installer from Netgate, verified its checksum, and wrote it to a USB drive
2. Connected `igc0` (WAN) to the ISP router **before** starting the install
3. Booted the Protectli from the USB drive and installed pfSense
4. At the console, assigned interfaces: WAN = `igc0`, LAN = `igc1`

> [!IMPORTANT]
> The current Netgate installer for pfSense CE is an **online installer**. It downloads the installation from Netgate's servers, so the WAN port needs a working internet connection during the install. Without one, the install fails.

### Setup wizard
Plug the Admin PC directly into `igc1` and browse to the default LAN address `https://192.168.1.1` (user `admin`, password `pfsense`).

```pfsense
System -> Setup Wizard
```

| Wizard step | Setting |
|-------------|---------|
| General Information | Hostname `core-fw`, domain `cyberlab.arpa`, DNS servers `1.1.1.1` and `1.0.0.1` |
| General Information | Uncheck **Override DNS**, so the ISP router's DNS is not added to the list |
| Time Server | Time zone `America/Chicago` |
| WAN | Type **DHCP**, leave **Block RFC1918 private networks** and **Block bogon networks** checked |
| LAN | `10.27.0.1/24` (temporary bootstrap network, removed at the end of the setup) |
| Admin Password | A long, unique password stored in a password manager |

After the wizard reloads, browse to `https://10.27.0.1`.

> [!NOTE]
> The untagged LAN (`10.27.0.0/24`) is only a **bootstrap network**. It gives the Admin PC a way in before any VLANs or switch configuration exist. Once Management works on VLAN 10, it is removed. See [Remove the bootstrap LAN](#remove-the-bootstrap-lan).

### Update pfSense
```pfsense
System -> Update
```

1. Installed any available update before configuring anything else

### Create the VLANs
```pfsense
Interfaces -> Assignments -> VLANs
```

1. Clicked **Add** and created one VLAN per network, all with parent interface `igc1`:

| VLAN Tag | Description |
|----------|-------------|
| 10 | Management |
| 20 | Workstations |
| 30 | Servers |
| 40 | Labs |
| 50 | DMZ |
| 60 | Guest |
| 70 | Monitoring |

### Assign the VLAN interfaces
```pfsense
Interfaces -> Assignments
```

1. Selected each VLAN from **Available network ports** and clicked **Add**
2. Opened each new interface and set:

| Setting | Value |
|---------|-------|
| Enable | ☑ Enable interface |
| Description | `VLAN10`, `VLAN20`, ... `VLAN70` |
| IPv4 Configuration Type | Static IPv4 |
| IPv4 Address | `10.27.<VLAN>.1` / `24` (e.g. `10.27.10.1/24`) |

3. Saved and applied the changes on each interface

> [!TIP]
> New interfaces have **no firewall rules**, so pfSense blocks everything that arrives on them. A VLAN that "doesn't work" right after it is created is usually just missing rules. See [Firewall rules](#firewall-rules).

### DHCP
pfSense 2.8 still ships the old ISC DHCP server but marks it end-of-life. Switch to Kea before creating any DHCP scopes:

```pfsense
System -> Advanced -> Networking -> DHCP Options
```

1. Set **Server Backend** to **Kea DHCP** and saved

Then enable a pool on every VLAN except Management:

```pfsense
Services -> DHCP Server -> VLAN20
```

1. ☑ Enable DHCP server on the interface
2. Set **Address Pool Range** to `10.27.20.100` – `10.27.20.199`
3. Saved, then repeated for VLANs 30, 40, 50, 60, and 70

| Interface | DHCP |
|-----------|------|
| VLAN10 (Management) | **Disabled**, static IPs only |
| VLAN20 – VLAN70 | `10.27.<VLAN>.100` – `10.27.<VLAN>.199` |

> [!IMPORTANT]
> Leave DHCP **off** on VLAN10. Every Management device gets a static IP from [02 - IP Schemes](<02 - IP Schemes.md>), so an unknown device plugged into a Management port gets no address.

### DNS resolver
```pfsense
Services -> DNS Resolver -> General Settings
```

| Setting | Value |
|---------|-------|
| Enable | ☑ Enable DNS resolver |
| Network Interfaces | All |
| DNS Query Forwarding | ☑ Enable Forwarding Mode |

1. Saved and applied

With forwarding mode on, pfSense answers every VLAN's DNS queries and forwards them to `1.1.1.1` / `1.0.0.1` from the General Information step. Each VLAN's DHCP pool hands out its own gateway (`10.27.<VLAN>.1`) as the DNS server automatically.

> [!TIP]
> Optional hardening: check **Use SSL/TLS for outgoing DNS Queries to Forwarding Servers** so upstream queries are encrypted. Set the DNS server hostname to `cloudflare-dns.com` next to each server in `System -> General Setup` so the certificate can be verified.

### Firewall rules
pfSense evaluates rules **top to bottom per interface, and the first match wins**. Each untrusted VLAN gets the same pattern: allow what it needs from the firewall, block the firewall itself, block every private network, then allow the internet.

#### Create an alias for private networks
```pfsense
Firewall -> Aliases -> IP
```

| Setting | Value |
|---------|-------|
| Name | `PRIVATE_NETWORKS` |
| Type | Network(s) |
| Networks | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `100.64.0.0/10` |
| Description | All private and Tailscale ranges |

Including `192.168.0.0/16` keeps the lab off the home network behind the ISP router. Including `100.64.0.0/10` keeps lab VLANs from reaching tailnet devices through pfSense's Tailscale interface.

#### VLAN10 - Management
```pfsense
Firewall -> Rules -> VLAN10
```

| # | Action | Protocol | Source | Destination | Port | Description |
|---|--------|----------|--------|-------------|------|-------------|
| 1 | Pass | TCP | VLAN10 subnets | This Firewall | 443 | Allow web GUI from Management |
| 2 | Pass | Any | VLAN10 subnets | Any | Any | Allow Management to anywhere |

Rule 1 is technically covered by rule 2. It stays as an explicit, separate rule so the GUI is never lost by accident when rule 2 is edited later. It takes over from the anti-lockout rule once the bootstrap LAN is removed.

#### VLAN20 – VLAN70 - Untrusted networks
Repeat this rule set on each of VLAN20, VLAN30, VLAN40, VLAN50, VLAN60, and VLAN70, replacing `VLANxx` with the interface:

| # | Action | Protocol | Source | Destination | Port | Description |
|---|--------|----------|--------|-------------|------|-------------|
| 1 | Pass | TCP/UDP | VLANxx subnets | VLANxx address | 53 | Allow DNS to pfSense |
| 2 | Pass | ICMP (echo request) | VLANxx subnets | VLANxx address | Any | Allow ping to gateway |
| 3 | Block | Any | VLANxx subnets | This Firewall | Any | Block access to pfSense |
| 4 | Block | Any | VLANxx subnets | `PRIVATE_NETWORKS` | Any | Block other VLANs and private networks |
| 5 | Pass | Any | VLANxx subnets | Any | Any | Allow internet |

> [!TIP]
> Build the rule set once on VLAN20, then use the **Copy** button on the rules page to copy the rules to the other interfaces. Check each copied rule afterwards, because source, destination and description fields that name `VLAN20` must be changed to the target interface.

#### Exceptions
Add exceptions **above** rule 4 on the interface where the traffic starts, and only when something needs them:

| Interface | Action | Protocol | Source | Destination | Port | Description |
|-----------|--------|----------|--------|-------------|------|-------------|
| VLAN40 | Pass (disabled by default) | Any | VLAN40 subnets | VLAN30 subnets | Any | Lab exercises: Lab → Servers |

The lab exercise rule exists so scans from Kali can cross the trunk and be seen by the IDS. Enable it only while running an exercise, then disable it again.

> [!WARNING]
> Changing a rule does not end connections that already exist. After tightening rules, clear the old connections so they are re-checked:
> ```pfsense
> Diagnostics -> States -> Reset States
> ```
> This briefly drops every connection through the firewall, including your own GUI session.

### Harden admin access
```pfsense
System -> User Manager
```

1. Clicked **Add** and created a personal admin account (`<your-username>`) with a long password
2. Added it to the **admins** group and saved
3. Logged out and logged back in **as the new user** to prove it works
4. Edited the default `admin` user, checked **This user cannot login**, and saved

```pfsense
System -> Advanced -> Admin Access
```

| Setting | Value |
|---------|-------|
| Protocol | HTTPS |
| Secure Shell | Disabled (enable only when needed, with key-only authentication) |

> [!WARNING]
> Do not disable `admin` before testing the new account. If the new account is broken, there is no other way into the GUI except the console.

### Back up the configuration
```pfsense
Diagnostics -> Backup & Restore
```

1. Set **Backup area** to **All**
2. ☑ Checked **Encrypt this configuration file** and entered a passphrase stored in a password manager
3. Clicked **Download configuration as XML**

> [!IMPORTANT]
> The configuration file contains password hashes, VPN credentials, and the full network layout. Always encrypt it, and **never commit it to this repository**. Take a new backup after every significant change.

### Remove the bootstrap LAN
Do this only after the switch is configured and the Admin PC reaches the GUI at `https://10.27.10.1` from a VLAN 10 port.

1. Confirmed nothing still depends on the bootstrap LAN:
```pfsense
Status -> DHCP Leases
```
   No active leases on LAN, and every Management device answers on its VLAN 10 address

2. Disabled DHCP on LAN:
```pfsense
Services -> DHCP Server -> LAN
```
   Unchecked **Enable DHCP server on LAN interface**, removed any old static mappings, and saved

3. Removed the LAN address:
```pfsense
Interfaces -> LAN
```
   Set **IPv4 Configuration Type** to **None**, saved, and applied

`igc1` stays assigned as the parent of the VLANs, it just no longer carries an untagged network. The anti-lockout rule still appears on the LAN rules tab, but it targets the LAN address, which no longer exists, so it matches nothing. VLAN10 rule 1 now provides GUI access.

> [!TIP]
> Recovery without the bootstrap LAN: connect a monitor and keyboard to the Protectli and use the console menu. **Option 2** sets an interface's IP address, which is enough to get back into the GUI.

---

## ✅ Verification
From a laptop on an access port in each VLAN:

| Test | VLAN10 | VLAN20 – VLAN70 |
|------|--------|-----------------|
| Gets an address | Static IP only, no DHCP lease | `10.27.<VLAN>.100` – `.199` |
| `nslookup example.com` | ✅ Resolves | ✅ Resolves |
| `ping 1.1.1.1` | ✅ Replies | ✅ Replies |
| `ping 10.27.<VLAN>.1` (own gateway) | ✅ Replies | ✅ Replies |
| `https://10.27.10.1` (GUI) | ✅ Loads | ❌ Blocked |
| `ping 10.27.10.3` (Proxmox) | ✅ Replies | ❌ Blocked |
| `ping 192.168.0.1` (ISP router) | ✅ Replies | ❌ Blocked |

Blocked results confirm the firewall rules, not a broken network: DNS and internet still work.

---

## Troubleshooting
| Symptom | Cause | Fix |
|---------|-------|-----|
| Installer fails partway through | The online installer cannot reach Netgate | Connect WAN to a working internet connection before installing |
| New VLAN gets no DHCP lease | No firewall rules on the interface, or DHCP not enabled | Add the rule set and enable the pool |
| Client has an address but no internet | Missing allow rule, or rule order wrong | Check that the internet rule (5) is below the block rules and present |
| Client cannot resolve DNS | DNS rule (1) missing or below a block rule | Move the DNS rule to the top |
| Old traffic still allowed after tightening rules | Existing connection states | `Diagnostics -> States -> Reset States` |
| Locked out of the GUI | Rule mistake or the bootstrap LAN removed too early | Use the console menu (option 2) to set an interface IP |
| ISP router's DNS appears in pfSense's DNS list | **Override DNS** is still checked | Uncheck it in `System -> General Setup` |

---

## Notes & Observations
1. pfSense blocks everything on a new interface until rules are added. Building the rules right after creating each VLAN avoids a long "why doesn't this work" phase
2. A first-match rule order means **allows for the firewall's own services go first, blocks second, and the general internet allow last**
3. Keeping the bootstrap LAN after the VLANs worked was a leftover risk: any untagged device on the trunk would get an address and reach the GUI through the anti-lockout rule
4. The ISC DHCP server shows an end-of-life warning in 2.8. Choosing Kea at the start avoids migrating later
5. An unchecked **Override DNS** box matters even in forwarding mode: otherwise the ISP router is silently added as an upstream DNS server
6. The domain `cyberlab.arpa` works for a lab, but the name officially reserved for home networks is `home.arpa` (RFC 8375)

## ➡️ Next Up
[**04 - Managed Switch:** Configure 802.1Q VLANs, trunk and access ports, and port mirroring](<04 - Managed Switch.md>)
