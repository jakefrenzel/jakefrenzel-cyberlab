# 📋 Lab TODO

Open work items, with enough context to pick up cold in a later session.

> Last updated: 2026-10-24, after documenting and applying
> [03 - pfSense Firewall](<cyberlab-documentation/03 - pfSense Firewall.md>) and
> [04 - Managed Switch](<cyberlab-documentation/04 - Managed Switch.md>).

---

## Current State (for context)

**Network** (as of 2026-10-24):

- pfSense CE 2.8.1 routes VLANs 10 – 70 with Kea DHCP (none on Management) and least-privilege rules per VLAN
- The untagged bootstrap LAN is removed; the switch trunks use black-hole native VLAN 700
- Switch management is on VLAN 10, mirroring port 1 → port 3
- Remote access is through the Tailscale package on pfSense, advertising the Management subnet only

**IDS** (built 2026-08-15, see [05 - Intrusion Detection System](<cyberlab-documentation/05 - Intrusion Detection System.md>)):

- Raspberry Pi 5 sensor (`ids-sensor`), Ubuntu 24.04 LTS
- `eth0` (built-in) = capture, no IP, promiscuous, on switch port 3 (mirror destination)
- USB-Ethernet adapter = management, `10.27.10.7/24`, sole default route, switch port 4
- Suricata + Zeek → Filebeat → Logstash → Elasticsearch → Kibana
- Suricata rules auto-update daily at 3am, live reload via `SIGUSR2`
- **The ELK VM (`10.27.10.5`) is currently powered off.** The pipeline was verified end to end on 2026-08-15 with an `nmap` scan producing `ET SCAN` alerts in Kibana

Nothing below is required for the lab to function. These are improvements.

---

## 0. Follow-ups from 2026-10-24

**Priority: high** — small items left over from the firewall and switch work

- **Start the ELK VM** and confirm it comes up on its static `10.27.10.5`. Management has no DHCP anymore, so a VM without a static IP gets no address
- **Tailscale on the Proxmox host**, so it is reachable even when routing through pfSense fails (as on `pve1`)
- **Proxmox laptop settings**: ignore lid close, and set the BIOS to power on after power loss
- **Proxmox USB network adapter**: its only NIC is a USB adapter. Replace it with a more reliable one, or a built-in NIC if available
- **Watch pfSense `igc1`**: on 2026-10-24 the LAN port stopped receiving and the whole lab went dark until pfSense was rebooted. If it happens again, check `Status -> Interfaces -> LAN` first, then suspect the cable to switch port 1 or the port itself
- **Switch Access Control** (do it at home, lockout risk): allow only the Management subnet and the Tailscale range to reach the switch GUI
- **Optional cleanups**: delete the unused NordVPN client on pfSense, find out what the running FreeRADIUS service is for, and set the switch's time zone to Central

---

## 1. Migrate Sensor + ELK to Monitoring VLAN 70

**Priority: medium** — architectural correctness

`01 - Network Architecture.md` designates VLAN 70 (`10.27.70.0/24`) for "IDS,
Suricata, monitoring, logging", but both the sensor and the ELK VM currently sit
on VLAN 10. This was a deliberate choice: keeping them on the same VLAN meant
Filebeat → Logstash needed **zero firewall rules**, which removed a variable
while getting everything working for the first time.

**Move both together.** Moving only one turns an intra-VLAN connection into an
inter-VLAN one and requires pfSense rules to keep working — more moving parts
than moving the pair.

Planned addressing (from [02 - IP Schemes](<cyberlab-documentation/02 - IP Schemes.md>)):

| Host | Current | Target |
|------|---------|--------|
| Pi sensor (management NIC) | `10.27.10.7` | `10.27.70.10` |
| ELK VM | `10.27.10.5` | `10.27.70.20` |

**Steps:**
1. On the switch, make **port 4** (sensor management) an untagged member of VLAN 70, set its PVID to 70, and remove it from VLAN 10
2. In Proxmox, change the ELK VM's network device **VLAN Tag** from `10` to `70`
3. Re-address both hosts (`netplan set ... --origin-hint 98-usb-mgmt` on the Pi)
4. Update `output.logstash.hosts` in `/etc/filebeat/filebeat.yml`
5. Update `server.publicBaseUrl` in `/etc/kibana/kibana.yml`
6. To reach Kibana over Tailscale, add `10.27.70.0/24` to pfSense's advertised Tailscale routes and to the tailnet policy

No new pfSense rules are needed: VLAN 70 already exists with DHCP, the sensor and ELK stay on the same subnet, and Management (VLAN 10) is allowed to reach every VLAN, including Kibana on VLAN 70.

**Gotchas:**
- The capture NIC (`eth0`) is **not** involved — it has no IP and stays on port 3.
- Don't forget the Kibana data view; index patterns are unaffected by IP changes,
  but the `publicBaseUrl` will 404 if missed.
