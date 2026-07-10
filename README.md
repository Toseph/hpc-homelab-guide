# Senior Linux Admin & HPC Homelab Study Guide

A from-scratch, hands-on path for practicing the systems engineering skills that senior Linux/HPC roles actually test: Linux fundamentals, GPU scheduling, Kubernetes, RDMA/MPI concepts, identity and secure access, storage, and boot/firmware internals — all on two consumer boxes and a NAS you may already own.

This isn't a theory course. It's a build log, written as I actually ran it — including the mistakes, because the mistakes are often the most instructive part. If you're prepping for interviews in this space, treat each stage's closing questions as a self-test: can you explain this out loud, from memory, to someone who'll push back?

---

## 0. Why This Exists, and What It Assumes

Most "learn Kubernetes" or "learn HPC" tutorials run everything in one disposable VM or a cloud sandbox that costs money by the hour. That's fine for syntax, but it doesn't teach you the things that actually come up in senior interviews and senior jobs: what happens when two physical hosts disagree about the network, what a GPU passthrough failure actually looks like at 1am, why your bootloader matters, why identity has to come before scheduling.

This guide uses **two physical machines** (one dedicated lab host, one that stays your daily driver) plus a **NAS** for shared storage — enough hardware separation to make the networking, scheduling, and storage lessons real instead of simulated. A third spare box of any kind is a bonus, not a requirement; Stage 10 shows how to fold one in, using an Apple Silicon Mac mini as the worked example.

**What you'll need**, roughly:
- One "lab" box you're willing to wipe and dedicate — ideally with a GPU, 16–32 GB RAM, and any local SSD storage.
- Your normal daily-driver machine, used for administration and, later, as a second physical node for multi-host GPU work.
- Optional: a NAS or any second machine with spare storage, for backups and shared filesystem practice.
- Optional: any additional spare computer on the same network — an old laptop, a mini PC, a Raspberry Pi 5, or an Apple Silicon Mac mini used as a part-time auxiliary node. If it's an ARM box, it unlocks a whole extra lesson (Stage 10).
- Free software only: Proxmox VE, Ubuntu, SLURM, Kubernetes tooling, all open source.

**Example hardware used for this build**, so you can calibrate against yours:

| Hardware | Role |
|---|---|
| Ryzen 3800X / RTX 2080 Ti / 32 GB RAM / two SATA SSDs (480 GB + 1 TB) | Dedicated Proxmox VE 9.2 host — call it `lab1` in what follows. |
| Ryzen 7800X3D / RTX 4090 / 32 GB RAM workstation | Daily driver; later dual-boots Linux for multi-node GPU work. |
| 6-bay NAS, ~24 TB usable | NFS backup/ISO target and shared storage practice. |
| Mac mini M1 / 16 GB RAM | Optional auxiliary node — hosts a Linux VM that becomes the lab's **arm64** node in Stage 10. |

One correction I had to make out loud to myself partway through, because it's a good reminder to verify your own hardware rather than trust labels: the 1 TB drive is a **Samsung 860 EVO, which is SATA, not NVMe** — despite "EVO" branding overlapping with Samsung's NVMe line. Know your actual bus speeds before you plan around them; it changes what you should expect and what bottlenecks first.

---

## 1. Architecture at a Glance

### Roles

| Hardware | Role |
|---|---|
| Lab host (3800X/2080 Ti box) | Runs every VM. Whatever OS was on it before gets wiped. |
| Daily-driver workstation | `ssh`, web UIs, `tofu`, `ansible`, `kubectl` from here. Later: dual-boots Linux as a second physical GPU node. |
| NAS | NFS exports for backups/ISOs/templates, plus backing capacity for an object-storage stage later on. |
| Spare box (optional; example: M1 Mac mini with 16 GB RAM) | Runs one Linux VM that joins the lab as an arm64 SLURM node and/or Kubernetes worker (Stage 10). Stays on the main LAN and can remain a part-time node. |

### Networks

| Bridge | Purpose | Subnet |
|---|---|---|
| `vmbr0` | Your real LAN. Internet, NAS, admin access. **Your router owns DHCP here — never run a second DHCP server on this bridge.** | Whatever your router uses — verify it (`ipconfig`/`ip route`), don't assume. |
| `vmbr1` | Isolated cluster/provisioning network. VLAN-aware. The head node VM is its router/DHCP/DNS. | 10.10.0.0/24 |
| `vmbr2` (later stage) | Routing-lab segment. | as designed later |

### Cluster IP plan (10.10.0.0/24), domain `hpc.internal`

| Host | IP | Notes |
|---|---|---|
| head1 | 10.10.0.1 | dual-homed to vmbr0; NAT gateway, DHCP/DNS, slurmctld/slurmdbd |
| ipa1 | 10.10.0.5 | identity stage |
| node1–node4 | 10.10.0.11–.14 | compute nodes |
| gpu-node1 | 10.10.0.21 | GPU passthrough VM |
| k8s1–k8s3 | 10.10.0.31–.33 | Kubernetes stage |
| minio1 | 10.10.0.40 | object storage stage |
| DHCP/PXE pool | 10.10.0.100–.199 | on head1 |
| arm-node1 (Linux VM on the spare box) | a **LAN** address, static or DHCP-reserved | optional, Stage 10 — reaches 10.10.0.0/24 via a static route through head1 |

A note on naming: this guide uses `lab1.pve.hpc.internal` for the Proxmox host. `.internal` is an IANA-reserved TLD specifically meant for private networks like this (RFC 8375) — it won't resolve publicly and doesn't expose anything. Name yours whatever you like; the structure (`hostname.role.domain.internal`) is the part worth keeping.

### Storage plan

