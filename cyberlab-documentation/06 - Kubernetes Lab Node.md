# ☸️ 06 - Kubernetes Lab Node

> [!NOTE]
> In progress. The three-node Talos cluster is built and managed by Terraform. CI with GitHub Actions and the first workloads are next.

## Objective
Stand up a second Proxmox node (`pve1`) and manage it as code: Terraform creates the VMs, Talos Linux runs Kubernetes on them, and GitHub Actions will validate changes. Everything should be reproducible from a git repository rather than clicked together in a web GUI.

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
| GitHub Actions | CI for the infrastructure repo (planned) |
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
| talos-wk-01 | 4021 | Worker | 2 | 6 GB | 40 GB | 10.127.40.21 |
| talos-wk-02 | 4022 | Worker | 2 | 6 GB | 40 GB | 10.127.40.22 |

- **Versions:** Talos v1.14.4, Kubernetes v1.37.0, containerd 2.3.7, flannel CNI
- **VM IDs:** `40` (Lab VLAN) followed by the host octet, so the ID tells you the IP
- **Room to grow:** `.12` – `.13` for a three-node control plane, `.23` – `.29` for more workers
- **Budget on `pve1`:** 6 / 6 vCPU allocated (shared, idle most of the time), 16 / 31 GiB RAM, 120 / 373 GB thin-provisioned disk. RAM is the real limit; about 15 GiB is left for other lab VMs

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

## ➡️ Next Up
- Add GitHub Actions (`terraform fmt` / `validate`) to `cyberlab-infra`
- Back up Terraform state, then move it to a remote backend
- Deploy the first workloads
