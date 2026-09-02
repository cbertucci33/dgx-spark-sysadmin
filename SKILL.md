---
name: dgx-spark-sysadmin
description: Administer DGX Spark hosts safely through remote access.
metadata:
  openclaw:
    requires:
      bins: [ssh]
---

# DGX Spark Remote Sysadmin

Administer NVIDIA DGX Spark / GB10 systems remotely. Keep management, ConnectX-7 fabric, and client service traffic distinct; discover live state before changes; preserve recovery access; verify every affected node and real workload path.

## When to Use

Use for SSH and Tailscale setup, multi-node administration, ConnectX-7 networking, monitoring, headless operation, workload/host restarts, reboot/shutdown, and coordinated DGX OS, driver, CUDA, firmware, or container-stack updates.

Do not improvise firmware/UEFI recovery without a console or operate systems outside the user's authorization.

## Operating Contract

Resolve rather than invent:

```text
<admin-user>  <ssh-key-path>
<node-a-management-name>  <node-b-management-name>
<node-a-fabric-address>   <node-b-fabric-address>
<node-a-fabric-interface> <node-b-fabric-interface>
<node-a-roce-device>      <node-b-roce-device>
<service-or-container>
```

For each additional rail, define another complete fabric tuple.

| Plane | Purpose |
|---|---|
| Management | LAN, Wi-Fi, or Tailscale for administration and recovery |
| Fabric | ConnectX-7 Ethernet/RoCE for node-to-node workloads |
| Service | Address clients use for the application/API |

The fabric must not become a default route or the only recovery path.

### Change rules

1. Read-only discovery may run in parallel. Preserve per-host stdout, stderr, duration, and exit code.
2. Obtain explicit maintenance authorization before persistent network changes, GUI disablement, updates, reboot, shutdown, or other disruption. State scope, interruption, success gates, and rollback.
3. Make one state change at a time during access/network recovery. Back up edited files and syntax-check before activation.
4. Never handle passwords, private keys, Tailscale auth keys, or tokens in prompts, command arguments, logs, or files. Transfer only public keys through trusted paths.
5. Never reboot both nodes together. Treat partial multi-host success as failure until reconciled.
6. Distinguish **started**, **ready**, and **functionally healthy**. A process, port, or health endpoint alone is not full verification.

## 1. Inventory and Parity

Collect the same timestamped manifest from every node and compare it programmatically:

```bash
hostnamectl; cat /etc/os-release; cat /etc/dgx-release 2>/dev/null || true
uname -a; uptime; who; systemctl is-system-running; systemctl get-default
ip -br link; ip -br address; ip route; ip rule
ibdev2netdev 2>/dev/null || true; rdma link show 2>/dev/null || true
nvidia-smi; cat /proc/driver/nvidia/version 2>/dev/null || true
free -h; cat /proc/pressure/memory; swapon --show
findmnt; lsblk; df -hT
systemctl --failed --no-pager; systemctl --type=service --state=running --no-pager
ss -lntup; docker ps --no-trunc 2>/dev/null || true
readlink -f /usr/local/cuda 2>/dev/null || true; nvcc --version 2>/dev/null || true
dpkg-query -W 'nvidia*' 'cuda*' 'linux-*nvidia*' 'libnccl*' 2>/dev/null || true
apt-mark showhold; dpkg --audit
fwupdmgr get-devices 2>/dev/null || true; tailscale status 2>/dev/null || true
```

List differences in OS, kernel, loaded/installed driver, firmware, CUDA, NCCL/RDMA, container runtime, fabric, package health, and workload artifacts. Matching containers do not compensate for divergent hosts.

## 2. Bootstrap SSH

Initial access needs a trusted path: local console, NVIDIA Sync, existing SSH, or enterprise provisioning. An agent cannot bypass first-login or password prompts.

Create a dedicated key only when an approved one does not exist:

