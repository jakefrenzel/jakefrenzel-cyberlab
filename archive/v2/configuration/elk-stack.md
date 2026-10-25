# Elastic Stack (Elasticsearch, Logstash, Kibana) + Filebeat

Centralized logging and visualization for the lab's IDS/DPI sensors. This
ingests **Suricata** (IDS/IPS alerts + flows) and **Zeek** (deep packet
inspection) logs, parses them, and presents them in **Kibana** dashboards.

## Prerequisites
- [x] Complete [Suricata IDS](suricata-ids.md) — writes EVE JSON to `/var/log/suricata/eve.json`
- [x] Complete [Zeek IDS](zeek-ids.md) — JSON logging enabled, logs in `/usr/local/zeek/logs/current/*.log`
- [x] Proxmox host available to run a new VM
- [x] A management path to reach the sensor (see the mirror-port note below)

## Architecture

```
                          Proxmox VM (Ubuntu 24.04 LTS)
 Raspberry Pi (sensor)   ┌───────────────────────────────────────┐
 ┌───────────────────┐   │  Logstash  →  Elasticsearch  →  Kibana │
 │ Suricata          │   │   :5044         :9200           :5601  │
 │ Zeek              │   └───────────────────────────────────────┘
 │ Filebeat  ───────────────►  (Beats/5044)          ▲
 └───────────────────┘                               │
        eth0  = capture (port mirror, no IP)  your workstation (browser)
        enx…  = management / log shipping (USB-Ethernet)
```

### What runs where — and why

| Component | Host | Reason |
|-----------|------|--------|
| Suricata + Zeek | Raspberry Pi | Must sit where the mirrored traffic lands. CPU-heavy packet capture / DPI. |
| **Filebeat** | Raspberry Pi | Must be local to read the log files, but is very lightweight (tails files, ships them). |
| **Logstash** | Proxmox VM | JVM parsing/enrichment process — too heavy for the Pi. |
| **Elasticsearch** | Proxmox VM | Heaviest: indexing, storage, JVM heap. |
| **Kibana** | Proxmox VM | Web UI for dashboards. |

> **Why not put everything on the Pi?** The Pi is already busy running Suricata
> and Zeek against mirrored traffic. Elasticsearch alone wants several GB of RAM.
> Moving Logstash + Elasticsearch + Kibana to a Proxmox VM keeps the Pi doing
> only its light job (capture + ship). Filebeat is the one piece that *must*
> stay on the Pi, and it barely registers on the CPU.

> **⚠️ Mirror-port gotcha:** On many managed switches the port-mirroring
> **destination** port only *receives* — the Pi may not be able to send its logs
> back out over the same `eth0` it captures on. Use a **separate management
> interface** to ship logs. This lab uses a USB-Ethernet adapter: capture on
> `eth0` (no IP), ship over the USB NIC. See
> [04 - Intrusion Detection System](<../../../cyberlab-documentation/04 - Intrusion Detection System.md>)
> for the full dual-NIC build.

> **Example IPs used below** (adjust to your scheme — Management VLAN10 is
> `10.27.10.0/24` and static-only, with core infrastructure in `.2`–`.9`; see
> [02 - IP Schemes](<../../../cyberlab-documentation/02 - IP Schemes.md>)):
> - ELK VM: `10.27.10.5`
> - Pi sensor (management IP, on the USB NIC): `10.27.10.7`

---

# Part 1 — Provision the ELK VM on Proxmox

1. Create a new VM on Proxmox
   - OS: **Ubuntu Server 24.04 LTS**
   - vCPU: `4`
   - RAM: `8 GB` minimum (8–12 GB recommended — Elasticsearch, Logstash, and Kibana all run here)
   - Disk: `100 GB+` (logs grow fast)
   - NIC: management VLAN (VLAN10)

2. Set a static IP on the management VLAN (e.g. `10.27.10.5`)

3. Update the server
```shell
sudo apt update && sudo apt upgrade -y
```

4. Raise the memory-map limit required by Elasticsearch
```shell
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-elasticsearch.conf
sudo sysctl --system
```

