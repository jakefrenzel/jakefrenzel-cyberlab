# 🛡️ 05 - Intrusion Detection System

## Objective

Deploy a passive network IDS on a Raspberry Pi 5 that inspects mirrored switch
traffic with **Suricata** (signature detection) and **Zeek** (protocol analysis),
and ships the results to the ELK stack for visualization.

> [!IMPORTANT]
> This supersedes the v2 `suricata-ids.md` and `zeek-ids.md` documents, which
> assumed a **single NIC** doing both capture and management. That design cannot
> work on this switch — see [Why Two NICs](#why-two-nics).

---

## Why Two NICs

A port-mirroring **destination** port on the Netgear GS308EP is effectively
receive-only. The sensor can see everything copied to it, but cannot transmit
back out the same port. That breaks a single-NIC design in a way that is easy to
misdiagnose:

- You cannot SSH to the sensor over the capture interface.
- Filebeat cannot ship logs to ELK over the capture interface.

The fix is a **second network interface** dedicated to management. This lab uses
a USB-to-Ethernet adapter (ASIX AX88179) alongside the Pi's built-in NIC.

### Interface Roles

| Interface | Hardware | Role | Address | Switch Port |
|-----------|----------|------|---------|-------------|
| `eth0` | Built-in (PCIe) | **Capture** | none | 3 (mirror destination) |
| `enxAABBCCDDEEFF` | USB adapter | **Management** | `10.27.10.7/24` | 4 (VLAN 10 access) |

> [!NOTE]
> Replace `enxAABBCCDDEEFF` throughout with your adapter's actual interface name.
> USB NICs are named `enx` + their MAC address; find yours with `ip -br link`.

**Capture belongs on the built-in NIC.** USB adapters drop packets under
sustained load, and dropped packets on a sensor mean missed detections. The
built-in NIC is PCIe-attached and more reliable. Management traffic is low
bandwidth, so the USB adapter is a good fit there.

Identify which is which by device path, not by guessing:

```shell
ls -l /sys/class/net/
```

The built-in NIC resolves through `...ethernet/net/eth0`; the USB adapter
through `...usb/xhci-hcd...`.

---

# Part 1 — Configure the Management Interface

Work in this order. The management interface must be **proven working before**
`eth0` is stripped of its address, otherwise a failure locks you out of the Pi.

> [!TIP]
> Use `netplan set` rather than editing YAML by hand. It generates valid syntax
> itself, which avoids indentation corruption — a common failure when pasting
> multi-line YAML through SSH.

1. Plug the USB adapter into a **VLAN 10 access port** (PVID 10 untagged). Not a
   mirror port yet.

2. Give it an address, with **no gateway** for now:

```shell
sudo netplan set ethernets.enxAABBCCDDEEFF.dhcp4=false --origin-hint 98-usb-mgmt
sudo netplan set ethernets.enxAABBCCDDEEFF.dhcp6=false --origin-hint 98-usb-mgmt
sudo netplan set ethernets.enxAABBCCDDEEFF.optional=true --origin-hint 98-usb-mgmt
sudo netplan set ethernets.enxAABBCCDDEEFF.addresses=[10.27.10.7/24] --origin-hint 98-usb-mgmt
sudo netplan apply
```

3. Fix ARP behaviour. With two interfaces on one subnet, Linux will answer ARP
   for either address out of either NIC ("ARP flux"), so a connectivity test can
   succeed over the *wrong* interface and prove nothing:

```shell
echo 'net.ipv4.conf.all.arp_ignore=1' | sudo tee /etc/sysctl.d/98-ids-arp.conf
echo 'net.ipv4.conf.all.arp_announce=2' | sudo tee -a /etc/sysctl.d/98-ids-arp.conf
sudo sysctl --system
```

4. Verify the link is real:

```shell
ip -br addr show enxAABBCCDDEEFF
sudo ethtool enxAABBCCDDEEFF | grep -E 'Speed|Link detected'
```

> [!NOTE]
> With the generic `cdc_ncm` driver, `ethtool` may report `Duplex: Half` and
> `Auto negotiation: no`. These are unpopulated fields, not real link problems.
> Judge the link by actual throughput and packet loss instead.

5. **Definitive test — unplug `eth0` physically**, then from another machine:

```shell
ssh youruser@10.27.10.7
```

If that connects with `eth0` disconnected, the USB path genuinely works. This is
the only test ARP flux cannot fake.

---

# Part 2 — Convert `eth0` to a Capture Interface

Do this while `eth0` is still unplugged. Management is already proven, so there
is no window where two default routes compete.

1. Move the default route and DNS onto the management NIC:

```shell
sudo netplan set 'ethernets.enxAABBCCDDEEFF.routes=[{"to": "default", "via": "10.27.10.1"}]' --origin-hint 98-usb-mgmt
sudo netplan set ethernets.enxAABBCCDDEEFF.nameservers.addresses=[10.27.10.1] --origin-hint 98-usb-mgmt
```

2. Strip `eth0` down to a bare listener:

```shell
sudo netplan set ethernets.eth0.addresses=NULL --origin-hint 99-static-mgmt
sudo netplan set ethernets.eth0.routes=NULL --origin-hint 99-static-mgmt
sudo netplan set ethernets.eth0.nameservers=NULL --origin-hint 99-static-mgmt
```

3. Silence it completely. A mirror destination port cannot carry transmissions,
   so the capture NIC must not emit IPv6 router solicitations or MLD reports:

```shell
sudo netplan set 'ethernets.eth0.link-local=[]' --origin-hint 99-static-mgmt
sudo netplan set ethernets.eth0.accept-ra=false --origin-hint 99-static-mgmt
```

> [!WARNING]
> Netplan validates the **whole merged config** on every `set`, so intermediate
> commands may print `Conflicting default route declarations for IPv4`. This is
> expected while both interfaces briefly declare a default route, and clears once
> `ethernets.eth0.routes=NULL` is applied. The writes still succeed.

4. Review the merged result, then apply:

```shell
sudo netplan get all
sudo netplan apply
ip route
```

Expected: exactly one default route, via the management NIC.

```
default via 10.27.10.1 dev enxAABBCCDDEEFF proto static
10.27.10.0/24 dev enxAABBCCDDEEFF proto kernel scope link src 10.27.10.7
```

5. Plug `eth0` into **switch port 3**. Confirm it comes up with no address:

```shell
ip -br addr show eth0
```

## Prevent the Boot Hang

`optional: true` is supposed to mark an interface as not-required-for-boot, but
netplan **does not emit `RequiredForOnline=` when an interface has no address
config** — which is exactly the case for a capture NIC. systemd then defaults to
*required*, and `systemd-networkd-wait-online` blocks boot for ~2 minutes waiting
for an interface that can never become routable.

Check for the problem:

```shell
networkctl status eth0 | grep -E 'Required|State'
```

Fix it with a networkd drop-in. Placing it under `/etc` means it survives every
future `netplan apply`, which rewrites the generated file in `/run`:

```shell
sudo mkdir -p /etc/systemd/network/10-netplan-eth0.network.d
echo '[Link]' | sudo tee /etc/systemd/network/10-netplan-eth0.network.d/override.conf
echo 'RequiredForOnline=no' | sudo tee -a /etc/systemd/network/10-netplan-eth0.network.d/override.conf
sudo systemctl restart systemd-networkd
```

Verify:

```shell
networkctl status eth0 | grep -E 'Required|State'
```

Expected: `State: carrier (configured)` and `Required For Online: no`. Carrier
present with no address is the correct state for a capture interface.

---

# Part 3 — Configure Port Mirroring

In the Netgear GS308EP web GUI:

```netgear-web-gui
System -> Monitoring -> Mirroring
```

(Older firmware: `Diagnostics -> Port Mirroring`)

| Setting | Value |
|---------|-------|
| Mirroring | Enable |
| Destination Port | `3` (the sensor's capture NIC) |
| Source Port | `1` (the pfSense LAN trunk) |

> [!NOTE]
> The GS308EP is a Smart Managed *Plus* switch and has **no Tx/Rx/Both direction
> selector** — it mirrors bidirectionally by default. That option exists only on
> the Smart Managed *Pro* line (GS308T, GS310TP).

## Mirror the Trunk, Not Every Port

The obvious instinct is to mirror all ports. Don't:

- **Duplicates.** Traffic between Proxmox (port 2) and the internet also crosses
  the pfSense trunk on port 1. Mirroring both makes Suricata see every packet
  twice — duplicate alerts and inflated flow counts that mimic real anomalies.
- **Oversubscription.** Several gigabit sources funnelled into one gigabit
  destination overflows, and the switch drops the excess silently.
- **Coverage.** Port 1 carries all inter-VLAN and internet-bound traffic, which
  is essentially the entire threat surface of this lab.

**Trade-off:** traffic between two devices in the *same* VLAN on different switch
ports switches locally and never reaches port 1, so it is not captured. Add
source ports deliberately if you need that, rather than starting broad.

Always exclude **port 4** (the management NIC) as a source — mirroring it just
captures Filebeat shipping its own logs.

---

# Part 4 — Verify the Mirror

**Do this before touching Suricata.** Configuring an IDS on top of a dead feed
produces a silent failure that is far harder to diagnose later.

```shell
sudo timeout 15 tcpdump -i eth0 -nn -c 20 'not host 10.27.10.7'
```

The filter excludes the sensor's own traffic, so **anything you see is proof the
mirror works** — conversations between hosts unrelated to the Pi.

```
IP 10.27.10.3.8006 > 10.27.10.1.22898: Flags [P.], ...
IP 10.27.10.1.22898 > 10.27.10.3.8006: Flags [.], ack 1169, ...
```

Seeing **both directions** (data one way, acks the other) confirms bidirectional
mirroring.

Confirm trunk tagging:

```shell
sudo timeout 15 tcpdump -i eth0 -nn -e -c 20 vlan
```

Frames should show `vlan 10`, `vlan 20`, etc. Suricata and Zeek both handle
802.1Q tagged traffic natively.

> [!WARNING]
> If `tcpdump` shows nothing, stop and fix the mirror. Do not continue.

---

# Part 5 — Bind Suricata and Zeek

Both must be told which interface to capture on — they do not guess.

**Suricata** — set the interface in the `af-packet` block of
`/etc/suricata/suricata.yaml` rather than passing `-i` on the command line, so it
survives restarts:

```yaml
af-packet:
  - interface: eth0
    cluster-id: 99
    cluster-type: cluster_flow
    defrag: yes
```

**Zeek** — `/usr/local/zeek/etc/node.cfg`:

```
[zeek]
type=standalone
host=localhost
interface=eth0
```

> [!NOTE]
> `/etc/default/suricata` may contain `RUN=no` and `LISTENMODE=nfqueue`. That
> file drives the old SysV init script and is ignored by the systemd unit.
> Confirm what is actually running with `ps -eo args | grep [s]uricata` — it
> should show `--af-packet`.

## Interface Tuning

Disable NIC offloading and enable promiscuous mode on the **capture** interface.
Offloading reassembles segments before the kernel sees them, so Suricata would
inspect packets that never existed on the wire.

`/usr/local/bin/ids-interface-tuning.sh`:

```shell
#!/bin/bash
/usr/sbin/ethtool -K eth0 gro off
/usr/sbin/ethtool -K eth0 gso off
/usr/sbin/ethtool -K eth0 tso off
/usr/sbin/ethtool -K eth0 lro off
/usr/sbin/ip link set eth0 promisc on
exit 0
```

Run at boot via `ids-interface-tuning.service` (see v2 `suricata-ids.md` for the
unit file). Verify:

```shell
ethtool -k eth0 | grep -E 'generic-receive|generic-segmentation|large-receive'
ip -br link show eth0
```

Expected: offloads `off`, and the `PROMISC` flag present.

---

# Part 6 — Verify Detection End to End

Confirm the engines are processing **mirrored** traffic, not just their own:

```shell
tail -200 /var/log/suricata/eve.json | python3 -c "
import sys,json,collections
c=collections.Counter()
for l in sys.stdin:
    try: e=json.loads(l)
    except: continue
    s,d=e.get('src_ip'),e.get('dest_ip')
    if s and d and '10.27.10.7' not in (s,d): c[(s,d)]+=1
for (s,d),n in c.most_common(8): print(n,s,'->',d)
"
```

Any conversation not involving the sensor is proof of mirrored capture.

## Generating a Test Alert

```shell
nmap -A 10.27.10.3
```

> [!IMPORTANT]
> **Do not scan `10.27.10.1` (pfSense) from outside the switch.** Those packets
> terminate on the firewall itself and never egress port 1, so the mirror never
> sees them and no alert fires — even though everything is configured correctly.
>
> **The mirror sees what transits the trunk, not what terminates on the
> firewall.** Pick a target behind the switch, or scan from a VLAN 40 host
> against a VLAN 30 target so the traffic is routed across port 1.

> [!NOTE]
> The firewall blocks Lab (VLAN 40) from reaching other VLANs. Before scanning a
> VLAN 30 target, enable the **Lab exercises: Lab → Servers** rule under
> `Firewall -> Rules -> VLAN40`, and disable it again when the exercise is done.
> See [03 - pfSense Firewall](<03 - pfSense Firewall.md>).

Check for alerts:

```shell
tail -20 /var/log/suricata/fast.log
```

---

# Part 7 — Ship Logs to ELK

Filebeat runs on the sensor and ships over the **management** interface. Full
setup is in the ELK stack documentation; the sensor-side essentials:

```shell
sudo filebeat modules enable suricata zeek
sudo filebeat test output
```

Expected: `dial up... OK` and `talk to server... OK`.

> [!WARNING]
> Verify the Logstash host address carefully. A single wrong octet produces
> `dial tcp ...: i/o timeout`, and Filebeat then fails silently and indefinitely
> — indistinguishable from "the IDS isn't detecting anything."

```shell
sudo grep -nE "10\.[0-9]+\.[0-9]+\.[0-9]+" /etc/filebeat/filebeat.yml
```

Confirm it is publishing:

```shell
sudo journalctl -u filebeat -n 40 --no-pager | grep -E 'established|ERROR'
```

Expected: `Connection to backoff(async(tcp://10.27.10.5:5044)) established`.

`TLS... WARN secure connection disabled` is expected — the Logstash `beats` input
has no SSL configured. Acceptable on a trusted management VLAN; a candidate for
future hardening.

---

# Troubleshooting

- **Nothing in Kibana, and no alerts at all** — check the chain in order:
  `tcpdump` on the capture NIC → alerts in `fast.log` → `filebeat test output` →
  Logstash on 5044 → Elasticsearch doc counts. Whichever stage is empty is the
  broken one. Note that **multiple independent faults can mask each other** —
  a mirror that was never configured and a Filebeat typo produce identical
  symptoms, and fixing only one looks like no progress.
- **Sensor unreachable after re-cabling** — the mirror destination port cannot
  transmit. Management must be on a separate interface and a normal access port.
- **Boot hangs ~2 minutes** — the capture NIC is marked required-for-online. See
  [Prevent the Boot Hang](#prevent-the-boot-hang).
- **`Network is unreachable` from the sensor** — no default route. Check that the
  gateway moved to the management interface with `ip route`.
- **Connectivity test passes but the interface is actually dead** — ARP flux.
  Apply the `arp_ignore`/`arp_announce` sysctls and re-test by physically
  unplugging the other NIC.
- **Netplan YAML corrupted after pasting** — use `netplan set` instead of
  heredocs. A `<<'EOF'` terminator must sit at column 0 with no leading
  whitespace, which indentation-preserving pastes routinely break.
- **`nmap` produces no alert** — confirm the scan traffic actually crosses the
  mirrored port. Scans terminating on pfSense itself are never mirrored.
- **Duplicate alerts / inflated flow counts** — more than one source port is
  mirrored and packets are being counted twice.

---

# Design Decisions

| Decision | Rationale |
|----------|-----------|
| Capture on built-in NIC | USB adapters drop packets under load; dropped packets mean missed detections |
| Management on USB NIC | Low bandwidth, so the adapter's limitations do not matter |
| Sensor on VLAN 10 (not 70) | Same VLAN as the ELK VM, so log shipping needs no firewall rules. Migrate both to Monitoring VLAN 70 together, as a deliberate step |
| Mirror port 1 only | Full inter-VLAN and internet coverage with no duplicates and no oversubscription |
| Capture NIC fully silent | A mirror destination port cannot transmit; IPv6 link-local and RA would generate useless traffic |
| Both NICs proven before role swap | Eliminates lockout risk — the fallback stays live until the replacement is verified |

---

# Future Enhancements

- Migrate the sensor and ELK VM to **Monitoring VLAN 70**, with pfSense rules for
  `5044/tcp` from sensor to ELK.
- Enable **TLS between Filebeat and Logstash**.
- Disable `wlan0` on the sensor once the wired management path is proven stable —
  an auto-joining open SSID would add an unmanaged third route.
- Add source ports to the mirror if intra-VLAN visibility becomes necessary,
  accepting the duplicate-packet trade-off.
- Monitor `capture_loss.log` (Zeek) and Suricata's `stats.log` for packet drops
  as traffic grows.

## ➡️ Next Up
[**06 - Kubernetes Lab Node:** Proxmox, Terraform, and Talos Linux as code](<06 - Kubernetes Lab Node.md>)