```bash
ssh-keygen -t ed25519 -a 100 -f <ssh-key-path> -C dgx-spark-admin
```

Install only its public key through the trusted path. On each node:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Verify noninteractive access and intended sudo policy:

```bash
ssh -o BatchMode=yes -o ConnectTimeout=10 -i <ssh-key-path> \
  <admin-user>@<node-management-name> 'id; hostname; sudo -n true'
```

Before SSH configuration changes, keep an independent session and recovery path open. Inspect `sudo sshd -T`, back up the active configuration, run `sudo sshd -t`, then `sudo systemctl reload ssh`. Disable older authentication only after the replacement succeeds independently. Treat changed host keys as a security/reimage event; verify fingerprints out of band rather than deleting records blindly.

## 3. Tailscale Management

Use Tailscale as an alternate management plane, not the distributed data plane.

1. Follow current official Linux instructions for the installed DGX OS release; prefer reviewed repository/package steps over piping an unreviewed script into a privileged shell.
2. Enable `tailscaled` and authenticate interactively or through an approved secret channel. Never expose auth keys.
3. Prefer standard OpenSSH over Tailscale/MagicDNS. Enable Tailscale SSH only after separate tailnet-policy review.
4. Verify:

```bash
systemctl is-active tailscaled
tailscale status
tailscale ip
tailscale ping --until-direct <peer-management-name>
```

Record direct versus relayed connectivity. Do not advertise the fabric as a subnet route unless the topology and ACLs require it. Confirm MagicDNS does not break local cluster resolution. Prove both primary management and Tailscale access before risky work.

## 4. Connect Two Systems over ConnectX-7

Prefer NVIDIA Sync Cluster Assistant or NVIDIA's current topology-specific playbook. Use manual setup only when needed and remain connected through the management plane.

### 4.1 Discover mappings

Never guess interface names: one physical QSFP port can expose multiple Ethernet/RoCE devices.

```bash
ibdev2netdev
ip -br link
rdma link show
lspci -nn | grep -i -E 'ethernet|network|mellanox|nvidia'
ethtool <fabric-interface>
```

For every rail, record physical port/cable, Ethernet interface, RoCE device, PCIe identity, link speed/state, MTU, addresses, and routes. Use recommended cables and the supplied power adapter.

### 4.2 Configure both endpoints

Back up the authoritative NetworkManager/netplan configuration. Each endpoint needs:

```text
interface = discovered ConnectX Ethernet interface
address/prefix = approved unique fabric endpoint
method = manual
never-default = true
autoconnect = true
MTU = identical at both ends
DNS = none unless explicitly required
```

Do not mix persistent NetworkManager/netplan ownership with undocumented ad-hoc `ip` configuration. Syntax-check netplan before applying. Apply one endpoint at a time through management access and retain rollback commands. Give separate rails unambiguous layer-3 prefixes unless the supported playbook specifies another topology.

### 4.3 Prove two-way layer-3 traffic

On A:

```bash
ip route get <node-b-fabric-address>
ping -I <node-a-fabric-interface> -c 3 <node-b-fabric-address>
```

On B:

```bash
ip route get <node-a-fabric-address>
ping -I <node-b-fabric-interface> -c 3 <node-a-fabric-address>
```

Each route must select the intended interface and local fabric source. An unbound ping can silently use management networking.

For asymmetry, compare both nodes rather than weakening security globally:

```bash
ip address show dev <fabric-interface>; ip route; ip rule
ip neigh show dev <fabric-interface>
sysctl net.ipv4.conf.all.rp_filter
sysctl net.ipv4.conf.<fabric-interface>.rp_filter
sudo nft list ruleset 2>/dev/null || true
sudo ufw status verbose 2>/dev/null || true
ip -s link show dev <fabric-interface>
ethtool -S <fabric-interface>
```

Correct the exact route, reverse-path, or firewall cause.

### 4.4 Prove two-way SSH and application traffic