Local SSD/NVMe holds VM boot disks — VMs want fast local storage. The NAS gets NFS exports: one added to the hypervisor as a backup/ISO/template datastore, one reserved for later object-storage practice. Cluster `/home` and `/apps` are exported by the head node itself from a dedicated virtual disk — the classic HPC pattern of local scratch plus shared NFS for home/apps.

### RAM budget

Budget your build in "stages," not "everything running at once" — this is realistic anyway; production fleets manage capacity the same way. A rough split for a 32 GB host: hypervisor + capped ZFS cache ~5 GB, head node 4 GB, an identity VM 3 GB, three compute nodes at 2 GB each, one GPU-passthrough VM at 10 GB — leaving headroom. Swap compute nodes out for Kubernetes nodes when you move to that stage, rather than running both simultaneously.

---

# Part A — Hypervisor Deployment Runbook (Proxmox VE 9.2)

Budget one evening for A1–A7, one dedicated evening for A8 (GPU passthrough — the fiddliest single step), one more for A9–A10.

## A1. Prep, USB, BIOS

1. Download the **Proxmox VE** ISO from proxmox.com and verify its SHA256 checksum.
2. Write it to USB in **raw/DD mode** — Rufus ("DD Image mode" when prompted), balenaEtcher, or `dd`. Avoid Ventoy for this particular ISO; it has a history of quirks with Proxmox specifically.
3. **If your lab box has more than one drive and any of them has an old OS on it, decide your storage layout before you touch the installer.** I initially installed to the wrong drive by accident, with a forgotten old OS install sitting on a second drive I hadn't accounted for — which created boot ambiguity I had to clean up afterward (documented in A3, so you can learn from it without repeating it). The cleanest prevention, if you have multiple drives: **physically disconnect every drive except your install target** before booting the installer. It removes an entire category of mistake for the cost of one cable.
4. BIOS settings (naming varies by board; this is AMD-typical):
   - **SVM Mode: Enabled** (virtualization extensions)
   - **IOMMU: Enabled**, set explicitly rather than left on "Auto" — this matters later for GPU passthrough
   - **CSM: Disabled** — pure UEFI boot
   - **fTPM: Enabled** — gives you a TPM 2.0 for later boot-security experiments
   - Fast Boot off

## A2. Install

1. Boot the USB. Either the Graphical or Terminal UI installer produces an identical result — pick Graphical unless your hardware is console-only, since the console disappears anyway once you're done here (managed headlessly from here on).
2. Target disk: your chosen boot drive. Filesystem: **ZFS (RAID0)** if it's a single disk — despite the name, this just means "one vdev, no redundancy," which is the only valid choice with one drive. Durability for this lab lives in scheduled backups (A10), not disk mirroring.
3. If the installer's Advanced Options exposes a ZFS ARC max setting, set it modestly (2–3 GB is plenty for a lab-sized pool) — and double-check the units shown (MiB vs GiB) before accepting; it's an easy field to misread.
4. Country/timezone, a strong root password, a real email.
5. Management network: a static IP on your LAN. **Verify your actual gateway first** (`ipconfig` on Windows, `ip route | grep default` on Linux/Mac) rather than assuming — don't guess a common default and hope.
6. Reboot, remove the USB.

## A3. First boot — two fixes worth knowing before you hit them blind

**If your screen goes black right after kernel handoff:** this is a known rough edge on some NVIDIA cards (particularly Turing-generation, e.g. the 2080 Ti) — the card's firmware-level display output works fine through the boot menu, but the Linux kernel's own framebuffer driver can fail the handoff. The fix: at the boot menu, press `e` to edit, append `nomodeset` to the kernel line, boot once to confirm it resolves the issue, then make it permanent.

Find out which bootloader you're actually running first:
```bash
proxmox-boot-tool status
```
If it reports **systemd-boot** (typical for a ZFS-root install):
```bash
nano /etc/kernel/cmdline
# append nomodeset to the single existing line, e.g.:
# root=ZFS=rpool/ROOT/pve-1 boot=zfs nomodeset
proxmox-boot-tool refresh
```
If it reports **GRUB** instead, the same edit goes in `/etc/default/grub`'s `GRUB_CMDLINE_LINUX_DEFAULT` line, followed by `update-grub`.

Cold power-cycle (not just a warm reboot) to confirm it holds automatically.

**If you have leftover boot entries from a previous OS install** (a common byproduct of reusing hardware): your UEFI boot menu may show multiple entries — the new Linux one, a stale Windows Boot Manager pointing at nothing useful, and sometimes a generic "UEFI OS" fallback. Clean it up from inside the running hypervisor:
```bash
efibootmgr -v              # lists all entries by Boot number; identify which is which
efibootmgr -b XXXX -B      # deletes a specific stale entry
efibootmgr -o YYYY,ZZZZ    # sets your desired boot order, correct entry first
```
Do this only after any old OS on another drive has actually been wiped (next step) — no point deleting an entry for a still-bootable install twice.

## A4. Wiping and repurposing a second drive, safely

