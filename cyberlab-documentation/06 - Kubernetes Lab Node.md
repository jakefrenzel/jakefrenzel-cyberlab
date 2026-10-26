# ☸️ 06 - Kubernetes Lab Node

> [!NOTE]
> In progress. The three-node Talos cluster runs Cilium, deploys from git with Argo CD, is monitored by Prometheus and Grafana, keeps data on persistent volumes, runs a default-deny host firewall and Falco runtime threat detection on every node, and sends alerts to Discord.

## Objective
Stand up a second Proxmox node (`pve1`) and manage it as code: Terraform creates the VMs, Talos Linux runs Kubernetes on them, Argo CD deploys workloads from git, and GitHub Actions validates every change. Everything should be reproducible from a git repository rather than clicked together in a web GUI.

## Hardware
| Device | Purpose |
|--------|---------|
| HP EliteDesk | Proxmox VE 9.2 node `pve1` (6 vCPU, 31 GiB RAM, 373 GB storage) |
| Dell Laptop | Admin PC: runs Terraform, `talosctl`, and `kubectl` over Tailscale |

## Stack
| Tool | Role |
|------|------|
| Proxmox VE 9.2 | Hypervisor |
| Terraform | Creates and tracks VMs through the Proxmox API (`bpg/proxmox` provider) |
| Talos Linux | Minimal, API-only Kubernetes OS: no SSH, no shell, configured entirely by machine config (`siderolabs/talos` provider) |
| Cilium | eBPF pod networking: replaces flannel and kube-proxy, enforces NetworkPolicy, Hubble flow visibility, and a host firewall on every node |
| Argo CD | GitOps: deploys everything under `k8s/` in the repo and keeps the cluster matching `main` |
| Prometheus + Grafana | Metrics and dashboards for nodes, Kubernetes, the control plane, and Cilium/Hubble network flows, with a custom overview as the home page |
| Alertmanager | Routes warning and critical alerts to a private Discord channel |
| Falco + Falcosidekick | Runtime threat detection (an eBPF-based host IDS) on every node; Falcosidekick forwards its events to Alertmanager and a web UI |
| Local Path Provisioner | Persistent volumes carved out of each worker's data disk |
| GitHub Actions | CI on every push: `terraform fmt`/`validate`, `tflint`, ShellCheck, gitleaks |
| Tailscale | The only way into the node and the cluster network |

Talos was chosen over k3s because it is configured purely through declarative machine config, which fits a Terraform workflow without needing SSH or configuration scripts.

The Terraform code lives in a separate repository, `cyberlab-infra`, so this repository stays documentation only.

## Where the Node Lives
`pve1` is temporarily on a college network until it moves into the home lab. That network hands out **public, internet-routable IPs** by DHCP with short leases, which shaped every decision on this page:

| Phase | Management IP | Access |
|-------|---------------|--------|
| Now (college) | Public DHCP, changes often | Tailscale only |
| Later (home) | 10.27.10.10 on Management VLAN 10 | Tailscale + local |

> [!IMPORTANT]
> Do not cluster `pve1` with home Proxmox nodes over Tailscale. Corosync (Proxmox clustering) is latency-sensitive and is not designed to run over a WAN link. Cluster once the node is physically home.

## Cluster
| VM | VM ID | Role | vCPU | RAM | Disk | IP |
|----|-------|------|------|-----|------|----|
| talos-cp-01 | 4011 | Control plane | 2 | 4 GB | 40 GB | 10.127.40.11 |
| talos-wk-01 | 4021 | Worker | 2 | 6 GB | 40 GB + 20 GB data | 10.127.40.21 |
| talos-wk-02 | 4022 | Worker | 2 | 6 GB | 40 GB + 20 GB data | 10.127.40.22 |

- **Versions:** Talos v1.14.4, Kubernetes v1.37.0, containerd 2.3.7, Cilium 1.20.4, Argo CD v3.5.6, kube-prometheus-stack 92.13.1, Local Path Provisioner v0.0.37, Falco 0.45.1 (chart 9.2.1), Falcosidekick 2.32.0
- **VM IDs:** `40` (Lab VLAN) followed by the host octet, so the ID tells you the IP
- **Room to grow:** `.12` – `.13` for a three-node control plane, `.23` – `.29` for more workers
- **Budget on `pve1`:** 6 / 6 vCPU allocated (shared, idle most of the time), 16 / 31 GiB RAM, 160 / 373 GB thin-provisioned disk. RAM is the real limit; about 15 GiB is left for other lab VMs

```
                 ┌──────────────────────────────┐
  kubectl ──────►│ talos-cp-01      .11         │  control plane (tainted NoSchedule)
                 │ apiserver · etcd · scheduler │
                 └──────────────┬───────────────┘
                                │ :6443
               ┌────────────────┴────────────────┐
     ┌─────────▼──────────┐           ┌──────────▼─────────┐
     │ talos-wk-01   .21  │           │ talos-wk-02   .22  │  workers: workloads run here
     └────────────────────┘           └────────────────────┘
                    all on vmbr1 (10.127.40.0/24, NAT behind pve1)
```

## Steps taken