Controller-to-node SSH does not prove node-to-node launch capability. Use NVIDIA Sync's cluster-key workflow or create dedicated cluster keys on both nodes and exchange only public keys.

```bash
# From A
ssh -o BatchMode=yes -o ConnectTimeout=10 \
  <admin-user>@<node-b-fabric-address> 'hostname'
# From B
ssh -o BatchMode=yes -o ConnectTimeout=10 \
  <admin-user>@<node-a-fabric-address> 'hostname'
```

Also bind an approved temporary TCP listener to each fabric endpoint in turn and verify a real payload from the peer. Remove test listeners and temporary firewall rules afterward.

### 4.5 Benchmark both directions and every rail

Run `iperf3` bound to fabric addresses. Measure A-to-B and B-to-A separately (`--reverse` may test the return direction). Record sender/receiver throughput, retransmits, CPU, MTU, and link errors/drops before and after. Repeat with explicit source binding for every rail. Advertised link speed is not measured throughput.

Verify RoCE/GID state, then run supported RDMA bandwidth tests with each endpoint in server and client roles:

```bash
rdma link show
ibdev2netdev
show_gids 2>/dev/null || true
```

Finally run an NCCL collective using the real CUDA/container stack. Pin only devices already validated:

```text
NCCL_SOCKET_IFNAME=<fabric-interface>
NCCL_IB_HCA=<roce-device>
NCCL_IB_DISABLE=0
```

Validate every rail independently before multi-rail use. NCCL `INFO` plugin probes are not failures without a failed collective, fallback, timeout, or error evidence.

Fabric acceptance requires **both directions** for bound ICMP, SSH, TCP payload, `iperf3`, RDMA, and NCCL, while management traffic still selects the management plane.

## 5. Multi-Node Operations

Read-only fan-out may be parallel. Aggregate in code and fail if any host is missing or nonzero.

For state changes:

1. Quiesce/drain the distributed workload.
2. Change the worker/non-coordinator first.
3. Verify its boot, driver, memory, management, fabric, and workload readiness.
4. Change the coordinator/head.
5. Verify every rank and an end-to-end functional request.

Use the deployment's supported launcher/supervisor and rank order. Before relaunch, prove no stale ranks/containers/CUDA processes remain, ports are free, memory recovered, and both nodes match on image digest, artifact identity, launch parameters, driver, and fabric selection. Inspect every rank's logs; a ready API can conceal a failed worker or transport fallback.

## 6. Performance Monitoring

DGX Spark uses unified CPU/GPU DRAM; `nvidia-smi` may report framebuffer memory as unsupported. Corroborate rather than relying on one counter.

```bash
# Unified memory and pressure
free -h
grep -E 'MemAvailable|SwapFree|SwapTotal|HugePages' /proc/meminfo
cat /proc/pressure/memory
vmstat 1 5
swapon --show
nvidia-smi
nvidia-smi pmon -c 1 2>/dev/null || true

# CPU, storage, thermals, cgroups
uptime
mpstat 1 5 2>/dev/null || true
iostat -xz 1 5 2>/dev/null || true
pidstat 1 5 2>/dev/null || true
systemd-cgtop -b -n 1
sensors 2>/dev/null || true
nvidia-smi dmon -d 1 -c 5 2>/dev/null || true

# Fabric and services
ip -s link show dev <fabric-interface>
ethtool <fabric-interface>
ethtool -S <fabric-interface>
rdma link show; ss -s
docker stats --no-stream 2>/dev/null || true
systemctl --failed --no-pager
journalctl -p warning..alert --since '<approved-window>' --no-pager
```

Report the measurement window and workload state. Distinguish free memory, `MemAvailable`, page cache, swap allocated versus fresh `si`/`so`, reserved cache capacity versus active use, and process/cgroup memory versus CUDA/UVM allocation. Stable swap occupancy with no I/O is not active thrashing; PSI shows recent contention. Compare network counters at both ends before and after a bounded workload.