5. Keep time in sync (timestamps must line up with the sensor)
```shell
sudo apt install -y chrony
sudo systemctl enable --now chrony
```

---

# Part 2 — Install Elasticsearch (9.x)

1. Install the prerequisite and Elastic GPG key
```shell
sudo apt install -y apt-transport-https curl gnupg
```
```shell
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

2. Add the Elastic 9.x repository
```shell
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-9.x.list
```

3. Install Elasticsearch
```shell
sudo apt update
sudo apt install -y elasticsearch
```

4. **Save the output.** On first install Elasticsearch prints a one-time block
   containing the **`elastic` superuser password**. Copy it somewhere safe.
   If you miss it, reset it later:
```shell
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

5. Configure it as a single-node cluster reachable on the LAN
```shell
sudo nano /etc/elasticsearch/elasticsearch.yml
```
   Set / adjust these keys:
```yaml
network.host: 0.0.0.0
http.host: 0.0.0.0
discovery.type: single-node
```
   Then **delete** the auto-added `cluster.initial_master_nodes:` line — it
   conflicts with `discovery.type: single-node`.

6. (Optional) Cap the JVM heap so ES doesn't eat all RAM. Set to ~50% of VM RAM
```shell
echo -e '-Xms4g\n-Xmx4g' | sudo tee /etc/elasticsearch/jvm.options.d/heap.options
```

7. Enable and start the service
```shell
sudo systemctl daemon-reload
sudo systemctl enable --now elasticsearch
```

8. Verify (security/TLS is ON by default in 9.x — note the `https` and `-k`)
```shell
curl -k -u elastic https://localhost:9200
```
   You should get a JSON response with `"cluster_name" : "elasticsearch"`.

9. Note the CA certificate path — Logstash, Kibana, and Filebeat will need it
```shell
ls -l /etc/elasticsearch/certs/http_ca.crt
```

---

# Part 3 — Install Kibana

1. Install (same repo already added)
```shell
sudo apt install -y kibana
```

2. Make Kibana reachable from your workstation
```shell
sudo nano /etc/kibana/kibana.yml
```
   Set:
```yaml
server.host: "0.0.0.0"
server.publicBaseUrl: "http://10.27.10.5:5601"
```

3. Enable and start
```shell
sudo systemctl daemon-reload
sudo systemctl enable --now kibana
```

4. Generate a Kibana enrollment token on the **Elasticsearch** side
```shell
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

5. In a browser go to `http://10.27.10.5:5601`. Paste the enrollment token.
   Kibana then asks for a verification code — get it with:
```shell
sudo /usr/share/kibana/bin/kibana-verification-code
```

> **⚠️ The Kibana screen shows the _Windows_ command** (`bin\kibana-verification-code.bat`).
> You're on Ubuntu — drop the `.bat` and run the path above. The code is
> 6 digits and short-lived; if it expires just run the command again.

6. Log in as user `elastic` with the password saved in Part 2, step 4.

## Alternative: manual Kibana ↔ Elasticsearch setup (skip the enrollment token)

The enrollment-token + verification-code wizard can be finicky (expired tokens,
long tokens fat-fingered in a VNC console). You can bypass it entirely and wire
Kibana to Elasticsearch by hand — more reliable, and every value is explicit.

1. Set a password for the built-in `kibana_system` user (the account Kibana uses
   to talk to ES — **not** the one you log in with). Copy the printed password:
```shell
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system
```

2. Give Kibana the Elasticsearch CA cert:
```shell
sudo mkdir -p /etc/kibana/certs
sudo cp /etc/elasticsearch/certs/http_ca.crt /etc/kibana/certs/http_ca.crt
sudo chown root:kibana /etc/kibana/certs/http_ca.crt
sudo chmod 640 /etc/kibana/certs/http_ca.crt
```

3. Add these to `/etc/kibana/kibana.yml` (alongside the `server.*` keys already
   set in step 2):
```yaml
elasticsearch.hosts: ["https://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "PASTE_KIBANA_SYSTEM_PASSWORD_HERE"
elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/http_ca.crt"]
```

4. Restart and watch it come up:
```shell
sudo systemctl restart kibana
sudo journalctl -u kibana -f      # wait for "http server running at http://0.0.0.0:5601"
```