### Proxmox install and hardening
1. Set BIOS: VT-x and VT-d on, AHCI storage mode, Fast Boot off, power on after power loss
2. Verified the Proxmox VE 9.2 ISO's SHA256, flashed it with Rufus (DD mode), and installed
3. Switched `vmbr0` to DHCP for the college network
4. Installed Tailscale on the Proxmox host itself, so access does not depend on the changing public IP
5. Enabled the Proxmox firewall at datacenter and node level with input policy **DROP**, then allowed only:
   - `tcp/8006` (web GUI) from the tailnet `100.64.0.0/10`
   - `tcp/22` (SSH) from the tailnet
   - ICMP from the tailnet
   - DHCP client traffic (`udp 67 → 68`) so the node can keep its lease
6. Disabled the enterprise repositories and added the no-subscription repository

### Admin PC tooling
1. Installed Terraform, `talosctl`, and `kubectl` on the admin laptop
2. Confirmed the Proxmox API answers over Tailscale on port 8006
3. Added a `cyberlab` function to the PowerShell profile that sets `KUBECONFIG` and `TALOSCONFIG` to the cluster's credential files for the current session only (`cyberlab -Off` undoes it). The credentials are deliberately not merged into `~/.kube/config`, which also holds Docker Desktop's context

### Terraform API token
Terraform gets its own user and a role with only the privileges it needs, instead of using `root`.

```bash
pveum role add Terraform -privs "Datastore.Allocate Datastore.AllocateSpace Datastore.AllocateTemplate Datastore.Audit Pool.Allocate Pool.Audit Sys.Audit Sys.Console Sys.Modify SDN.Use VM.Allocate VM.Audit VM.Clone VM.Config.CDROM VM.Config.Cloudinit VM.Config.CPU VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Console VM.Migrate VM.PowerMgmt VM.GuestAgent.Audit"
pveum user add terraform@pve
pveum aclmod / -user terraform@pve -role Terraform
pveum user token add terraform@pve tf --privsep 0
```

1. Created the role, user, ACL, and token above
2. Verified the result with `pveum user token permissions terraform@pve tf`
3. Stored the token secret in a password manager and a gitignored `terraform.tfvars`, never in the repository
4. Tested the token from the laptop against `/api2/json/version` before writing any Terraform

> [!NOTE]
> The role has no `User.Modify` or `Permissions.Modify`, so even though it is granted at `/`, the token cannot create users or raise its own privileges.

### Infrastructure repository
1. Created the `cyberlab-infra` repository with a `.gitignore` that excludes Terraform state, `*.tfvars`, `talosconfig`, and `kubeconfig`
2. Pinned providers: `bpg/proxmox ~> 0.118` and `siderolabs/talos ~> 0.12`
3. Wrote a read-only smoke test (Proxmox version and node list data sources)
4. Ran `terraform plan` and confirmed it authenticated and read `pve1` without proposing any changes to real infrastructure
5. Kept every environment-specific value (API endpoint, token, cluster subnet) in the gitignored `terraform.tfvars`, with a committed `terraform.tfvars.example` showing the shape. The repository is public, so committed files only ever contain placeholders

| File | Contents |
|------|----------|
| `providers.tf` | Provider pins and Proxmox connection |
| `variables.tf` | Inputs: endpoint, token, `cluster_cidr` (validated as a /24), Talos version |
| `image.tf` | Talos image: Image Factory schematic and download to `pve1` |
| `nodes.tf` | Cluster topology map and the VMs |
| `talos.tf` | Cluster secrets, machine configs, bootstrap, kubeconfig |
| `host/pve1/setup-vmbr1.sh` | Host network setup that can't be done through the API |

### Private cluster network (`vmbr1`)
If the Talos VMs were bridged to `vmbr0`, they would pull public IPs from the college network and expose the Talos API (`50000`) and Kubernetes API (`6443`) to the internet. Instead, they sit on a private NAT bridge:

```
auto vmbr1
iface vmbr1 inet static
        address 10.127.40.1/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up   iptables -t nat -A POSTROUTING -s 10.127.40.0/24 -o vmbr0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s 10.127.40.0/24 -o vmbr0 -j MASQUERADE
        post-up   iptables -t raw -I PREROUTING -i fwbr+ -j CT --zone 1
        post-down iptables -t raw -D PREROUTING -i fwbr+ -j CT --zone 1
```

- `bridge-ports none`: no physical NIC, the bridge only connects VMs
- `MASQUERADE`: VMs reach the internet through `pve1`, but nothing outside can start a connection to them
- The `fwbr+` conntrack zone lines are from the Proxmox docs and keep NAT working alongside the Proxmox firewall

1. Applied the config with an idempotent script kept in `cyberlab-infra` (`host/pve1/setup-vmbr1.sh`), copied over with `scp` instead of pasted into a terminal
2. Brought up only the new bridge with `ifup vmbr1`, not `ifreload -a`, so a mistake could not take down `vmbr0` on a remote node
3. Verified the bridge address, `ip_forward = 1`, and the `MASQUERADE` rule

### Reaching the cluster network over Tailscale
1. Advertised the subnet from `pve1`: `tailscale set --advertise-routes=10.127.40.0/24`
2. Approved the route in the Tailscale admin console
3. Added the subnet to the existing subnet-router grant in the tailnet policy:

```jsonc
{
    "src": ["autogroup:member"],
    "dst": ["10.27.10.0/24", "10.127.40.0/24"],
    "ip":  ["*"],
},
```

4. Confirmed the laptop received the route and could ping `10.127.40.1`

`src` stays `autogroup:member`, so tagged devices (the honeypot sensor) are still excluded from the new subnet.

### Talos image
Talos images are built to order by [Image Factory](https://factory.talos.dev). A **schematic** describes the customizations and the factory returns a content-addressed ID for it.

1. Created the schematic in Terraform (`talos_image_factory_schematic`) with one extension, `siderolabs/qemu-guest-agent`, so Proxmox can see the VMs' IPs and shut them down cleanly
2. Chose the **`nocloud`** platform: it reads network settings from the Proxmox cloud-init drive, so every node gets a static IP from first boot with no DHCP server on `vmbr1`
3. Chose the **`qcow2`** disk format. It is uncompressed, so Proxmox can import it directly; the `.raw.xz` default would need decompression Proxmox does not support
4. Had `pve1` download the image itself (`proxmox_download_file`, content type `import`) rather than uploading ~235 MB from the laptop

### Control-plane VM
The first apply created only the image and one VM, to prove the VM side worked before adding Talos configuration.

| Setting | Value | Why |
|---------|-------|-----|
| CPU type | `host` | Talos needs x86-64-v2+, which the default `kvm64` doesn't provide |
| Machine / SCSI | `q35`, `virtio-scsi-single` | Modern chipset; allows an I/O thread per disk |
| Disk | 40 GB on `local-lvm`, imported from the Talos image | No installer ISO needed |
| Network | `vmbr1` only | Never exposed to the upstream network |
| IP | cloud-init static, computed as `cidrhost(cluster_cidr, 11)` | No hard-coded addresses in the code |
| Guest agent | On, but Terraform doesn't wait for it | The IP is already known |

1. Applied the saved plan (3 resources: schematic, image download, VM)
2. Confirmed the VM booted into Talos **maintenance mode**, the state where it has no config yet, with `talosctl get addresses --insecure`
3. Checked the install disk (`sda`), the default route via `10.127.40.1`, and that the guest agent extension loaded

> [!WARNING]
> In maintenance mode Talos accepts a machine config from anyone who can reach port `50000`. Keeping the VMs on the private `vmbr1` network is what makes this window safe.

### Talos configuration and bootstrap
Talos 1.14 machine config is split into typed documents (`UnattendedInstallConfig`, `HostnameConfig`, `KubeNodeConfig`, and so on), and Terraform patches them by kind:

| Patch | Why |
|-------|-----|
| `UnattendedInstallConfig`: installer image + disk selector `/dev/sda` | Use the `nocloud` installer from our schematic so upgrades keep the guest agent (the default is the generic `metal` installer) |
| Delete the generated `HostnameConfig` | It sets `auto: stable` (a generated hostname), which can't coexist with a fixed hostname |
| New `HostnameConfig` with the VM's name | Hostnames match Proxmox VM names |

1. Generated a sample config with `talosctl gen config` and tested each patch locally with `talosctl validate` before putting it in Terraform
2. Applied the second stage: cluster secrets, machine config, bootstrap, and kubeconfig (4 resources)
3. Ran `talosctl health` and `kubectl get nodes`: `talos-cp-01` was `Ready` with every control-plane component running

> [!IMPORTANT]
> Terraform state now holds the cluster's root certificate authority. It is gitignored and must be backed up; losing it means losing admin access to the cluster.

### Workers
Instead of copying the VM block per node, the cluster is described as one map and every resource loops over it with `for_each`:

```hcl
nodes = {
  talos-cp-01 = { role = "controlplane", host = 11, vm_id = 4011, cores = 2, memory = 4096 }
  talos-wk-01 = { role = "worker", host = 21, vm_id = 4021, cores = 2, memory = 6144 }
  talos-wk-02 = { role = "worker", host = 22, vm_id = 4022, cores = 2, memory = 6144 }
}
```

Moving the control plane into the map changed its Terraform address (`vm.cp` → `vm.node["talos-cp-01"]`), which Terraform would read as "destroy and recreate". `moved` blocks record the rename:

```hcl
moved {
  from = proxmox_virtual_environment_vm.cp
  to   = proxmox_virtual_environment_vm.node["talos-cp-01"]
}
```

1. Validated the same patches against a generated **worker** config locally
2. Planned and confirmed `4 to add, 0 to change, 0 to destroy`, with the control plane only listed as moved
3. Applied: two VMs created and given worker configs, which join through the control plane's API using the shared cluster secrets. No bootstrap step; that only runs once per cluster
4. Both workers were `Ready` about a minute after booting

### Verification
```
NAME          STATUS   ROLES           VERSION   INTERNAL-IP
talos-cp-01   Ready    control-plane   v1.37.0   10.127.40.11
talos-wk-01   Ready    <none>          v1.37.0   10.127.40.21
talos-wk-02   Ready    <none>          v1.37.0   10.127.40.22
```

- `talosctl health`: all 16 checks passed, including a single etcd member on the control plane
- A test `nginx` pod was scheduled on `talos-wk-02`, not the control plane, so the `NoSchedule` taint works. It got an IP from the pod CIDR (`10.244.0.0/16`)

### CI with GitHub Actions
`cyberlab-infra` is public, so CI only does static checks. GitHub's runners can't reach the lab, and no Proxmox token or Tailscale key is ever given to a public repository's workflows.

| Job | Checks |
|-----|--------|
| Terraform | `fmt -check`, `init -backend=false`, `validate`, `tflint` (recommended preset) |
| ShellCheck | Every `*.sh` in the repo |
| Secret scan | gitleaks over the full git history |

1. Pinned every action to a commit SHA instead of a tag, so a moved tag can't change what runs. The workflow token is read-only
2. Ran every check locally before the first push so the first CI run would be green
3. Tested gitleaks against the whole working tree, including gitignored files: it flagged 149 secrets (private keys and API-key-like values) in the state and credential files, proving `.gitignore` is what keeps them out of git
4. Found that gitleaks' default rules **don't recognize Proxmox API tokens**, and added a custom rule (`user@realm!tokenid=<uuid>`) with an allowlist for the all-zero placeholder in the example file
5. Before going public, audited exactly what would be committed: no state, tfvars, or credentials, and no real IPs, hostnames, or usernames in any file

### State backups
Terraform state holds the cluster's root CA, so it is backed up **encrypted**:

1. Encrypted with [age](https://github.com/FiloSottile/age) to a public key, so taking a backup needs no passphrase; only the private key can decrypt
2. Each backup bundles the state, `terraform.tfvars`, `talosconfig`, and `kubeconfig`, written to a cloud-synced folder; the newest 10 are kept
3. The private key is stored in a password manager, so losing the laptop doesn't lose the backups
4. Tested a full restore: every file came back byte-identical and Terraform read all resources from the restored state

### Cilium
Flannel, the Talos default, **doesn't enforce NetworkPolicy**, so pods can't be firewalled from each other. The cluster was rebuilt on Cilium while it was still empty.

| Talos 1.14 patch | Effect |
|------------------|--------|
| Delete the `KubeFlannelCNIConfig` document | No default CNI |
| `KubeProxyConfig` with `enabled: false` | No kube-proxy; Cilium's eBPF replacement handles Services |
| `KubeInlineManifestConfig` named `cilium` | Talos applies Cilium during bootstrap |

1. Rendered the Cilium chart inside Terraform with the Helm provider's `helm_template` data source, which never connects to the cluster, using the Talos guide's settings: Kubernetes IPAM, kube-proxy replacement through KubePrism (`localhost:7445`), `SYS_MODULE` dropped from the capabilities (Talos forbids kernel module loading), and cgroup/bpffs auto-mounting off
2. Set Hubble's TLS certificates to be generated in-cluster. The default generates new random certificates on every render, which would change the machine config on every plan
3. Planned twice and compared the rendered configs to prove the output is deterministic, then validated it with `talosctl validate`
4. Destroyed only the VMs (`terraform destroy -target`), keeping the Talos image and the cluster secrets, so the rebuilt cluster kept the same CA and the existing `talosconfig` kept working
5. Verified: `cilium-dbg status` healthy on all nodes, `KubeProxyReplacement: True`, no `kube-proxy` DaemonSet, and Hubble recording flows
6. Tested enforcement: a client pod could fetch a page from an nginx pod, then timed out after a default-deny ingress NetworkPolicy was applied

### Argo CD (GitOps)
Argo CD is installed by Talos at bootstrap the same way as Cilium, following the Talos "Deploy Argo CD" guide. From then on it owns everything under `k8s/` in the repository.

```
Terraform ──► Talos inline manifests ──► Cilium + Argo CD
                                              │
                                              ▼
                     root Application ──► k8s/apps/*.yaml ──► each app
```

1. Rendered the `argo-cd` chart with `helm_template` (Dex and notifications disabled) and bundled it with the `argocd` namespace and a `root` Application
2. The `root` Application is an **app of apps**: it syncs `k8s/apps` from `main`, where every file is another Application. Sync is automated with `prune` and `selfHeal`, so manual changes are reverted and deleting a file removes its app
3. Gave `root` no finalizer, so deleting it can never cascade into the apps it created; child apps do have one, so removing an app's file cleans up its resources
4. Relied on Talos' documented manifest ordering (Namespaces, then CRDs, then everything else) so the `root` Application is created after its CRD
5. The admin password is generated in-cluster and was changed after the first login; it never appears in git or Terraform state
6. Verified: `root` reached `Synced` / `Healthy` at the same commit CI had just checked

### Monitoring
kube-prometheus-stack is the first workload deployed purely through Argo CD.

| File in `k8s/` | Purpose |
|----------------|---------|
| `apps/monitoring-config.yaml` | Syncs `k8s/monitoring/` first (sync wave `-1`) |
| `apps/kube-prometheus-stack.yaml` | The Helm chart, with server-side apply (its CRDs are too large for client-side apply) |
| `monitoring/namespace.yaml` | `monitoring` namespace with Pod Security `privileged`, because node-exporter needs host access |
| `monitoring/cilium-monitors.yaml` | Scrape configs for Cilium and Hubble |

1. Disabled scraping for the scheduler, controller manager, and etcd (Talos binds their metrics to localhost) and for kube-proxy (not running), so no targets sit permanently down. The first three were turned back on later (see [Control-plane metrics](#control-plane-metrics))
2. Pointed Grafana's admin login at a hand-made Secret. The chart default (`prom-operator`) is publicly known, and a chart-generated random password would change on every Argo CD render. Restarting Grafana after updating the Secret proved the password survives restarts
3. Turned on Cilium agent, operator, and Hubble metrics and Cilium's bundled dashboards in `cilium.tf`. The chart only creates Services for Hubble and Envoy, so the agent and operator are scraped with PodMonitors
4. Copied the `k8s_app` and `io_cilium_app` pod labels onto the scraped series, because Cilium's dashboards filter on them (113 and 6 queries respectively)
5. Verified: 32 of 32 targets up, Hubble processing about 34 flows per second, 6 Cilium/Hubble dashboards loaded, and only the always-on `Watchdog` alert firing

> [!IMPORTANT]
> Talos treats inline manifests as **create-only**: after an object exists, Talos never updates or deletes it. Changing Cilium's settings created the new objects but left the running Cilium pods untouched. Updating them takes `talosctl upgrade-k8s --to <current version>`, which syncs manifests (server-side apply, with pruning) without upgrading Kubernetes. Check the diff with `--dry-run` first.

### Persistent storage
Without persistent volumes, Prometheus lost its history and Grafana its database on every restart. The volumes live on a second disk on each worker, kept apart from the Talos system disk.

| Layer | What it does |
|-------|--------------|
| `nodes.tf` | Adds a 20 GB data disk to each worker (`data_disk` in the node map; `0` for the control plane, which runs no workloads) |
| `storage.tf` | Talos `UserVolumeConfig` named `local-path-provisioner`: formats the data disk (XFS) and mounts it at `/var/mnt/local-path-provisioner` |
| `k8s/storage/local-path/` | Local Path Provisioner v0.0.37 (Argo CD, sync wave `-2`), pointed at that mount and set as the default StorageClass |

1. Selected the data disk with `!system_disk && disk.transport == 'virtio'`, so neither the system disk nor the cloud-init CD-ROM (SATA) could ever be picked
2. Patched Local Path Provisioner per the Talos "Local Storage" guide: its default `/opt` path is read-only on Talos. Also pinned its helper pod's `busybox` image (upstream uses an untagged one) and labelled its namespace Pod Security `privileged`, since the helper pods mount host paths
3. Gave the monitoring stack volumes: Prometheus 10 Gi (15 days of retention, capped at 8 GiB so it can never fill the volume), Alertmanager 1 Gi, Grafana 2 Gi
4. Verified: `talosctl get volumestatus` shows the volume `ready` on both workers, every claim is `Bound`, and Prometheus kept its history through two pod restarts and a control-plane reboot

> [!NOTE]
> A local volume exists on one node only, so the pod using it is pinned there: Prometheus and Alertmanager always run on `talos-wk-01`, Grafana on `talos-wk-02`. If a worker dies, its volumes go with it. That's an acceptable trade-off for monitoring data in a lab, but not for anything that can't be rebuilt.

### Control-plane metrics
Talos binds the scheduler, controller manager, and etcd metrics to localhost, so Prometheus couldn't scrape them.

| Talos 1.14 patch (`metrics.tf`, control plane only) | Effect |
|-----------------------------------------------------|--------|
| `KubeSchedulerConfig` `extraArgs.bind-address: 0.0.0.0` | Scheduler metrics on `:10259` (HTTPS, token required) |
| `KubeControllerManagerConfig` `extraArgs.bind-address: 0.0.0.0` | Controller-manager metrics on `:10257` (HTTPS, token required) |
| `cluster.etcd.extraArgs.listen-metrics-urls: http://0.0.0.0:2381` | etcd metrics on `:2381` (plain HTTP, metrics only) |

1. Generated a sample config and validated the patch locally. etcd has no typed document yet in Talos 1.14, so its setting still goes in the legacy `v1alpha1` document
2. Applied it with Terraform. The scheduler and controller manager restarted on their own within a minute, but etcd only reads its flags at startup, and Talos refuses to restart etcd through its API, so the control plane needed a reboot (about two minutes without the Kubernetes API; workloads on the workers kept running)
3. Re-enabled the three exporters in kube-prometheus-stack. The Talos static pods carry the same `component` labels as kubeadm's, so the chart defaults found the scheduler and controller manager as-is
4. etcd runs as a Talos system service, not a pod, so there is nothing for a Service to select. Instead, its Service selects the **kube-apiserver** pod: it uses host networking, so its address is the node's IP, and Prometheus scrapes `:2381` there. This needs no IP addresses in the public repository and follows the control plane if more nodes are added
5. Stopped Prometheus from sending its service-account token to etcd's plain-HTTP endpoint, which needs no authentication
6. Verified: 35 of 35 targets up (was 32), data in the etcd, scheduler, and controller-manager dashboards, and no new alerts

> [!WARNING]
> This made the three ports reachable from every pod and over the Tailscale route, and etcd's `:2381` has no authentication. The host firewall below closes that.

### Host firewall
Cilium can also filter traffic **to the nodes themselves**, with the same identity-based policies it uses for pods. Every node now drops incoming traffic that isn't explicitly allowed:

| From | Allowed |
|------|---------|
| Other nodes, the API server, Cilium health checks | Everything (VXLAN, kubelet, Talos API proxying, health checks) |
| Outside the cluster (the admin laptop) | Kubernetes API `6443`, Talos API `50000`, ping |
| Any pod | API server `6443`, Talos host DNS `53` (CoreDNS forwards to it) |
| Prometheus pod only | kubelet, node-exporter, and Cilium/Hubble metrics on every node; etcd `2381`, controller manager `10257`, and scheduler `10259` on the control plane |
| Hubble Relay pod only | Hubble `4244` |

Rules name pods by their labels, not their addresses, so "Prometheus only" holds wherever Prometheus is scheduled, and the policy files contain no IP addresses. Outgoing traffic from the nodes isn't filtered.

1. Enabled the feature in `cilium.tf` (`hostFirewall.enabled`), synced it with `talosctl upgrade-k8s`, and restarted the Cilium pods, which the chart doesn't do on its own when a setting changes. With no policy selecting a node, nothing was filtered yet
2. Before writing rules, checked which address the traffic actually comes from with Hubble:
   - The laptop's traffic arrives from `pve1`'s bridge address (`10.127.40.1`), because the Tailscale subnet router rewrites the source. Cilium sees it as `world`, which is why only the two APIs are open to `world`
   - Prometheus' scrapes of another node keep the Prometheus pod's own identity (they aren't rewritten to the node's address), so a "Prometheus only" rule can work
3. Wrote two `CiliumClusterwideNetworkPolicy` objects (`k8s/security/host-firewall.yaml`, deployed by Argo CD). They apply only to nodes labelled `cyberlab.io/host-firewall=enabled`, so pushing them changed nothing
4. Rolled out one node at a time: `talos-wk-02`, `talos-wk-01`, then the control plane. For each node:
   1. Put the node's host endpoint into **audit mode** (would-be drops are logged, nothing is dropped) and added the label
   2. Watched the policy verdicts (`cilium-dbg monitor -t policy-verdict`) while generating normal traffic: `kubectl logs`, `talosctl`, ping, Hubble, and a restart of Prometheus and Hubble Relay to force fresh connections
   3. Ran a negative test: a plain `busybox` pod connecting to node-exporter and the kubelet (and etcd on the control plane) had to show up as `audit`
   4. Switched audit mode off and repeated the checks
5. Moved the label into the Talos config (`KubeNodeConfig` labels in `talos.tf`), so a rebuilt node comes up firewalled
6. Verified: a plain pod's connections to etcd `:2381`, the scheduler, node-exporter, and the kubelet are dropped (`Policy denied`); `kubectl`, `talosctl`, Hubble Relay, and Cilium health all still work; Prometheus is 35 of 35

> [!IMPORTANT]
> Audit mode is the emergency off switch: it takes effect immediately on one node. Removing the node's label or deleting the policy by hand doesn't stick, because Talos restores the label within seconds and Argo CD recreates the policy. Audit mode is lost if that node's Cilium agent restarts, so the real fix always goes through git.

### Runtime security (Falco)
The host firewall and Pod Security Admission **prevent**: they block traffic and pod settings before anything happens. Falco **detects**: an eBPF program in each node's kernel sees every syscall (about a million every five minutes per node), and Falco matches them against rules for suspicious behaviour, such as a shell spawned in a container, a read of `/etc/shadow`, logs being cleared, or writes below system directories. Each match becomes an event naming the process, pod, and namespace. It's a host-based IDS that understands containers, the counterpart to the Suricata and Zeek network IDS in [05 - Intrusion Detection System](<05 - Intrusion Detection System.md>): that one sees packets on the wire, this one sees what processes do on the nodes.

| Event goes to | What |
|---------------|------|
| Falco's log (JSON) | Every event |
| Falcosidekick UI | Every event, kept 7 days in Redis (1 Gi volume) |
| Alertmanager | Warning and above (held back from Discord until the rules are tuned) |
| Prometheus + Grafana | Event counts per rule, plus Falco's own CPU, memory, and dropped events |

1. Checked the kernel before picking a driver. Talos only loads signed kernel modules (`module.sig_enforce=1`), so Falco's kernel module was out. The kernel (6.18) exposes BTF type information and has kernel lockdown off, which is what Falco's **modern eBPF** probe needs: it's built into Falco, with no driver to download or compile
2. Ran it with four capabilities (`BPF`, `PERFMON`, `SYS_RESOURCE`, `SYS_PTRACE`) instead of as a privileged container. The `falco` namespace still needs Pod Security `privileged` for the host mounts; Argo CD creates the namespace with those labels (`managedNamespaceMetadata`), so it needs no manifest of its own
3. Read what the chart would mount before deploying. It mounts the directory of every container-engine socket it knows about (Docker, Podman, CRI-O, ...) as a hostPath with no type check, so the kubelet would have created empty directories for all of them on every node. Narrowed it to containerd's CRI socket, the only one Talos has; Falco reads pod names and namespaces from it
4. Rendered the chart locally with `helm template` and checked the output: the capabilities, the mounts, and Falcosidekick posting to Alertmanager's v2 API (v1 is gone in current Alertmanager)
5. Gave the Falcosidekick UI a login from a hand-made Secret; the chart default is `admin:admin`. Verified the default is rejected
6. Turned on Falco and Falcosidekick metrics and their Grafana dashboards. Both charts name their dashboard file `falco-dashboard.json`, so Grafana's sidecar kept overwriting one with the other; a folder each fixed it
7. Verified: read `/etc/shadow` from a throwaway pod, and a `Read sensitive file untrusted` event with the pod's name showed up in the UI and in Alertmanager within seconds. Falco uses 50–65 MiB and about 1% of a core per node with zero dropped syscalls, and Prometheus is 39 of 39 targets up

> [!NOTE]
> The first false positive came from the platform itself. When a pod restarted, containerd unpacked an image layer containing `/var/log/dpkg.log` onto the node, which looks like log tampering to the `Clear Log Activities` rule. It and anything else noisy get tuned after a few days of baseline, so the rules are tuned against real data and not guesses. Until then, Falco events don't go to Discord.

### Overview dashboard
The charts installed 36 Grafana dashboards, but Grafana opened on an empty Home page, and nothing showed the lab as a whole. **Cyberlab Overview** is now the home page:

| Section | Shows |
|---------|-------|
| Cluster | Nodes not ready, alerts firing, scrape targets down, pods not running, restarts, nearly full volumes; node CPU, memory, and disk |
| Network | Policy drops (host firewall and NetworkPolicy), drops by reason, Hubble flows by verdict, DNS responses |
| Security | Falco events (24 h) and a table by rule; Falco nodes reporting, dropped syscalls, Falcosidekick delivery errors |

Every count at the top of a section is **zero when healthy**, so a problem shows up as colour. Panel links open the detailed dashboard behind each number.

1. Kept the dashboard in git as plain JSON (`k8s/monitoring/dashboards/`). Kustomize turns it into a ConfigMap that Grafana's sidecar loads, and Grafana's `default_home_dashboard_path` points at it. It's read-only in the UI; changes go through git like everything else
2. Fixed a counting problem on the way. Falco's event counters only exist once a rule fires, and Prometheus' `increase()` needs an earlier sample to compare against, so every rule's first event on a node counted as zero. The stock Falco dashboard showed nothing for the real events. The overview subtracts the value 24 hours ago instead, falling back to the whole counter for newer series, so every event counts
3. Verified: ran all 20 panel queries against live data before pushing; the 24-hour Falco count showed the 4 real events the stock dashboard showed as 0

### Alerting to Discord
Until now Alertmanager sent every alert to a `null` receiver, so alerts only existed if someone opened its UI. They now go to a private Discord channel, visible only to a `Blue Team` role:

| Alert | Goes to |
|-------|---------|
| Warning and critical | Discord: grouped by namespace, reminded every 12 hours while firing, and a message when resolved |
| `Watchdog` | Muted: it fires all the time by design, to prove the alert pipeline works |
| `InfoInhibitor`, `severity=info` | Muted: informational, for dashboards |
| Falco (`source=falco`) | Muted until the rules are tuned |

1. Kept the webhook URL out of git. Anyone with it can post to the channel, so it lives in a hand-made Secret that Alertmanager mounts as a file (`webhook_url_file`), and it never appears in the repository or the rendered config
2. Validated the new config inside the running Alertmanager pod before pushing: `amtool check-config`, and `amtool config routes test` with sample alerts for each row of the table
3. Verified: a test alert (`amtool alert add`) arrived in Discord about 30 seconds later, followed by a resolved message

> [!IMPORTANT]
> Alertmanager mounts that Secret as a volume, so if it's missing the pod can't start, and alerting goes down with it. After a rebuild, create the Secret before the first sync.

## Choosing the Cluster Subnet
The subnet must not overlap any network the admin laptop or the cluster already uses:

| In use | Range |
|--------|-------|
| Home lab VLANs | 10.27.0.0/16 |
| Tailscale | 100.64.0.0/10 |
| Docker Desktop on the laptop | 172.17.0.0/16+, 192.168.65.0/24 |
| Podman default | 10.88.0.0/16 |
| Kubernetes pods (Talos default) | 10.244.0.0/16 |
| **Kubernetes services (Talos default)** | **10.96.0.0/12** |

The first subnet picked was `10.99.40.0/24`. It looked free, but `10.96.0.0/12` covers **10.96.0.0 – 10.111.255.255**, so every node would have had service-CIDR routes covering real VM addresses. It was caught before any VM existed and replaced with `10.127.40.0/24`. The third octet matches the Lab VLAN (40), following the convention from [02 - IP Schemes](<02 - IP Schemes.md>).

## Notes & Observations
1. A public IP on the management interface changes the threat model completely. Putting Tailscale on the host and dropping everything else at the Proxmox firewall made the node safe to leave on a network I do not control
2. Testing the API token with a single `curl` before writing Terraform separated "auth is broken" from "my Terraform is broken"
3. Secrets end up in shell history easily. The token secret was briefly pasted into a command line instead of the masked prompt, so the history lines were removed and the token was rotated. Reading secrets with `Read-Host -MaskInput` keeps them out of history
4. Check new subnets against Kubernetes' **service** CIDR, not just the pod CIDR. `10.96.0.0/12` is much larger than it looks
5. A Tailscale grant with `dst: autogroup:member` only matches devices' `100.x` addresses. Every subnet behind a subnet router has to be listed explicitly, and a route that is approved but not granted is silently never sent to clients
6. Running scripts with `scp` + `ssh` is more reliable than pasting multi-line blocks, which break on terminal line wrapping
7. Some keyboards are not recognized by the boot menu before an OS loads. A firmware update (Ducky One 2 Mini, firmware V1.22) fixed it
8. Building in stages (image + one VM, then Talos config, then workers) meant every failure would have had one obvious cause
9. Validate config patches locally first. Patching only the installer image in `UnattendedInstallConfig` silently dropped the disk selector; `talosctl validate` caught it before it reached a node
10. Provider resources get renamed between versions (`proxmox_virtual_environment_version` → `proxmox_version`, `..._download_file` → `proxmox_download_file`). Deprecation warnings from `terraform validate` are worth acting on immediately
11. `moved` blocks make refactoring safe: always check the plan says `0 to destroy` before applying a rename
12. Talos enables Pod Security Admission by default: pods that break the "baseline" level are blocked, and anything below "restricted" gets a warning. A bare `kubectl run nginx` triggers the warning, a useful reminder to write workloads that pass "restricted"
13. Windows PowerShell 5.1's `>` writes UTF-16, which `talosctl` can't read. Exporting Terraform outputs with `Set-Content -Encoding ascii` avoids it
14. Secret scanners only know the formats in their rules. Testing gitleaks against real (gitignored) secrets showed it missed the Proxmox token entirely until a custom rule was added
15. Swap the CNI while the cluster is empty. With everything in Terraform, a rebuild took about a minute of apply time plus a few minutes to go healthy
16. Anything rendered into machine config must be deterministic. Charts that generate random certificates or passwords at render time cause a change on every plan, so those values are generated in-cluster instead
17. `terraform console` shows values from state, not freshly read data sources, so it showed the old config after a change. Inspecting the plan's JSON (`terraform show -json`) shows what will actually be applied
18. Cilium's kube-proxy replacement answers "no route to host" for a Service with no ready endpoints. Argo CD's first sync hit this for a few seconds while its repo server was still starting
19. Talos inline manifests are create-only. Configuration changes to components it installs need `talosctl upgrade-k8s` at the same version to reach the cluster
20. With no persistent storage, Grafana's database resets on every restart, so a password changed in the UI reverts to the Secret's value. Keeping the Secret in step with the real password makes it survive restarts
21. Applying a Talos config change doesn't mean every component picked it up. The scheduler and controller manager restarted themselves, but etcd needed a reboot. Checking the listening ports (`talosctl netstat -l`) showed which ones hadn't changed
22. Not everything is a pod. etcd on Talos is a host service, so the usual "Service selects pods" pattern needed a stand-in: a host-network pod on the same node
23. A firewall rule is only as good as your picture of the traffic. Hubble showed the laptop arriving as `pve1`'s address and Prometheus keeping its pod identity across nodes, which decided how the rules were written
24. Cilium logs a policy verdict only when a connection opens. Long-lived connections made before a rule existed never show up, so the audit included restarting Prometheus and Hubble Relay to watch their fresh connections
25. A negative test proves a firewall works. Seeing a plain pod blocked from etcd shows the rules do something, not just that nothing broke
26. Rolling out node by node, starting with a worker, kept every mistake small and left the control plane for last, after the rules had been proven twice
27. An escape hatch has to be tested too. Removing the node label looked like the obvious way out, but Talos put it back within five seconds once it owned the label
28. Changing a Cilium Helm value updates its ConfigMap but not the running agents; they read it at startup. Restarting the DaemonSet applies it
29. Check the kernel before picking a security tool's driver. Talos's signed-modules-only policy ruled out Falco's kernel module, but BTF and no lockdown made the eBPF probe work with four capabilities instead of full privilege
30. Read what a chart mounts, not just what it runs. Falco's defaults would have created empty directories on every node for container engines Talos doesn't have
31. Two charts can collide where you don't expect it. Falco's and Falcosidekick's dashboards had the same filename, and the Grafana sidecar's logs (writing the same file every minute) were the giveaway
32. Prometheus' `increase()` can't see a counter's first value. Event counters that only appear when something happens need a different query, or the first event of each kind never shows up
33. Baseline before tuning. The first false positive was the platform's own image unpacking, which no amount of guessing up front would have predicted
34. A `kubectl port-forward` is tied to one pod. Restarting the pod ends it, and the browser just reports a network error
35. In an app-of-apps setup, a child Application's spec is synced by `root`. Refreshing the child alone keeps using its old spec
36. Mounting a credential as a file (`webhook_url_file`) keeps it out of git and out of the rendered config, but it also makes the Secret a hard dependency for the pod to start
37. Validate config where it will run. `amtool` inside the running Alertmanager pod checked the syntax and every route before the change was pushed

## ➡️ Next Up
- Tune Falco against a few days of baseline, then send its alerts to Discord
- Move Terraform state to a remote backend (encrypted, with locking)
- Deploy an app of my own end to end: build in GitHub Actions, publish to GHCR, deploy with Argo CD