## 7. Reclaim Unified Memory and Run Headless

DGX Spark uses unified CPU/GPU memory. A stopped CUDA job can leave the host looking full for three different reasons that require different actions:

1. **Reclaimable file/page cache** — high `Cached`, but usually also high `MemAvailable`.
2. **A live owner** — process RSS/anonymous memory, a container, a stale distributed rank, or a CUDA/UVM user.
3. **Unavailable memory with no visible owner** — possible pinned/UVM/driver retention; cache dropping may not repair it.

Never use low `MemFree` alone as evidence of a leak. Prefer `MemAvailable`, then explain `Cached`, `SReclaimable`, swap activity, PSI, processes/cgroups, containers, and CUDA/UVM ownership.

### 7.1 Diagnose before reclaiming

Capture the same snapshot on every affected node before and after cleanup:

```bash
awk '/^(MemTotal|MemFree|MemAvailable|Cached|Buffers|SReclaimable|Shmem|AnonPages|SwapTotal|SwapFree):/ {print}' /proc/meminfo
cat /proc/pressure/memory
vmstat 1 5
swapon --show
ps -eo pid,ppid,user,rss,vsz,stat,comm,args --sort=-rss | head -40
systemd-cgtop -b -n 1
docker ps --no-trunc 2>/dev/null || true
nvidia-smi
nvidia-smi pmon -c 1 2>/dev/null || true
sudo fuser -v /dev/nvidia* 2>/dev/null || true
```

Before reclaiming anything:

1. Stop only the failed/finished workload through its real launcher or supervisor.
2. Prove every distributed rank and matching container is gone on every node.
3. Prove the intended ports are free and no process still owns `/dev/nvidia*`.
4. Record `MemAvailable`, `Cached`, `AnonPages`, swap `si`/`so`, and PSI.

Use this decision table:

| Evidence | Meaning | Action |
|---|---|---|
| `Cached` high and `MemAvailable` also high | Normal reclaimable Linux cache | Do nothing; capacity is available |
| `AnonPages`, process RSS, cgroup, container, or CUDA owner high | Live allocation | Stop/fix the exact owner; do not hide it with cache dropping |
| No owner, `Cached` high, `MemAvailable` unexpectedly low after a failed load | Page-cache pressure is plausible | Run one authorized one-shot cache drop, then measure the delta |
| No owner and one-shot cache drop does not restore expected availability | Not ordinary page cache | Preserve logs and perform a worker-first rolling reboot |
| Fresh swap-in/out or high memory PSI | Active pressure/thrashing | Quiesce workloads before another model load |

A cache drop is useful before retrying a **failed** large model load when page-cache pressure is proven. If a model loaded successfully, leave it alone; do not run reclaim tools underneath it merely because `MemFree` is low.

If the behavior began after a DGX OS or NVIDIA driver update, also record the exact DGX release, kernel, loaded driver, installed package versions, and bounded kernel logs:

```bash
cat /etc/dgx-release 2>/dev/null || true
uname -a
nvidia-smi --query-gpu=driver_version --format=csv,noheader
cat /proc/driver/nvidia/version 2>/dev/null || true
dpkg-query -W 'nvidia*' 'linux-*nvidia*' 2>/dev/null || true
journalctl -k --since '<workload-stop-time>' --no-pager
```

Call it a driver/UVM retention problem only when the issue is repeatable, owners are gone, ordinary cache reclaim does not explain or repair it, and logs/version comparison support that conclusion. The cache cleaner is a workaround for UMA page-cache pressure, not proof of a driver leak.

### 7.2 One-shot cache reclaim

NVIDIA's DGX Spark guidance documents manually flushing buffer/page cache for UMA memory pressure:

```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```

Run it only after all relevant ranks and containers are stopped. Then immediately repeat the memory snapshot and report the exact before/after `MemAvailable`, `Cached`, swap activity, and PSI. This command drops clean page cache, dentries, and inodes; it does not terminate owners or guarantee release of pinned/UVM memory.

