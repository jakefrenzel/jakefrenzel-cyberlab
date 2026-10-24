# 📋 Lab TODO

Open work items, with enough context to pick up cold in a later session.

> Last updated: 2026-08-15, after completing the dual-NIC IDS build documented in
> [03 - Intrusion Detection System](<cyberlab-documentation/03 - Intrusion Detection System.md>).

---

## Current State (for context)

The IDS is **fully working** as of 2026-08-15:

- Raspberry Pi 5 sensor (`ids-sensor`), Ubuntu 24.04 LTS
- `eth0` (built-in) = capture, no IP, promiscuous, on switch port 3 (mirror destination)
- USB-Ethernet adapter = management, `10.27.10.7/24`, sole default route, switch port 4
- Switch mirroring: source port 1 (pfSense trunk) → destination port 3
- Suricata + Zeek → Filebeat → Logstash → Elasticsearch → Kibana
- ELK VM at `10.27.10.5`, all on **VLAN 10 (Management)**
- Suricata rules auto-update daily at 3am, live reload via `SIGUSR2`
- Verified end to end with an `nmap` scan producing `ET SCAN` alerts in Kibana

Nothing below is required for the lab to function. These are improvements.

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

Planned addressing (from `archive/v2/setup/implementation/02-ip_schemes_implementation.md`):

| Host | Current | Target |
|------|---------|--------|
| Pi sensor (management NIC) | `10.27.10.7` | `10.27.70.10` |
| ELK VM | `10.27.10.5` | `10.27.70.20` |

**Steps:**
1. Create VLAN 70 interface + DHCP scope on pfSense if not already present
2. Set switch **port 4** (sensor management) to PVID 70 untagged
3. Move the Proxmox ELK VM to a VLAN 70 tagged bridge
4. Re-address both hosts (`netplan set ... --origin-hint 98-usb-mgmt` on the Pi)
5. Update `output.logstash.hosts` in `/etc/filebeat/filebeat.yml`
6. Update `server.publicBaseUrl` in `/etc/kibana/kibana.yml`
7. Add a pfSense rule: admin workstation → ELK VM `5601/tcp` (Kibana)

**Gotchas:**
- The capture NIC (`eth0`) is **not** involved — it has no IP and stays on port 3.
- Don't forget the Kibana data view; index patterns are unaffected by IP changes,
  but the `publicBaseUrl` will 404 if missed.
- Verify with `sudo filebeat test output` before assuming it works — a wrong
  address fails **silently and indefinitely** (this exact bug cost hours on
  2026-08-15: `10.27.0.5` instead of `10.27.10.5`).
- Check for stale references to the old sensor IP outside the Pi: pfSense DHCP
  static mappings, firewall rules, Tailscale ACLs, SSH configs.

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

- `01 - Network Architecture.md` shows the Pi on switch ports 3 and 4 — verify it
  reflects the final wiring, and note which port is the mirror destination
- The IP scheme table in
  `archive/v2/setup/implementation/02-ip_schemes_implementation.md` assigns
  `10.27.30.10` to **both** Web Server and Database Server (lines 47–48). Fix
  before building out VLAN 30.