5. Browse to `http://10.27.10.5:5601` — you get a plain **login page** (no token,
   no code). Log in as **`elastic`** (not `kibana_system` — that's a background
   service account with no UI access) using the Part 2 password.

> If the log warns about missing encryption keys, that's non-fatal for a lab,
> but you can silence it and make sessions/saved-objects persistent:
> `sudo /usr/share/kibana/bin/kibana-encryption-keys generate -q`, then paste the
> three `xpack...encryptionKey` lines into `kibana.yml` and restart again.

---

# Part 4 — Install Logstash (same VM)

Logstash receives Beats data from the Pi, optionally enriches it, and forwards
to Elasticsearch. Because it's on the **same VM** as Elasticsearch it can talk
to ES over `https://localhost:9200`, which avoids certificate hostname issues.

1. Install
```shell
sudo apt install -y logstash
```

2. Give Logstash access to the Elasticsearch CA cert
```shell
sudo mkdir -p /etc/logstash/certs
sudo cp /etc/elasticsearch/certs/http_ca.crt /etc/logstash/certs/http_ca.crt
sudo chown root:logstash /etc/logstash/certs/http_ca.crt
```

3. Create the pipeline
```shell
sudo nano /etc/logstash/conf.d/beats-to-es.conf
```
   Paste:
```ruby
input {
  beats {
    port => 5044
  }
}

filter {
  # Base Suricata/Zeek parsing is done by the Filebeat module ingest
  # pipelines (loaded in Part 5). This is where YOUR custom enrichment goes.

  # Tag everything from this lab
  mutate { add_field => { "[labels][environment]" => "cyberlab" } }

  # Route each event to its module's data stream (logs-suricata.eve-default,
  # logs-zeek.connection-default, ...). Filebeat stamps [event][dataset] on
  # every event; we copy it into [data_stream][dataset] so the ES output's
  # auto-routing sends it to the right stream. Without this, everything piles
  # into `logs-generic-default` and the prebuilt dashboards stay empty.
  if [event][dataset] {
    mutate {
      replace => {
        "[data_stream][type]"      => "logs"
        "[data_stream][dataset]"   => "%{[event][dataset]}"
        "[data_stream][namespace]" => "default"
      }
    }
  }

  # Example enrichment: GeoIP on the external source IP for the Kibana map.
  # (Comment out if the module pipeline already adds geo data you prefer.)
  if [source][ip] and [source][ip] !~ /^(10\.|192\.168\.|172\.(1[6-9]|2[0-9]|3[0-1])\.)/ {
    geoip {
      source => "[source][ip]"
      target => "[source][geo]"
    }
  }
}

output {
  elasticsearch {
    hosts    => ["https://localhost:9200"]
    user     => "elastic"
    password => "PASTE_ELASTIC_PASSWORD_HERE"
    ssl_enabled => true
    ssl_certificate_authorities => "/etc/logstash/certs/http_ca.crt"

    # Preserve the Filebeat module's data-stream routing + ingest pipeline,
    # so the prebuilt Suricata/Zeek dashboards still work.
    # NOTE: use `true`, not `auto` — on 9.x `auto` silently resolves to
    # `false` and dumps everything into an `ecs-logstash-*` index instead of
    # the module data streams. `data_stream_auto_routing` then uses the
    # [data_stream][dataset] field set in the filter above.
    data_stream => "true"
    data_stream_auto_routing => true
    pipeline    => "%{[@metadata][pipeline]}"
  }
}
```
   Replace `PASTE_ELASTIC_PASSWORD_HERE` with the `elastic` password.

> **⚠️ Keep the real password out of the repo.** This file holds the `elastic`
> password in cleartext (Logstash needs it). That's fine on the VM, but never
> commit the real value — leave the `PASTE_ELASTIC_PASSWORD_HERE` placeholder in
> any copy you publish. If it leaks, rotate it:
> `sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic`,
> then update this file and restart Logstash.

> **What this Logstash stage buys you:** a central collection point, buffering
> if Elasticsearch is briefly down, and a place to add custom enrichment
> (GeoIP, threat-intel tagging, dropping noisy events) without touching the
> sensor. The heavy per-event *parsing* still happens in the module pipelines.