### 7.3 Install and use NVIDIA's continuous cache cleaner

Current NVIDIA VSS prerequisites publish a DGX Spark cache-cleaner script. Install it from the current NVIDIA documentation rather than downloading an unreviewed copy. The documented implementation is equivalent to:

```bash
sudo tee /usr/local/bin/sys-cache-cleaner.sh >/dev/null <<'EOF'
#!/bin/bash
set -e
echo 0 > /proc/sys/vm/nr_hugepages
echo "Starting cache cleaner - Running"
echo "Press Ctrl+C or stop its service to exit"
while true; do
  sync
  echo 3 > /proc/sys/vm/drop_caches
  sleep 3
done
EOF
sudo chmod 0755 /usr/local/bin/sys-cache-cleaner.sh
sudo test -x /usr/local/bin/sys-cache-cleaner.sh
```

The script does two consequential things: it sets runtime huge pages to zero and drops page cache every three seconds. Record the original huge-page count first:

```bash
cat /proc/sys/vm/nr_hugepages
```

Prefer a transient systemd unit so the cleaner is observable, easy to stop, and does not silently persist across reboot:

```bash
sudo systemd-run --unit=sys-cache-cleaner \
  --property=Description='NVIDIA DGX Spark cache cleaner' \
  /usr/local/bin/sys-cache-cleaner.sh
systemctl is-active sys-cache-cleaner.service
systemctl status sys-cache-cleaner.service --no-pager
journalctl -u sys-cache-cleaner.service -n 20 --no-pager
```

Stop it when the memory-sensitive deployment or recovery window ends:

```bash
sudo systemctl stop sys-cache-cleaner.service
systemctl is-active sys-cache-cleaner.service
```

Restore a nonzero prior huge-page value only if one was recorded and the host's workload requires it:

```bash
sudo sysctl vm.nr_hugepages=<recorded-prior-value>
```

Do not make the cleaner persistent by default. Continuous cache eviction can reduce filesystem/model reload performance, and the huge-page change may affect other workloads. Use the one-shot command first for ordinary failed-load cleanup; use the continuous cleaner only when a repeatable workload needs it or current NVIDIA guidance for that deployment requires it. Verify that `pgrep -af sys-cache-cleaner.sh` shows no duplicate cleaners.

### 7.4 Headless-service audit

Inventory service enablement, activation, and memory before deciding what is nonessential:

```bash
systemctl get-default
loginctl list-sessions
systemctl status display-manager --no-pager
systemctl --type=service --state=running --no-pager
systemctl list-unit-files --type=service --no-pager
systemd-cgtop -b -n 1
ps -eo pid,ppid,%cpu,%mem,rss,comm,args --sort=-rss
```

Common **candidates** on an SSH-only Spark are:

- `display-manager.service`, GDM, and GNOME Remote Desktop;
- `bluetooth.service`;
- `cups.service` and `cups-browsed.service`;
- `avahi-daemon.service` when mDNS discovery is not used;
- `ModemManager.service` when no cellular modem is used;
- `switcheroo-control.service` on a fixed-GPU server;
- `upower.service` on a noninteractive headless appliance;
- `colord.service` when no display/color workflow exists;
- `udisks2.service` when removable-media automount/desktop storage management is not needed;
- `dgx-dashboard.service`, `dgx-dashboard-admin.service`, and `dgxstation-desktop.service` only when administration is exclusively through SSH and the DGX Dashboard is intentionally relinquished.

Do **not** disable SSH, NetworkManager/systemd-networkd, Tailscale used for recovery, time synchronization, Docker/containerd used by workloads, NVIDIA persistence/power/fabric services, `dgx-release.service`, storage needed by models, or an unfamiliar unit merely because its name looks desktop-related. Unit presence and names vary by DGX OS release; discover them live.

### 7.5 Convert one node to headless operation