- Verify with `sudo filebeat test output` before assuming it works — a wrong
  address fails **silently and indefinitely** (this exact bug cost hours on
  2026-08-15: `10.27.0.5` instead of `10.27.10.5`).
- Check for stale references to the old sensor IP outside the Pi: firewall rules,
  Tailscale ACLs, SSH configs.

---

## 2. Enable TLS Between Filebeat and Logstash

**Priority: low** — hardening

`filebeat test output` currently reports `TLS... WARN secure connection disabled`.
Beats → Logstash traffic is plaintext on port 5044. Acceptable on a trusted
management VLAN, but the lab is a place to practise doing it properly.

**Steps:**
1. Generate a CA and a server cert for Logstash (SAN must include the ELK VM's IP)
2. Configure `ssl_enabled`, `ssl_certificate`, `ssl_key` on the `beats` input in
   `/etc/logstash/conf.d/beats-to-es.conf`
3. Copy the CA cert to the sensor and set
   `output.logstash.ssl.certificate_authorities` in `filebeat.yml`
4. Optionally add client-cert auth (`ssl_client_authentication => required`)

**Gotcha:** the existing `http_ca.crt` is Elasticsearch's HTTP CA — it does **not**
cover Logstash's beats input. This needs its own certificate.

---

## 3. Expand Mirror Sources for Intra-VLAN Visibility

**Priority: low** — only if a specific blind spot appears

Currently only port 1 (the pfSense trunk) is mirrored. That covers all
inter-VLAN and internet-bound traffic — effectively the whole threat surface —
without duplicates or oversubscription.

**The blind spot:** two devices in the *same* VLAN on *different* switch ports
talk directly through the switch fabric and never cross port 1, so that traffic
is invisible to the sensor.

Only add sources if you hit a concrete need (e.g. lateral movement exercises
between two VLAN 40 hosts). Understand the cost first:

- **Duplicates** — traffic that also crosses port 1 gets captured twice,
  producing duplicate alerts and inflated flow counts that mimic real anomalies
- **Oversubscription** — multiple gigabit sources into one gigabit destination;
  the switch drops the overflow silently, with no indication of what was lost

Never add **port 4** (sensor management) as a source — that just captures
Filebeat shipping its own logs.

---

## 4. Monitor for Packet Loss as Traffic Grows

**Priority: low** — ongoing hygiene

Currently clean (Zeek `percent_lost: 0.0`, no Suricata kernel drops), but the lab
is quiet. Re-check after adding VMs, running heavy exercises, or expanding mirror
sources.

```shell
tail -3 /usr/local/zeek/logs/current/capture_loss.log
grep -iE "kernel_drops" /var/log/suricata/stats.log | tail -5
```

If drops appear, in rough order of effort: reduce mirror sources → increase
Suricata's `af-packet` threads/ring size → move to a more capable sensor.

---

## 5. Elasticsearch Index Lifecycle Management

**Priority: medium** — will eventually break things if ignored

No ILM policy is configured. Indices grow forever until the ELK VM's disk fills,
at which point Elasticsearch goes read-only and ingestion stops.

Set a policy on `logs-suricata.*` and `logs-zeek.*` — e.g. rollover at 10GB or
7 days, delete after 30–90 days. Kibana: **Stack Management → Index Lifecycle
Policies**.

Sensor-side log growth is already handled (logrotate for Suricata, Zeek's own
rotation plus `zeekctl cron`).

---

## 6. Smaller Cleanups

- **Stale netplan files** — `/etc/netplan/` holds `00-installer-config.yaml.disabled`
  and `00-netcfg.yaml.bak`. Harmless (netplan only reads `*.yaml`), delete if you
  want tidiness.
- **`/etc/default/suricata`** — contains `RUN=no` and `LISTENMODE=nfqueue`, left
  over from the SysV init script. Ignored by the systemd unit. Cosmetic only.
- **Zeek intel feeds** — currently only the `8.8.8.8` test indicator. Replace with
  real feeds (EmergingThreats, AbuseIPDB, AlienVault OTX).
- **Kibana alerting** — no detection rules configured. Consider email/webhook on
  high-severity Suricata alerts.
- **v2 archive docs** — `suricata-ids.md` and `zeek-ids.md` still describe the
  single-NIC design and say to mirror "all ports". Superseded by the v3 doc.
  Fine to leave as historical record, but don't follow them.

---

## 7. Documentation Gaps

- **Proxmox host guide**: install, VLAN-aware `vmbr0` with the host on `vmbr0.10`, VM VLAN tags, and the follow-ups in section 0
- **Tailscale guide**: pfSense package, advertised routes, tailnet policy, and the tagged honeypot sensor
- **ELK guide** still lives in `archive/v2/configuration/elk-stack.md`. Move it into `cyberlab-documentation/` as a numbered guide
- **Suricata and Zeek installation**, and the daily rule-update job, are only documented in the v2 archive and this file
- **01 and the network diagram**: add the HP EliteDesk (`pve1`) and the VLAN 700 black hole once `pve1` is home