4. Enable and start
```shell
sudo systemctl enable --now logstash
```

5. Confirm it's listening on 5044
```shell
sudo ss -lntp | grep 5044
```

---

# Part 5 — Install & configure Filebeat on the Raspberry Pi

Run these on the **sensor (Pi)**, not the ELK VM.

1. Add the Elastic 9.x repo (same steps as the VM)
```shell
sudo apt install -y apt-transport-https curl gnupg
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-9.x.list
sudo apt update
sudo apt install -y filebeat
```

2. Enable the Suricata and Zeek modules
```shell
sudo filebeat modules enable suricata zeek
```

3. Point the **Suricata** module at the EVE log
```shell
sudo nano /etc/filebeat/modules.d/suricata.yml
```
```yaml
- module: suricata
  eve:
    enabled: true
    var.paths: ["/var/log/suricata/eve.json"]
```

> **⚠️ The default file ships with `enabled: false`.** If you don't flip the
> `eve` fileset to `enabled: true`, Filebeat won't just skip Suricata — it
> **crash-loops** on startup with `module suricata is configured but has no
> enabled filesets` and ships nothing at all. Same applies to each Zeek fileset
> in the next step. After fixing, clear the crash-loop lockout before
> restarting: `sudo systemctl reset-failed filebeat`.

> **💡 Disable Suricata `stats` events at the source.** Suricata writes a huge
> multi-thousand-field `stats` record to `eve.json` every few seconds. Shipped
> to Elasticsearch these produce 16 MB+ bulk batches that time ES out and stall
> the whole pipeline — and they're useless for the dashboards. On the sensor,
> edit `/etc/suricata/suricata.yaml`, find the `eve-log` output's `types:` list
> and comment out `- stats`, then `sudo systemctl restart suricata`. If a
> backlog already built up, clear it with
> `sudo truncate -s 0 /var/log/suricata/eve.json` (Filebeat re-syncs to the
> truncated file automatically).

4. Point the **Zeek** module at the source-installed log path. Zeek was built
   to `/usr/local/zeek`, so the default paths must be overridden for **each**
   fileset you use.
```shell
sudo nano /etc/filebeat/modules.d/zeek.yml
```
```yaml
- module: zeek
  connection:
    enabled: true
    var.paths: ["/usr/local/zeek/logs/current/conn.log"]
  dns:
    enabled: true
    var.paths: ["/usr/local/zeek/logs/current/dns.log"]
  http:
    enabled: true
    var.paths: ["/usr/local/zeek/logs/current/http.log"]
  ssl:
    enabled: true
    var.paths: ["/usr/local/zeek/logs/current/ssl.log"]
  notice:
    enabled: true
    var.paths: ["/usr/local/zeek/logs/current/notice.log"]
  intel:
    enabled: true
    var.paths: ["/usr/local/zeek/logs/current/intel.log"]
  # add more filesets (files, dhcp, ssh, x509 ...) the same way as needed
```

## One-time asset load (dashboards + ingest pipelines + templates)

`filebeat setup` must talk **directly** to Elasticsearch and Kibana to load the
dashboards, index templates, and module ingest pipelines. Do this once, then
switch Filebeat to ship through Logstash.

5. Copy the Elasticsearch CA cert from the ELK VM to the Pi.

   The cert is `-rw-rw---- root:elasticsearch` (mode `660`), so a normal SSH user
   **can't read it** — `scp` copying it directly fails with a misleading
   `No such file or directory`. First stage a readable copy on the VM, then pull
   that. The `http_ca.crt` is a public certificate (not a private key), so a
   temporary world-readable copy is safe.

```shell
# On the ELK VM — stage a readable copy
sudo cp /etc/elasticsearch/certs/http_ca.crt /tmp/http_ca.crt
sudo chmod 644 /tmp/http_ca.crt
```
```shell
# On the Pi — pull it and put it in place
scp youruser@10.27.10.5:/tmp/http_ca.crt /tmp/http_ca.crt
sudo mv /tmp/http_ca.crt /etc/filebeat/http_ca.crt
sudo chown root:root /etc/filebeat/http_ca.crt
sudo chmod 644 /etc/filebeat/http_ca.crt
```
```shell
# Back on the ELK VM — remove the staging copy
sudo rm /tmp/http_ca.crt
```