Before any persistent change, require explicit authorization, two proven management paths, no active desktop user, current default target, display-manager identity, a memory/PSI baseline, and exact restore commands.

First test temporary GUI shutdown:

```bash
sudo systemctl stop display-manager.service
systemctl is-active display-manager.service
```

If remote access, CUDA, networking, and workloads remain healthy, configure headless boot:

```bash
sudo systemctl set-default multi-user.target
systemctl get-default
```

Disable only reviewed, present units. A typical candidate set is:

```bash
sudo systemctl disable --now \
  gnome-remote-desktop.service bluetooth.service \
  cups.service cups-browsed.service avahi-daemon.service \
  ModemManager.service switcheroo-control.service
```

D-Bus or socket activation can restart `upower`, `colord`, or `udisks2` even when a service is disabled. Mask these only after proving their functions are unnecessary:

```bash
sudo systemctl mask --now upower.service colord.service udisks2.service
```

If the DGX Dashboard is intentionally removed from the operating path:

```bash
sudo systemctl disable --now \
  dgx-dashboard.service dgx-dashboard-admin.service dgxstation-desktop.service
```

A unit missing from the host is not an error; a unit unexpectedly still active after disable/mask is. Record each actual state instead of hiding errors with a broad `|| true`.

Reboot only the worker/non-coordinator first. Verify changed boot ID, SSH/Tailscale recovery, `multi-user.target`, disabled/masked units, NVIDIA driver/CUDA, management and fabric networking, storage, containers, memory/PSI, and a real workload request. Only then apply and verify the peer.

### 7.6 Restore desktop operation

Use the recorded unit list rather than blindly enabling every candidate:

```bash
sudo systemctl unmask upower.service colord.service udisks2.service
sudo systemctl enable <previously-enabled-units>
sudo systemctl set-default graphical.target
sudo systemctl start display-manager.service
systemctl get-default
systemctl is-active display-manager.service
```

A maintenance reboot is the final proof that the desktop and chosen services return. Re-measure `MemAvailable`, PSI, service memory, and workload behavior before declaring the headless change beneficial.

## 8. Restart, Reboot, and Shutdown

Prefer the smallest scope: application reload → service/container → distributed workload → host.

For a service:

```bash
sudo systemctl restart <service>
systemctl is-active <service>
journalctl -u <service> --since '<restart-time>' --no-pager
```

Use the deployment's compose/launcher contract for containers; verify image digest, mounts, environment, devices, network, and restart policy before replacement.

### Rolling reboot

Preflight each node:

```bash
who; systemd-inhibit --list; systemctl --failed --no-pager
docker ps --no-trunc
cat /proc/sys/kernel/random/boot_id
sync
```

Drain workloads and reboot the worker/non-coordinator:

```bash
sudo systemctl reboot
```

Require the management endpoint to disappear and return with a changed boot ID. Then verify:

```bash
uptime; systemctl is-system-running; systemctl --failed --no-pager
nvidia-smi; free -h
ip -br address; ip route
ibdev2netdev; rdma link show
tailscale status 2>/dev/null || true
docker ps --no-trunc 2>/dev/null || true
```

Reprove fabric traffic in both directions. Only then reboot the coordinator/head. Restart the workload in required rank order and run a functional request. Ping alone is not readiness.

Before `sudo systemctl poweroff`, state and verify the recovery mechanism (physical access, tested wake-on-LAN, smart power, or out-of-band management).

## 9. DGX OS, Driver, CUDA, and Firmware Updates

Treat DGX Spark as a coordinated appliance stack. Read current release notes and the official update guide. Prefer DGX Dashboard. Require stable power, backup, recovery access, stopped workloads, a maintenance window, and before-manifests from every node.

Preflight:

```bash
dpkg --audit
apt-mark showhold
sudo apt-get -s dist-upgrade
fwupdmgr get-updates
```