If you have a second local drive that previously held another OS (my case: an old Windows install on a second SSD I'd forgotten about), here's the safe way to reclaim it — whether you're doing this before or after the main install.

**If doing it before install, from the installer's Debug Mode shell:** most Proxmox installers have an "Advanced Options" submenu (easy to scroll past) offering a **Debug Mode** variant of the install. Booting it drops you into an early root shell — type `exit` once to reach a second, fuller shell with disk tools available — do your wipe there, `exit` again to continue into the normal installer.

**If doing it after install, from the running hypervisor shell:**
```bash
lsblk -o NAME,SIZE,MODEL,SERIAL     # positively identify the drive by its listed size and model — don't guess from letter alone
zpool import                          # if an old ZFS pool is importable, note it — do NOT import it
zpool labelclear -f /dev/sdX3        # ZFS keeps label copies at the end of the partition; a Proxmox-style install usually put its pool on partition 3 — confirm with lsblk first
sgdisk --zap-all /dev/sdX            # destroys the partition table on the whole disk
wipefs -a /dev/sdX                    # clears any residual filesystem/RAID/ZFS signatures
```
Substitute your actual device (`/dev/sdX`) — confirm it with `lsblk` every single time before running anything destructive; there's no undo here. Note this is a metadata wipe, not a full-disk erase — it runs in seconds because it only needs to remove the small structures (partition table, filesystem signatures, ZFS labels) that make the old layout discoverable, not overwrite the actual data. That's sufficient for reuse; if you need a forensic-grade wipe, reach for `blkdiscard` (fast on SSDs) instead.

**Why this order matters if you're doing the main install first:** installing to drive A while an old pool still exists, unwiped, on drive B creates a window where two ZFS pools can claim the same pool name during the *next* boot — which can produce ambiguous or failed imports. Wiping the second drive **before** first boot (via Debug Mode) sidesteps this category of problem entirely; physically disconnecting it during install is the even simpler version of the same idea.

## A5. Post-install configuration

1. **Repositories:** disable the enterprise repo, enable the no-subscription one (GUI: *Datacenter → your host → Updates → Repositories*), then `apt update && apt full-upgrade -y`.
2. **ZFS ARC cap**, if not already set in the installer:
```bash
echo "options zfs zfs_arc_max=3221225472" > /etc/modprobe.d/zfs.conf   # 3 GB, adjust to taste
update-initramfs -u -k all        # required since root lives on ZFS
```
3. **Sanity checks:**
```bash
dmesg | grep -i -e AMD-Vi -e IOMMU     # expect interrupt-remapping lines — confirms IOMMU is actually active
pvesm status
zfs list
```

## A6. Claiming a second drive as VM storage

If you have a second drive, dedicate it entirely to VM disks rather than sharing the boot drive — this gives your VMs faster, cleaner storage and keeps the hypervisor's own root filesystem in its own small, independent failure domain.
```bash
zpool create -o ashift=12 vmdata /dev/sdX
zfs set compression=lz4 vmdata
pvesm add zfspool vmdata --pool vmdata --content images,rootdir --sparse 1
```
From here on, when creating any VM, explicitly choose this pool as its storage target rather than the default pool living on your boot drive.

## A7. Bridges

Add the isolated cluster bridge (the main LAN bridge exists already from install):
```
auto vmbr1
iface vmbr1 inet manual
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 2-4094
```
Apply with `ifreload -a`. It deliberately has no host IP and no physical port — a pure virtual switch; a VM you build later will act as its router, which is also why any DHCP server you run on it can never leak onto your real LAN.

## A8. NAS and ISOs

1. On your NAS, create two NFS exports — one for the Proxmox datastore, one held in reserve for later object-storage practice.
2. `pvesm add nfs nas-pve --server <NAS-IP> --export /nfs/pve --content backup,iso,vztmpl`
3. Pull ISOs (GUI: *→ ISO Images → Download from URL*): a recent Ubuntu Server LTS release, and a recent Rocky/AlmaLinux minimal release if you'll want RPM-based identity tooling later.

## A9. GPU passthrough (its own evening)

If your lab host has no integrated graphics, once the GPU is claimed for passthrough its local console goes dark permanently — that's expected, and it's why this whole lab is designed to be managed over SSH and the web UI from here on.

1. **Map IOMMU groups:**
```bash
for g in /sys/kernel/iommu_groups/*; do
  echo "Group ${g##*/}:"
  for d in "$g"/devices/*; do echo -e "\t$(lspci -nns "${d##*/}")"; done
done
```
Modern NVIDIA cards are typically **multi-function devices** — VGA, HDMI audio, sometimes a USB-C controller — and ideally all functions sit in one clean IOMMU group. Consumer AMD boards are usually clean here; if yours aren't, check for an "ACS Enable" BIOS option before resorting to more invasive kernel-patch workarounds.

2. **Cmdline:** append `iommu=pt` to whatever kernel cmdline file you're already using (same file as the `nomodeset` fix above), then re-apply (`proxmox-boot-tool refresh` or `update-grub`, matching your bootloader).

3. **Bind the card to vfio:**
```bash
cat >> /etc/modules <<'EOF'
vfio
vfio_iommu_type1
vfio_pci
EOF

cat > /etc/modprobe.d/blacklist-gpu.conf <<'EOF'
blacklist nouveau
blacklist nvidia
blacklist nvidiafb
blacklist nvidia_drm
EOF

cat > /etc/modprobe.d/vfio.conf <<'EOF'
options vfio-pci ids=<your PCI IDs from lspci -nn, comma-separated> disable_vga=1
softdep nouveau pre: vfio-pci
softdep nvidia pre: vfio-pci
EOF

update-initramfs -u -k all && reboot
```

4. **Verify:** `lspci -nnk -s <bus:device>` should show `Kernel driver in use: vfio-pci` for every function of the card.

5. **Know your way back:** if you ever need the local console again, boot-menu edit → append `modprobe.blacklist=vfio_pci` for that one boot. vfio never binds, the normal framebuffer driver takes over, console returns.

## A10. Build the GPU VM and a reusable template

**GPU passthrough VM** (adjust IDs/paths to your environment):
```bash
qm create 201 --name gpu-node1 --machine q35 --bios ovmf --cpu host \
  --cores 6 --memory 10240 --balloon 0 \
  --scsihw virtio-scsi-single --scsi0 vmdata:64 \
  --net0 virtio,bridge=vmbr1 --ostype l26 --agent enabled=1
qm set 201 --efidisk0 vmdata:1,efitype=4m,pre-enrolled-keys=0
qm set 201 --tpmstate0 vmdata:1,version=v2.0
qm set 201 --ide2 nas-pve:iso/<your-ubuntu-iso>,media=cdrom
qm set 201 --hostpci0 <bus:device>,pcie=1
```
`--balloon 0` and fixed memory are mandatory — passthrough pins guest memory, so ballooning does nothing useful and just wastes host RAM accounting. Leave the default virtual display attached alongside the GPU so you keep a fallback console via noVNC; it doesn't interfere with CUDA. Inside the guest: install the vendor's recommended server-branch driver, confirm with `nvidia-smi` (or the equivalent for your GPU vendor).

**Cloud-init template**, so every future VM clones instead of hand-installing:
```bash
cd /var/lib/vz/template
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
qm create 9000 --name ubuntu-tmpl --machine q35 --bios ovmf --cpu host \
  --cores 2 --memory 2048 --scsihw virtio-scsi-single \
  --net0 virtio,bridge=vmbr1 --ostype l26 --agent enabled=1
qm set 9000 --efidisk0 vmdata:1,efitype=4m,pre-enrolled-keys=0
qm set 9000 --scsi0 vmdata:0,import-from=/var/lib/vz/template/noble-server-cloudimg-amd64.img
qm disk resize 9000 scsi0 +18G
qm set 9000 --ide2 vmdata:cloudinit
qm set 9000 --boot order=scsi0 --serial0 socket --vga serial0 --ipconfig0 ip=dhcp
qm set 9000 --ciuser <you> --sshkeys /root/<your-pubkey>.pub
qm template 9000
```
Test with `qm clone 9000 150 --name scratch1 --full` — it should boot in seconds and accept your SSH key immediately.

## A11. Backups

Configure scheduled backups to your NAS datastore (weekly, zstd compression, snapshot mode is usually sufficient for a lab). **Do one restore drill before moving on** — a backup you've never restored is a rumor, not a safety net.

**Part A is done when:** the host boots hands-free to a console with no manual intervention; any old boot-menu entries are cleaned up; a second drive (if present) is dedicated purely to VM storage; the isolated bridge is up; the NAS datastore is mounted; your GPU shows `vfio-pci` on every function; `nvidia-smi` (or equivalent) is clean inside the passthrough VM; a template clone boots and accepts your key; one backup has been successfully restored.

---

# Part B — Study Stages

Each stage below builds one coherent skill area and ends with a handful of **self-test questions** — the kind of thing a technical interviewer would actually ask, or that you should be able to answer cold if this is genuinely your area of expertise. Treat them as checkpoints, not homework.

## Stage 1 — Linux and Python fundamentals (start here, keep going throughout)

This is the foundation everything else sits on, and it needs no lab at all — you can start today.

Daily practice: pick a fundamentals topic, explain it out loud as if to an interviewer, then verify. Non-negotiable topics:
- Boot flow on a UEFI Linux box: firmware → bootloader → kernel → init, and who verifies what at each stage.
- `ssh host` end-to-end: DNS resolution, TCP handshake, key exchange, authentication, pty allocation, shell.
- `/proc` exploration, `strace` on a misbehaving command, signals, zombie vs. orphan processes, the difference between page cache and buffers in `free` output, the cgroup v2 hierarchy under `/sys/fs/cgroup`.
- Concurrency in Python for systems work — three facts worth knowing precisely rather than approximately: **CPython's `hashlib` releases the GIL for buffers above roughly 2 KB**, so threaded hashing of multiple files genuinely parallelizes across cores despite the GIL; **`ProcessPoolExecutor.map()` is lazy and returns results in input order, so one slow item blocks everything behind it, and an exception in any worker propagates at the point you consume that result** — `as_completed()` with per-future error handling is the more resilient pattern; **a second run over the same data being much faster is almost always the OS page cache at work**, and that speedup breaks down once your working set exceeds available RAM, when data lives behind a cold storage tier, or when the second run happens to land on a different machine with a cold cache.
- Practice katas: a streaming log parser that never loads the whole file into memory, a parallel file-checksum tool (try both a thread-pool and a process-pool version, and be able to argue for one depending on where the files actually live — local NVMe vs. a network share changes the right answer), a `subprocess` wrapper with proper timeout and retry handling.

**Self-test:** Walk through everything that happens between pressing the power button and seeing a login prompt on a UEFI Linux machine. Explain why threaded file-hashing can outperform sequential hashing despite the GIL. What's the difference between `executor.map()` and `as_completed()`, and when does it matter? Why might a second identical run of a data-processing job be dramatically faster than the first — and under what conditions would that stop being true?

## Stage 2 — Build a SLURM-scheduled compute cluster

Clone VMs from your template rather than hand-installing each one — this is both faster and closer to how real fleets actually get built.

- **Head node**: a VM with a second NIC on the isolated bridge, running as the cluster's router (NAT for outbound internet), DHCP/DNS server (`dnsmasq` is simple and effective for a lab), and NFS server exporting `/home` and `/apps` to the rest of the cluster.
- Clone 3–4 lightweight "compute node" VMs.
- Install **SLURM** (`slurm-wlm` from your distro's repos is fine for lab purposes): `slurmctld` + `slurmdbd` + a small database on the head node. Configure `SelectType=select/cons_tres` with `SelectTypeParameters=CR_Core_Memory` for combined core/memory-aware scheduling, `ProctrackType=proctrack/cgroup` so jobs are actually fenced by cgroups rather than trusted to behave, and a `gpu` partition once your GPU VM is available (a manual `gres.conf` entry is often necessary since distro SLURM builds don't always ship with GPU auto-detection compiled in).
- Add Lmod and a small module tree with Apptainer/Singularity for containerized workloads — this mirrors how most real HPC shops manage software across many users and dependency versions.

**Self-test:** What does cgroup v2 actually do when a job exceeds its memory limit — walk through the mechanism, not just the outcome. Trace a job from `sbatch` submission to a running process across `slurmctld`, `slurmd`, and `slurmstepd`. Why choose `CR_Core_Memory` over core-only scheduling? What goes wrong with a shared NFS home directory under `root_squash`, and why does that setting exist?

## Stage 3 — RDMA, MPI, and GPU-to-GPU communication concepts

This is the stage that converts "I've heard of RDMA/InfiniBand" into something you've actually measured — you won't have real RDMA hardware in a typical home lab, but you can build genuine intuition for the concepts and the software stack around them.

1. **OpenMPI under SLURM.** Run the OSU microbenchmarks (`osu_latency`, `osu_bw`, `osu_allreduce`) between your compute nodes over plain TCP first, and record the numbers as a baseline.
2. **Soft-RoCE** — the trick that makes real RDMA verbs programming possible without special hardware. Any Ethernet NIC can host a software RoCEv2 implementation:
```bash
sudo apt install -y rdma-core ibverbs-utils perftest
sudo modprobe rdma_rxe
sudo rdma link add rxe0 type rxe netdev <your-nic>
ibv_devinfo                     # confirm a real verbs device now exists
ib_send_bw / ib_write_bw / rping   # run between two nodes
```
Performance here will actually be worse than plain TCP — this is software emulation, not real hardware acceleration — but that's not the point. The point is direct, hands-on fluency with the verbs API itself: queue pairs, completion queues, memory regions, and the `rdma`/`ibv_*` tooling. Once you've done this, you can speak specifically to what real RDMA hardware changes: kernel bypass, NIC-offloaded transport, true zero-copy transfers.
3. **Multi-node GPU collective communication.** If you have two GPU-capable machines (a passthrough VM and your daily-driver workstation, for instance), boot both into Linux and try NCCL's `all_reduce_perf` benchmark between them over your regular network. Mixed GPU generations work fine here — NCCL doesn't require identical hardware, though a real training job would be paced by the slower card. Watch the reported bandwidth and learn to read it properly: **algorithm bandwidth (algbw) vs. bus bandwidth (busbw)**, where for an all-reduce operation `busbw = algbw × 2(N−1)/N`. On ordinary gigabit Ethernet you'll watch this pin at roughly 110 MB/s — a direct, felt demonstration of why real GPU training clusters need RDMA fabrics at all.
4. **Theory worth knowing cold, even without the hardware:** RoCEv2 encapsulates InfiniBand's transport semantics inside UDP; why that historically wants a "lossless" Ethernet fabric; Priority Flow Control and its failure modes (head-of-line blocking, pause storms); ECN and DCQCN as the modern congestion-control answer; why a single elephant flow can defeat ECMP load-hashing and how RDMA transports work around it; ring vs. tree topologies for all-reduce.
5. **One nuance worth knowing precisely:** GPUDirect RDMA — letting a NIC read/write GPU memory directly — requires data-center-class GPUs. Consumer GeForce cards route that traffic through host system memory instead. Knowing this distinction, and why it exists, is itself a mark of real depth rather than surface familiarity.
6. **Worthwhile optional spend (~$60–90):** a pair of used older-generation RDMA-capable NICs and a direct-attach cable, connected point-to-point between two machines, gets you genuine hardware verbs performance testing rather than software emulation.

**Self-test:** Walk through what happens between calling an RDMA "post send" operation and its completion being signaled. Explain InfiniBand's credit-based flow control versus RoCE's PFC/ECN approach — mechanisms and failure modes for each. Given a link speed and node count, compute the theoretical bus bandwidth of a ring all-reduce. Why can't a consumer GPU do GPUDirect RDMA, and what changes on a data-center GPU?

## Stage 4 — Kubernetes with real GPU scheduling

Kubernetes "familiarity" and Kubernetes "I run a GPU-enabled cluster and understand batch scheduling on it" are very different claims in an interview — this stage is how you earn the second one.

- Build with **kubeadm**, not a simplified distribution, at least once — you want to personally touch every component: the container runtime, CNI setup, control-plane initialization, node joining, and recovering from things you've deliberately broken. A CNI choice like **Cilium** (eBPF-based, can replace kube-proxy entirely) is worth using specifically because it generates good "how does this actually work" conversations.
- **Bring a real GPU into the cluster:** join your passthrough VM as a worker node and install the **NVIDIA GPU Operator** via Helm (with the in-guest driver already installed, disable the operator's own driver management). Schedule a pod requesting a GPU resource and confirm it lands correctly. Then configure **time-slicing** to oversubscribe a single GPU across multiple pods, and be ready to explain the isolation tradeoffs versus MIG (partition-level isolation, available on data-center-class GPUs) and MPS (a different sharing model entirely).
- **Batch and gang scheduling** — the conceptual bridge between traditional HPC schedulers like SLURM and Kubernetes: install **Kueue**, define a resource quota, and submit jobs that get suspended until admitted. Know **Volcano** by name as the alternative gang-scheduling approach and be able to contrast its model with Kueue's.
- **Observability:** a Prometheus/Grafana stack with a GPU metrics exporter (DCGM) gives you a real dashboard showing utilization, queue depth, and node health — genuinely useful, and a good centerpiece if you ever need to demonstrate this work.
- **GPU fleet operations, worth an evening on their own:** `nvidia-smi topo -m` to understand interconnect topology; running a GPU diagnostic and stress tool while watching thermals; learning to read GPU error codes (NVIDIA's XID error taxonomy, for instance) well enough to distinguish "the application misbehaved" from "the GPU needs a reset" from "the hardware needs replacing"; practicing node cordon/drain patterns for planned GPU maintenance.
- **Optional multi-architecture twist:** if you have any ARM box available, Stage 10 shows how to join it as an arm64 worker — which turns this cluster from single-arch into heterogeneous, and unlocks a class of scheduling and image-management lessons single-arch clusters can't teach.

**Self-test:** Trace a network packet from one pod to another pod on a different node under an eBPF-based CNI — what replaced the traditional kube-proxy path, and how? How does the device plugin API let kubelet discover that a node has GPUs at all? Why does a naive scheduler deadlock on a two-pod MPI job, and how does gang scheduling prevent it? Contrast bin-packing and preemption behavior between a traditional HPC scheduler and the default Kubernetes scheduler. A pod is stuck Pending and requesting a GPU — walk through your full debugging process out loud.

## Stage 5 — Identity, access control, and secure provisioning

Making compute "available securely and conveniently" is a real requirement at any shop running shared research or production infrastructure — this stage builds the identity layer that everything else should depend on.

- Stand up a directory service — **FreeIPA** is a strong choice for a Linux-centric estate: it bundles Kerberos, LDAP, DNS, host-based access control, and sudo rules with a scriptable API. (Samba as an Active Directory-compatible domain controller is the alternative if you specifically want Windows-style AD semantics, or want to practice building a trust relationship between the two — a genuinely impressive pattern if you have time for it.)
- Enroll every node into the directory; use **automount** maps for home directories rather than static NFS mounts, since that's closer to how real HPC shops manage this at scale.
- Build access-control rules deliberately, and **test the negative case** — confirm that a user who isn't in the right group is actually denied, not just that authorized users work. That habit alone separates people who've configured access control from people who've verified it.
- Tie identity to scheduling: a small script that syncs directory groups into SLURM accounting, so who someone is in the directory determines what they can run and how it gets billed/tracked.
- A useful capstone exercise: build a simple, deliberately boring **onboarding pipeline** — a small service that takes a structured "new user" request, maps a role to the right groups and scheduler account via a policy table, creates the user in your directory, and writes an audit record. It doesn't need to be clever; it needs to demonstrate that you understand identity as the actual control plane for everything else, not an afterthought bolted on last.
- Prove the whole chain end to end: authenticate with Kerberos, SSH in without a password using that ticket, submit a scheduler job, and confirm the accounting system correctly attributes it to the right account.

**Self-test:** Walk through a Kerberos authentication exchange from initial request to a usable service ticket — what's actually inside a ticket-granting ticket? Trace how a simple `id` command resolves a username, through name service switch and any caching layer, back to the directory. How does ticket-based SSH authentication differ operationally from key-based SSH? Contrast the underlying models of an LDAP/Kerberos-based directory versus Active Directory's SID-based model.

## Stage 6 — Provisioning at depth: network boot and infrastructure-as-code

- Build a real **PXE boot chain**, ideally one that respects UEFI Secure Boot rather than disabling it — this involves a signed shim bootloader, a signed network-bootable GRUB, and serving your kernel/initrd/install image over HTTP rather than TFTP (TFTP chokes badly on anything beyond a tiny file). One classic gotcha worth knowing in advance: cloud-init/autoinstall datasource URLs often contain a semicolon, which GRUB's config parser will interpret as a statement separator unless you quote the whole argument.
- Pair this with **Ansible** roles for post-boot configuration, so a freshly PXE-installed node ends up fully configured — networking, identity enrollment, scheduler agent, monitoring — with no manual steps.
- Add **OpenTofu** (or Terraform) to define your VMs declaratively against your hypervisor, so `tofu apply` can create or destroy lab nodes reproducibly. This is genuine infrastructure-as-code practice against real infrastructure, which reads very differently in an interview than "I followed a cloud provider's tutorial."
- If you dual-home any node onto both your isolated cluster network and your main LAN, watch for the classic multi-homing trap: two default routes fighting each other. Suppressing the second one is a small, specific networking lesson worth having internalized.

**Self-test:** Walk through the PXE boot process end to end — DHCP options, TFTP or HTTP handoff, the shim-to-GRUB-to-kernel chain, and how cloud-init picks up from there. Why must your lab's DHCP server never be reachable from your main home network? What changes about PXE booting when Secure Boot is enforced rather than disabled?

## Stage 7 — Self-hosted object storage

Most research and ML infrastructure eventually needs an S3-compatible object store alongside traditional file storage — this stage builds one from scratch rather than only consuming a cloud provider's.

- Deploy **MinIO** (or a lighter alternative like Garage) backed by real disk — ideally block storage rather than a network filesystem, since object stores generally want the stronger consistency and locking guarantees block storage provides. Know **Ceph's RGW** by name as the answer once you're talking about truly large scale.
- Practice the object-storage-specific concepts that don't map cleanly onto file storage: bucket versioning, lifecycle rules that expire old object versions automatically, presigned URLs, bucket policies.
- Point any data-transfer tooling you've built (checksum-verified copy scripts, for instance) at your new endpoint — running your own tools against infrastructure you personally operate is a different kind of understanding than running them against someone else's managed service.

**Self-test:** What is S3's actual consistency model today, as opposed to the "eventually consistent" reputation it still carries from years ago? Why does multipart upload exist, and what do per-part checksums protect against? Contrast erasure coding and simple replication in terms of storage overhead, rebuild cost, and failure-domain tolerance.

## Stage 8 — Boot and firmware security fundamentals

This stage has the best return on unusual effort: almost nobody builds hands-on experience here, and it converts vague resume language like "familiar with Secure Boot/TPM concepts" into something you've actually built and can defend under questioning.

- **Secure Boot and MOK (Machine Owner Key) enrollment.** In a VM with a virtual TPM and OVMF firmware, deliberately try to load an unsigned kernel module under Secure Boot and watch it fail; then generate your own signing key, enroll it via `mokutil`, sign the module yourself, and watch it load. This is exactly the workflow real fleets hit whenever a hardware vendor's driver isn't signed by a recognized authority.
- **Measured boot with a TPM.** Read PCR (Platform Configuration Register) values and the boot event log; then bind a LUKS-encrypted disk's unlock key to a specific PCR value with `systemd-cryptenroll`, watch it auto-unlock on a normal boot, then deliberately change something that affects that PCR (disabling Secure Boot, for instance) and watch it correctly fall back to a passphrase. Know precisely why you'd bind to a PCR that captures Secure Boot policy state rather than one that captures exact boot-manager binary hashes — one survives routine updates, the other doesn't, and picking the wrong one means bricking your own unlock on the next patch cycle.
- Be able to state the difference between verified boot and measured boot precisely: **verified boot (Secure Boot) blocks execution of anything unsigned**; **measured boot records hashes of everything executed into the TPM, blocking nothing but enabling later attestation and the sealed-secret pattern above.**
- **Firmware-level Linux, safely.** You almost certainly cannot and should not try flashing alternative firmware like coreboot onto a real consumer motherboard — real brick risk, no board-level support, not worth it. But you can build and run coreboot for QEMU's emulated hardware with zero risk, and pair it with a minimal Linux-based initramfs (the LinuxBoot approach) that `kexec`s into a full OS — which is a faithful hands-on demonstration of the actual idea: replacing most of a traditional firmware stack with ordinary, auditable Linux plus `kexec`.
- Know the standard boot-stage vocabulary cold (the stages a traditional UEFI/coreboot firmware stack moves through before handing off to an OS loader), and know where the hardware root of trust actually begins on your platform before any general-purpose CPU code executes at all.

**Self-test:** Narrate power-on to login prompt again, but this time with every verification step named explicitly. Why does a shim bootloader exist, and what specific problem does MOK enrollment solve that vendor-level signing doesn't? Why bind a LUKS unlock to a PCR that reflects Secure Boot policy state rather than exact boot-binary hashes? What does a LinuxBoot-style firmware stack actually replace, and what's the security argument for doing so?

## Stage 9 — Routing and network fabric simulation

- Build a router VM between two of your virtual network segments; carve VLANs and practice inter-VLAN routing; add a second router VM and bring up OSPF between them. If you want to go further, a tool like containerlab lets you spin up a disposable multi-node routing topology on your workstation for practicing things like ECMP load-hashing behavior, which connects directly back to the RDMA/congestion-control material in Stage 3.

**Self-test:** Why did BGP become the dominant protocol for large datacenter network fabrics rather than a purely IGP-based design? Where do the Layer 2/Layer 3 boundaries sit in a typical leaf-spine topology, and why there specifically?

## Stage 10 — Optional: A Heterogeneous ARM Node (Worked Example: an Apple Silicon Mac Mini)

Real fleets are increasingly multi-architecture — AWS Graviton, NVIDIA Grace, Ampere Altra — and "our software ran fine until the ARM nodes arrived" is a genuinely common operational story now. If you have any ARM box on your network, you can fold it into this lab and learn that whole lesson class first-hand. The worked example here is an M1 Mac mini with 16 GB RAM used as a part-time auxiliary node.

**Prepare the machine so you never touch it physically again.** On macOS: enable **Remote Login** (SSH) and **Screen Sharing** in System Settings → Sharing — after this, everything else can happen over SSH. Then stop it from sleeping out from under your cluster, which is the number-one way a part-time machine sabotages a lab: `sudo pmset -a sleep 0` (and `sudo pmset -a autorestart 1` so it survives power blips). Since it may remain a part-time machine, cap the lab VM at roughly half the machine's cores and RAM, and stop the VM whenever the hardware is needed elsewhere — a node that comes and goes is *itself* useful practice in drain/cordon discipline.

**The Linux VM does the real work.** macOS can't natively be a SLURM or Kubernetes node — no Linux kernel, no cgroups — so the pattern is one Ubuntu Server **arm64** VM. Two good free options: **UTM** (GUI, Apple Virtualization backend, and — critically — bridged networking is a simple dropdown) or **Lima** (CLI-first and scriptable, but note that bridged networking requires installing `socket_vmnet` separately; its default user-mode networking makes the VM unreachable from the rest of your LAN, which defeats the purpose here). Give the VM a bridged LAN address and reserve it in your router.

**Routing it to the cluster network.** The cluster lives on 10.10.0.0/24 behind head1, so two small pieces of plumbing:

```bash
# In the arm VM (persist via netplan):
sudo ip route add 10.10.0.0/24 via <head1-LAN-IP>
```

And on head1, make sure your NAT rule only masquerades traffic actually leaving for the internet — not east-west traffic between the cluster and your LAN:

```
ip saddr 10.10.0.0/24 ip daddr != <your-LAN-subnet> masquerade
```

Why this matters: if head1 source-NATs cluster-to-LAN traffic, every cluster node appears to the arm VM as head1's address, which breaks direct node-to-node communication — Kubernetes CNI tunnels especially. "Only NAT what's leaving the building" is a classic real-world router discipline, learned here in miniature.

**Fold it into SLURM.** Ubuntu packages SLURM for arm64, so: install `slurm-wlm` in the VM, copy over the munge key and `slurm.conf`, and define it as its own partition — `NodeName=arm-node1 ... Feature=aarch64` with `PartitionName=arm Nodes=arm-node1`. (Keep SLURM versions matched between controller and this node; cross-distro version skew is a real compatibility constraint.) Then run the single most instructive demo this stage offers: compile a hello-world on an x86 node, `srun -p arm ./hello`, and watch it die with **`Exec format error`**. That one error is the entire reason real heterogeneous shops maintain per-architecture software trees — and now you can make your Lmod `MODULEPATH` include `$(uname -m)` so each architecture resolves its own builds, exactly as production sites do.

**Fold it into Kubernetes.** kubelet/kubeadm ship for arm64, so a standard `kubeadm join` adds it as a worker. Confirm with `kubectl get nodes -L kubernetes.io/arch` — you now run a genuinely heterogeneous cluster. The lessons that only exist now: schedule onto it deliberately with a `nodeSelector` on `kubernetes.io/arch`; watch what happens when a pod lands there with an **amd64-only image** (either an immediate "no matching manifest for linux/arm64" pull failure, or — if a single-arch image sneaks through — a container that starts and instantly dies with exec format errors); then fix it properly by building a **multi-arch manifest** (`docker buildx build --platform linux/amd64,linux/arm64 ...`). Optionally taint the node so only workloads that explicitly tolerate ARM land on it — the polite pattern for a part-time node. One Apple-specific flourish if you used the Virtualization backend: Rosetta can be exposed *inside* the Linux VM, letting the arm node execute amd64 Linux binaries via translation — a fun demonstration, though the real lesson is building multi-arch images correctly, not translating around single-arch ones.

**Honest limits, stated up front:** the M1's GPU speaks Metal, not CUDA — this node adds *architecture* diversity, not GPU capacity, and it will never participate in the NCCL work from Stage 3. Its throughput is modest and its availability is part-time. None of that matters: what it teaches — image manifests, per-arch software management, constraint-based scheduling, a node that joins and leaves — is exactly the operational texture single-architecture labs can't produce. A spare display can also make a useful always-on status dashboard once the monitoring stage exists.

**Self-test:** At exactly what layer does an amd64-only workload fail on an arm64 node — and why are there two different failure modes depending on how the image was built and pulled? What is a multi-arch manifest list, and how does a container runtime resolve the right image from it? Compare SLURM's `Feature`/`--constraint` mechanism with Kubernetes' node labels and `nodeSelector` — same idea, different ecosystems; where do they differ? Why do heterogeneous HPC sites maintain per-architecture software trees rather than relying on emulation?

---

# Part C — Putting It All Together: A Signature End-to-End Demo

Once several stages are built, wire them into one coherent demonstration — this is worth far more than the sum of its parts, because it proves you understand how the pieces actually fit together, not just each piece in isolation.

A reasonable shape for this: an infrastructure-as-code command builds a new compute node from nothing → a simple onboarding request creates a real user in your directory service with the correct group memberships → that user authenticates with a Kerberos ticket and submits a real job to your scheduler → the job lands on your GPU node, properly fenced by cgroups → a dashboard shows the utilization and the accounting system shows correct attribution → the same infrastructure-as-code tooling tears the node back down.

Each stage above feeds one link in that chain. Rehearse narrating it end to end until you could do it live, under questions, without losing the thread.

---

# Appendices

## A. A Reasonable Pacing Guide

There's no single correct pace — go faster if a stage is already close to your existing experience, slower where it's genuinely new. As a rough shape for someone working evenings and weekends:

| Stage(s) | Time |
|---|---|
| Hypervisor build (Part A) | 1 week |
| Stage 1 (fundamentals) | ongoing throughout — never really "done" |
| Stage 2 (SLURM cluster) | 1–2 weeks |
| Stage 3 (RDMA/MPI/NCCL) | 2–3 weeks |
| Stage 4 (Kubernetes + GPU) | 2–3 weeks |
| Stage 5 (identity) | 1–2 weeks |
| Stage 6 (PXE + IaC) | 1–2 weeks |
| Stage 7 (object storage) | a few evenings |
| Stage 8 (boot/firmware security) | ongoing background, alongside everything else |
| Stage 9 (routing) | ongoing background |
| Stage 10 (ARM node, optional) | a weekend — best done alongside or after Stages 2 and 4 |

## B. Break-Glass Cheatsheet (Recovery Moves Worth Knowing Before You Need Them)

- Black screen right after kernel handoff → boot menu, `e`, append `nomodeset` → once confirmed, persist it properly in your bootloader's config.
- Console lost to GPU passthrough → boot menu edit, append `modprobe.blacklist=vfio_pci` for that one boot → normal display driver takes back over.
- A cmdline edit "didn't take" → you likely edited the wrong bootloader's config file, or forgot to run the apply step (`proxmox-boot-tool refresh` for systemd-boot, `update-grub` for GRUB) — check which one you're actually running first.
- Two ZFS pools with the same name both importable at boot → this is why a second drive with an old install gets wiped (label-cleared, not just partition-deleted) before it's ever powered on alongside a fresh install.
- Stale UEFI boot entries cluttering the boot menu → `efibootmgr -v` to list, `-b XXXX -B` to delete a specific one, `-o` to set your preferred order.
- Genuinely stuck at a GRUB rescue prompt → `set prefix=(hd0,gpt2)/boot/grub`, `insmod normal`, `normal` — gets you back to a real menu you can fix from.
- A VM locked after a failed backup operation → unlock it explicitly rather than fighting it manually.
- Anything truly unrecoverable → this is exactly why you rehearsed a restore in Part A. A backup that's never been tested is a hope, not a plan.

## C. Reading List

LLNL's MPI tutorials remain a genuinely excellent free resource. The OSU microbenchmark documentation and the NCCL/`nccl-tests` documentation are worth reading directly rather than only through summaries — the bus-bandwidth math in particular is explained clearly in the primary sources. NVIDIA's GPU Operator documentation, the Kueue project docs, and FreeIPA/RHEL Identity Management documentation are all worth working through directly. On the networking-theory side: the original DCQCN congestion-control paper (SIGCOMM 2015) and the "RDMA over Commodity Ethernet at Scale" paper (SIGCOMM 2016) are both approachable and foundational. The coreboot project's own documentation and the u-root/LinuxBoot project pages are the best primary sources for that stage.

## D. A Note on Cost

Everything in this guide is achievable with free and open-source software on hardware you likely already own. If you do want to spend anything, the single best-value purchase for the RDMA stage is a pair of used older-generation RDMA-capable network cards and a direct-attach cable — often findable secondhand for well under $100 — which upgrades that entire stage from software emulation to genuine hardware verbs testing.