6. Run setup pointing at ES/Kibana. `ssl.verification_mode: certificate`
   verifies the CA without requiring the cert to list the VM's IP.
```shell
sudo filebeat setup -e \
  -E 'output.logstash.enabled=false' \
  -E 'output.elasticsearch.hosts=["https://10.27.10.5:9200"]' \
  -E 'output.elasticsearch.username=elastic' \
  -E 'output.elasticsearch.password=PASTE_ELASTIC_PASSWORD_HERE' \
  -E 'output.elasticsearch.ssl.certificate_authorities=["/etc/filebeat/http_ca.crt"]' \
  -E 'output.elasticsearch.ssl.verification_mode=certificate' \
  -E 'setup.kibana.host=http://10.27.10.5:5601' \
  -E 'setup.kibana.username=elastic' \
  -E 'setup.kibana.password=PASTE_ELASTIC_PASSWORD_HERE'
```
   Look for `Loaded dashboards`, `Loaded Ingest pipelines`, and
   `Index setup finished`.

## Switch Filebeat to ship through Logstash

7. Edit the main config
```shell
sudo nano /etc/filebeat/filebeat.yml
```
   - **Comment out** the entire `output.elasticsearch:` block.
   - **Enable** the Logstash output:
```yaml
output.logstash:
  hosts: ["10.27.10.5:5044"]
```

8. Test the config and the connection to Logstash
```shell
sudo filebeat test config
sudo filebeat test output
```

9. Enable and start Filebeat
```shell
sudo systemctl enable --now filebeat
```

---

# Part 6 — Firewall (pfSense) rules

Allow the sensor to reach the ELK VM and your workstation to reach Kibana.
Keep these ports off untrusted VLANs.

| Source | Destination | Port | Purpose |
|--------|-------------|------|---------|
| Pi sensor | ELK VM | `5044/tcp` | Filebeat → Logstash |
| Your workstation | ELK VM | `5601/tcp` | Browser → Kibana |

> Do **not** expose `9200` (Elasticsearch) or `5601` (Kibana) to the internet or
> to lab/user VLANs. Logstash → Elasticsearch stays on `localhost`.

---

# Part 7 — Kibana Dashboards

1. Generate some traffic so there's data. From Kali on a lab VLAN:
```shell
nmap -A 10.27.0.1
curl http://8.8.8.8
```
   (The nmap triggers a Suricata `ET SCAN` alert; the curl hits the Zeek intel
   feed you configured for `8.8.8.8`.)

2. In Kibana, open **Discover** and filter to confirm data is arriving:
   - `event.module: "suricata"`
   - `event.module: "zeek"`