Stop if simulation removes platform packages, changes driver/kernel families unexpectedly, or exposes broken package state. Never layer a generic NVIDIA `.run` driver over DGX OS packages or casually mix package sources/driver branches.

When manual updates are required, NVIDIA's documented sequence is:

```bash
sudo apt update
sudo apt dist-upgrade
sudo fwupdmgr refresh
sudo fwupdmgr upgrade
sudo reboot
```

Inspect each step; do not hide errors in a command chain. Update and fully validate one node before its peer.

Post-update, compare peer/before manifests and verify package health, DGX release, kernel, loaded driver, CUDA realpath/compiler, firmware, container runtime, SSH/Tailscale, ConnectX mapping and two-way traffic, RoCE/RDMA/NCCL, CUDA initialization in the intended runtime, and real application performance.

## Gotchas

| Symptom | Required distinction/action |
|---|---|
| GPU memory unsupported in `nvidia-smi` | Use OS unified-memory, PSI, swap, process/cgroup, and workload counters together |
| Memory remains used after CUDA exit | Separate page cache, live owners, and pinned/unowned UVM; use one-shot reclaim only for proven cache pressure; collect driver/kernel evidence; rolling reboot if unrecovered |
| Cache cleaner appears to fix an updated-driver issue | Record before/after memory and driver evidence; cache relief alone does not prove a driver leak |
| `nvidia-smi` works but build/runtime fails | Compare loaded driver, installed packages, kernel modules, CUDA toolkit, container runtime, and noninteractive PATH |
| Fabric works one way | Bind source/interface in both directions; inspect routes, neighbors, reverse-path filtering, firewall, and counters |
| Jumbo MTU configured | Prove do-not-fragment payloads both ways and benchmark before/after; preserve rollback |
| API/head appears healthy | Inspect every rank and run a functional workload |
| NCCL logs plugin probes | Distinguish informational probes from collective failure, fallback, timeout, or error |
| SSH disconnects during reboot | Require endpoint down/up plus changed boot ID |
| Host key changes | Verify fingerprint out of band; never delete blindly |
| Update succeeds on one node | Restore host-stack parity and functional fabric/application tests before closure |

## Verification and Report

Applicable gates:

- primary and alternate management paths;
- noninteractive SSH and intended sudo behavior;
- complete before/after manifests with no unexplained divergence;
- discovered ConnectX Ethernet/RoCE mapping;
- bound routes, ICMP, SSH, TCP payload, `iperf3`, RDMA, and NCCL in both directions and every intended rail;
- management/fabric route separation;
- unified memory, swap activity, PSI, storage, thermals/power when available, and fabric counters;
- measured headless/service effect plus restore commands;
- changed boot ID and post-boot platform checks;
- every distributed rank plus a functional application request;
- removal of temporary listeners, firewall rules, files, and test processes;
- usable rollback.

Return:

```text
Scope and authorization:
Systems reached:
Management paths:
Changes and rollback:
Per-node effective state/parity:
ConnectX forward/reverse route evidence:
Ethernet/RDMA/NCCL bidirectional results:
Unified-memory/PSI results:
Service state: started | ready | functionally healthy
Errors, warnings, and unverified gates:
Recommended next action:
```

Never report secrets or claim totals, speeds, capacity, or health without current real output.

## Authoritative References

Check at execution time:

- https://docs.nvidia.com/dgx/dgx-spark/
- https://docs.nvidia.com/dgx/dgx-spark/os-and-component-update.html
- https://docs.nvidia.com/dgx/dgx-spark/spark-clustering.html
- https://build.nvidia.com/spark/connect-two-sparks
- https://build.nvidia.com/spark/nccl
- https://docs.nvidia.com/vss/latest/prerequisites.html
- https://build.nvidia.com/spark/vss/troubleshooting
- https://tailscale.com/docs/install/linux
- https://tailscale.com/docs/features/magicdns
- https://tailscale.com/docs/features/tailscale-ssh