3. **Fix the data view first (Logstash-specific).** `filebeat setup` builds the
   dashboards around the `filebeat-*` data view, but shipping through Logstash
   lands your data in `logs-suricata.*` / `logs-zeek.*` streams — which
   `filebeat-*` doesn't match, so the dashboards look empty even though the data
   is there (you'll see it in **Discover** under a `logs-*` view). Widen the data
   view once:
   - **Stack Management → Data Views** (direct URL:
     `http://10.27.10.5:5601/app/management/kibana/dataViews`)
   - Open **`filebeat-*`**, edit it, and change the index pattern to:
     ```
     filebeat-*,logs-suricata.*,logs-zeek.*
     ```
   - Confirm the timestamp field is `@timestamp`, then **Save**.

4. Open the prebuilt dashboards: **Menu → Dashboards**, then search for:
   - `[Filebeat Suricata] Alert Overview`
   - `[Filebeat Suricata] Events Overview`
   - `[Filebeat Zeek] Overview`
   - `[Filebeat Zeek] Connections`, `... DNS`, `... HTTP`, etc.

5. Confirm your nmap alert and Zeek connections show up on the dashboards. If a
   dashboard is empty, check the time range (**Last 24 hours**) and that you've
   generated the relevant traffic — the Alert dashboards need actual **alert**
   events (e.g. from the `nmap` scan), not just flows.

---

# Part 8 — End-to-end verification

1. **Sensor:** logs are being read
```shell
sudo filebeat test output
sudo journalctl -u filebeat -n 30 --no-pager
```

2. **Logstash:** no pipeline errors
```shell
sudo journalctl -u logstash -n 30 --no-pager
```

3. **Elasticsearch:** documents are landing in the data streams
```shell
curl -k -u elastic "https://localhost:9200/_cat/indices/*suricata*,*zeek*?v"
```
   Row counts (`docs.count`) should climb as traffic flows.

4. **Kibana:** alerts and connections appear on the dashboards in near real time.

---

# Troubleshooting

- **No data in Kibana** — check the chain in order: `filebeat test output` (Pi) →
  Logstash listening on 5044 (`ss -lntp | grep 5044`) → ES index counts. Whichever
  stage is empty is where it's broken.
- **Filebeat won't start / crash-loops** — `journalctl -u filebeat` shows
  `module ... has no enabled filesets`. A fileset is still `enabled: false` in
  `modules.d/suricata.yml` or `zeek.yml`. Set it to `true`, then
  `sudo systemctl reset-failed filebeat && sudo systemctl restart filebeat`
  (the `reset-failed` clears the "start request repeated too quickly" lockout).
- **Logstash flapping: `Elasticsearch Unreachable ... Read timed out` / batch
  "too large to fit into the pre-allocated memory pool"** — Suricata `stats`
  events are flooding the pipeline with 16 MB+ batches. Disable `stats` in
  `suricata.yaml` and truncate `eve.json` (see the Part 5 step 3 note).
- **Data lands in `ecs-logstash-*` or `logs-generic-default`, dashboards empty**
  — the data-stream routing isn't working. Confirm the Logstash output uses
  `data_stream => "true"` (not `"auto"`) plus `data_stream_auto_routing => true`,
  and that the filter copies `[event][dataset]` into `[data_stream][dataset]`
  (Part 4 step 3). Inspect a stored doc to verify:
  `curl -k -u elastic "https://localhost:9200/.ds-logs-*/_search?size=1&pretty"` —
  `data_stream.dataset` should read e.g. `suricata.eve`, not `generic`.
- **`curl` to ES returns 401 `security_exception`** — the password was mistyped
  at the silent prompt, or a `$VAR` you used is empty/unset in this shell (it
  doesn't survive a new SSH session). Re-type it, or `read -s -p "pw: " ESPW`.
- **Zeek data missing, Suricata fine** — almost always the Zeek `var.paths` in
  `modules.d/zeek.yml` don't match the real `/usr/local/zeek/logs/current/` path.
- **Pi can't reach ELK VM** — the mirror destination port can't transmit. Ship
  logs over a separate management interface (USB-Ethernet). Also verify the
  Logstash host address in `filebeat.yml`: a single wrong octet gives
  `dial tcp ...: i/o timeout` and Filebeat then fails silently forever, which
  looks identical to the IDS not detecting anything.
- **TLS / certificate errors** — make sure `http_ca.crt` was copied correctly;
  for the Pi's one-time `setup`, use `ssl.verification_mode: certificate`.
- **Elasticsearch won't start** — check `vm.max_map_count` (Part 1 step 4) and
  that `cluster.initial_master_nodes` was removed. See `journalctl -u elasticsearch`.
- **Timestamps look wrong / out of order** — verify `chrony` is synced on both
  the Pi and the VM (`chronyc tracking`).

---

# Future enhancements
- **Threat-intel enrichment** in Logstash — tag IPs against AbuseIPDB / AlienVault
  OTX feeds, extending the Zeek Intel Framework.
- **Kibana alerting / detection rules** — email or webhook on high-severity
  Suricata alerts.
- **Elastic Agent + Fleet** — migrate to Fleet-managed integrations for
  one-click Suricata/Zeek installs and centralized config if you add more sensors.
- **ILM (Index Lifecycle Management)** — roll over and delete old indices so the
  VM disk doesn't fill up.
