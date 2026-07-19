# Senior Linux Admin & HPC Homelab Study Guide

*Version 4.0.2*

A from-scratch, hands-on path for practicing the systems engineering skills that senior Linux, HPC, and SRE roles actually test: Linux fundamentals, scheduler operations, parallel and shared storage, GPU scheduling, Kubernetes, RDMA/MPI concepts, identity, and boot/firmware internals — on two consumer machines and a NAS you may already own.

This is a build log turned into a curriculum, including the mistakes, because the mistakes are usually the most instructive part. If you are preparing for interviews in this space, treat each stage's closing questions as a self-test: can you explain this out loud, from memory, to someone who will push back?

---

## 0. How to Use This Guide

### Status labels — read these first

Not every stage here is equally battle-tested, and pretending otherwise would waste your time. Every stage carries an honest status:

| Label | Meaning |
|---|---|
| **Validated** | Built and verified end-to-end on the reference hardware, including the failure paths. |
| **In progress** | Actively being built; commands are real but the full chain is not yet proven. |
| **Designed** | Architecture, command sequence, and validation steps worked out and individually sound, but not yet run start-to-finish as written. |
| **Planned** | Roadmap. Concepts and approach are outlined; treat the specifics as a starting point for your own research. |

If a section is **Designed** or **Planned**, expect to do some debugging the guide does not do for you. That is not a defect — it is the honest state of a lab that is still growing, and working through the gaps is where the learning actually happens.

### Three reading paths

**If you already administer production Linux or HPC:** skip Part A's installer walkthrough (skim it for the failure modes, which are the interesting part), and go to whichever of Stages 4–6 covers your weakest area. Most experienced administrators find Kubernetes (Stage 6) and verbs-level RDMA (Stage 5) are the genuine gaps; scheduler and storage will be familiar but Stage 2's second half and Stage 3 both go deeper than typical day-to-day operation.

**If you are a general Linux administrator moving toward HPC or SRE:** work Part A → Stage 1 → Stage 2 → Stage 3 in order. Those four build the foundation everything else assumes. Then pick between Stage 5 (if you are targeting HPC) and Stage 6 (if you are targeting cloud-native infrastructure).

**If you are newer to infrastructure:** Part A and Stage 1 alone are several months of genuinely valuable work. Do not rush to Kubernetes. The people who interview well are the ones who can explain what happens between power-on and a login prompt, not the ones who memorized the most tool names.

### Guide map

- **Part A** — Hypervisor deployment runbook *(Validated)*
- **Stage 1** — Linux and Python fundamentals *(In progress — ongoing)*
- **Stage 2** — SLURM cluster, then scheduler operations *(Designed)*
- **Stage 3** — HPC storage: metadata, failure modes, parallel filesystems *(Designed)*
- **Stage 4** — RDMA, MPI, and GPU collective communication *(Designed)*
- **Stage 5** — Kubernetes with real GPU scheduling *(Designed)*
- **Stage 6** — Identity, access control, secure provisioning *(Designed)*
- **Stage 7** — Network boot and infrastructure-as-code *(Designed)*
- **Stage 8** — Self-hosted object storage *(Planned)*
- **Stage 9** — Boot and firmware security *(Planned)*
- **Stage 10** — Routing and fabric simulation *(Planned)*
- **Stage 11** — Optional heterogeneous ARM node *(Planned)*
- **Part C** — The signature end-to-end demonstration
- **Appendices** — pacing, break-glass recovery, reading list, cost

---

## 1. Why This Exists, and What It Assumes

Most "learn Kubernetes" or "learn HPC" tutorials run everything in one disposable VM or a cloud sandbox billed by the hour. That is fine for syntax, but it does not teach the things that come up in senior interviews and senior jobs: what happens when two physical hosts disagree about the network, what a GPU passthrough failure looks like at 1am, why your bootloader matters, why identity has to come before scheduling, or what a shared filesystem does to your jobs when it goes away mid-write.

This guide uses **two physical machines** (one dedicated lab host, one that stays your daily driver) plus a **NAS** — enough hardware separation to make the networking, scheduling, and storage lessons real instead of simulated. A third spare box is a bonus, not a requirement; Stage 11 shows how to fold one in.

**What you will need:**
- One "lab" box you are willing to wipe and dedicate — ideally with a GPU, 16–32 GB RAM, and local SSD storage.
- Your normal daily-driver machine, for administration and later as a second physical node for multi-host GPU work.
- Optional: a NAS or any second machine with spare storage, for backups and shared-filesystem practice.
- Optional: any additional spare computer — an old laptop, a mini PC, a Raspberry Pi 5, or an Apple Silicon Mac mini. If it is an ARM box, it unlocks an extra lesson (Stage 11).
- Free and open-source software only.

**The reference hardware**, so you can calibrate against your own:

| Hardware | Role |
|---|---|
| Ryzen 3800X / RTX 2080 Ti / 32 GB RAM / two SATA SSDs (480 GB + 1 TB) | Dedicated Proxmox VE host. |
| Ryzen 7800X3D / RTX 4090 workstation | Daily driver; later dual-boots Linux for multi-node GPU work. |
| 6-bay NAS, ~24 TB usable | NFS backup/ISO target and shared-storage practice. |
| Mac mini M1 (doubles as a home-theater box) | Optional arm64 node (Stage 11). |

One correction worth making out loud, because it is a good reminder to verify hardware rather than trust branding: the 1 TB drive here is a **Samsung 860 EVO, which is SATA, not NVMe** — the "EVO" name overlaps with Samsung's NVMe line. Know your actual bus speeds before planning around them; it changes what bottlenecks first.

---

## 2. Architecture

### Roles

| Hardware | Role |
|---|---|
| Lab host | Runs every VM. Whatever OS was on it before gets wiped. |
| Daily-driver workstation | `ssh`, web UIs, `tofu`, `ansible`, `kubectl`. Later dual-boots Linux as a second physical GPU node. |
| NAS | NFS exports for backups/ISOs/templates, plus backing capacity for the object-storage stage. |
| Spare box (optional) | One Linux VM joins the lab as an arm64 node (Stage 11). |

### Networks

| Bridge | Purpose | Subnet |
|---|---|---|
| `vmbr0` | Your real LAN. Internet, NAS, admin access. **Your router owns DHCP here — never run a second DHCP server on this bridge.** | Whatever your router uses — verify it, do not assume. |
| `vmbr1` | Isolated cluster/provisioning network. VLAN-aware. The head node VM routes it. | 10.10.0.0/24 |
| `vmbr2` (Stage 10) | Routing-lab segment. | as designed later |

### The lab at a glance

```
        HOME LAN  (your router owns DHCP)
        |-- admin workstation   ssh / kubectl / tofu
        |-- NAS                 NFS: backups, ISOs, templates
        \-- arm-node1           optional; static route -> 10.10.0.0/24
                 |
   ==============|=========================================
                 |
   HYPERVISOR HOST  (Proxmox VE)
   |
   +- vmbr0 ---- bridged to LAN ---- head1:ens18
   |                                    |
   |                             [ NAT / ip_forward ]
   |                                    |
   \- vmbr1 ---- 10.10.0.0/24 ----- head1:ens19   10.10.0.1
        |                             - dnsmasq  (DHCP / DNS / TFTP)
        |                             - NFS  /home  /apps
        |                             - slurmctld + slurmdbd
        |
        |-- node1..3     10.10.0.11-13   compute
        |-- gpu-node1    10.10.0.21      GPU via vfio-pci  <-- host PCIe
        |-- ipa1         10.10.0.5       FreeIPA / Kerberos / DNS
        |-- k8s1..3      10.10.0.31-33   Kubernetes
        \-- minio1       10.10.0.40      S3 object store
```

Two things this diagram is trying to make obvious. First, `vmbr1` has **no physical port and no host IP** — it is a pure virtual switch, which is why a DHCP server on it cannot leak onto your home LAN. Second, everything on the cluster network reaches the outside world through **one hop** (head1's NAT), which means head1 is both your convenience and your single point of failure — a realistic tradeoff worth feeling.

### Cluster IP plan (10.10.0.0/24), domain `hpc.internal`

| Host | IP | Notes |
|---|---|---|
| head1 | 10.10.0.1 | dual-homed; NAT gateway, DHCP/DNS, slurmctld/slurmdbd, NFS |
| ipa1 | 10.10.0.5 | identity stage |
| node1–node4 | 10.10.0.11–.14 | compute nodes |
| gpu-node1 | 10.10.0.21 | GPU passthrough VM |
| k8s1–k8s3 | 10.10.0.31–.33 | Kubernetes stage |
| minio1 | 10.10.0.40 | object storage stage |
| DHCP/PXE pool | 10.10.0.100–.199 | on head1 |
| arm-node1 | a LAN address | optional (Stage 11); routes to 10.10.0.0/24 via head1 |

**A note on `.internal`:** the ICANN Board permanently reserved `.internal` from delegation in the public DNS root for private use, by resolution in July 2024 — so it will not collide with a real gTLD. (RFC 8375 is a different thing: it defines `home.arpa` for residential home networks. Either is a defensible choice; `.internal` is the one used throughout this guide.) Avoid `.local`, which collides with mDNS and will cause you real pain with SSSD and directory services later.

### Storage plan

Local SSD holds VM boot disks. The NAS provides NFS exports: one as the hypervisor's backup/ISO/template datastore, one reserved for the object-storage stage. Cluster `/home` and `/apps` are exported by the head node itself from a dedicated virtual disk — the classic HPC pattern of local scratch plus shared NFS, and the thing Stage 3 takes apart in detail.

### RAM budget

Budget in **stages, not simultaneity** — this is realistic anyway, since production fleets manage capacity the same way. A rough split for a 32 GB host: hypervisor plus capped ZFS cache ~5 GB, head node 4 GB, identity VM 3 GB, three compute nodes at 2 GB each, one GPU-passthrough VM at 10 GB. Swap the compute nodes out for Kubernetes nodes when you reach that stage rather than running both.

---

# Part A — Hypervisor Deployment Runbook  `[Validated]`

Proxmox VE. Budget one evening for A1–A9, one dedicated evening for A10 (GPU passthrough — the fiddliest single step), one more for A11–A12.

**A note on the command examples throughout Part A:** most steps show what correct output actually looks like, because "did that work?" is the question every runbook leaves unanswered and the one that costs the most time. Where a command succeeds silently, that is called out too — silence is a result, and knowing which commands are supposed to be quiet saves you from chasing a non-problem. Sample outputs use plausible values (IPs, UUIDs, serial numbers); yours will differ in the specifics and should match in the shape.

## A1. Prep, USB, BIOS

1. Download the **Proxmox VE** ISO and verify its SHA256 checksum.
2. Write it to USB in **raw/DD mode** — Rufus ("DD Image mode" when prompted), balenaEtcher, or `dd`. Avoid Ventoy for this ISO specifically; it has a history of quirks with it.
3. **If your lab box has more than one drive and any of them holds an old OS, decide your storage layout before touching the installer.** On the reference build, an old forgotten Windows install on a second drive created boot ambiguity that took a session to clean up (documented in A3 and A5 so you can skip the experience). The cleanest prevention with multiple drives: **physically disconnect every drive except your install target** before booting the installer. One cable removes an entire category of mistake.
4. BIOS settings (names vary by vendor; these are AMD-typical):
   - **SVM Mode: Enabled** — virtualization extensions.
   - **IOMMU: Enabled**, set explicitly rather than left on "Auto" — required for GPU passthrough in A10.
   - **CSM: Disabled** — pure UEFI boot.
   - **fTPM: Enabled** — *optional*. This gives the **physical host** a TPM 2.0. Note that the boot-security work in Stage 9 happens inside VMs using a **virtual** TPM (`tpmstate0`), which does not depend on host fTPM at all. Enable host fTPM only if you intend to do measured-boot experiments on the bare-metal host itself.
   - Fast Boot: off.

## A2. Install

1. Boot the USB. Graphical and Terminal UI installers produce identical results — pick Graphical unless your hardware is console-only. Either way this is the last time you will use a local console; everything after is web UI and SSH.
2. Target disk: your chosen boot drive. Filesystem: **ZFS (RAID0)** if it is a single disk — despite the name, that means "one vdev, no redundancy," which is the only valid single-drive choice. Durability for this lab comes from scheduled backups (A12), not disk mirroring.
3. If the installer's Advanced Options exposes a ZFS ARC maximum, set it modestly (2–3 GB is plenty for a lab-sized pool) — and check the units shown (MiB vs GiB) before accepting; that field is easy to misread by a factor of 1024 in either direction.
4. Country/timezone, a strong root password, a real email address.
5. Management network: a static IP on your LAN. **Verify your actual gateway first** (`ipconfig` on Windows, `ip route | grep default` on Linux or macOS) rather than assuming a common default.
6. Reboot, remove the USB.

## A3. First boot — two fixes worth knowing before hitting them blind

**If the screen goes black right after kernel handoff:** a known rough edge on some NVIDIA cards (Turing generation, e.g. the 2080 Ti, is a common case). The card's firmware-level output drives the boot menu fine, but the kernel's own framebuffer driver can fail the handoff. At the boot menu press `e`, append `nomodeset` to the kernel line, and boot once to confirm that fixes it.

To make it permanent, first find out which bootloader is actually in use:

```bash
proxmox-boot-tool status
```

Expected output:

```
Re-executing '/usr/sbin/proxmox-boot-tool' in new private mount namespace..
System currently booted with uefi
E4E4-E4E4 is configured with: uefi (versions: 6.14.8-2-pve, 6.14.11-1-pve)
```

Read it this way: **`booted with uefi` is Proxmox's label for a systemd-boot-managed ESP**, which is typical for a ZFS-root UEFI install — it does not print the string "systemd-boot," so do not go looking for it. The hex string is the ESP partition's UUID, and the versions listed are the kernels currently installed onto that ESP. If instead you see `booted with grub`, use the GRUB path below.

For the `uefi` (systemd-boot) case:
```bash
vim /etc/kernel/cmdline
# append to the single existing line, e.g.:
# root=ZFS=rpool/ROOT/pve-1 boot=zfs nomodeset
proxmox-boot-tool refresh
```
For the `grub` case, the same edit goes in `/etc/default/grub` under `GRUB_CMDLINE_LINUX_DEFAULT`, followed by `update-grub`.

Cold power-cycle (not a warm reboot) to confirm it holds by itself.

**If you have leftover boot entries from a previous OS install** — a common byproduct of reusing hardware — your UEFI boot menu may show several: the new Linux entry, a stale Windows Boot Manager pointing at nothing useful, and sometimes a generic "UEFI OS" fallback. That ambiguity is what causes a machine to boot the wrong thing intermittently. Clean it from the running hypervisor:

```bash
efibootmgr -v              # list all entries with Boot numbers; identify each
```

A messy machine (what you are fixing) looks like:

```
BootCurrent: 0000
BootOrder: 0002,0000,0001
Boot0000* Linux Boot Manager	HD(2,GPT,1a2b...,0x800,0x100000)/File(\EFI\systemd\systemd-bootx64.efi)
Boot0001* Windows Boot Manager	HD(1,GPT,9f8e...,0x800,0x32000)/File(\EFI\Microsoft\Boot\bootmgfw.efi)
Boot0002* UEFI OS	HD(2,GPT,1a2b...,0x800,0x100000)/File(\EFI\BOOT\BOOTX64.EFI)
```

Three entries, and `BootOrder` starting with the generic `UEFI OS` fallback rather than your actual bootloader — that ambiguity is what makes a machine boot the wrong thing intermittently.

```bash
efibootmgr -b 0001 -B      # delete a specific stale entry (repeat per entry)
efibootmgr -o 0000         # set boot order, correct entry first
```

Clean afterward:

```
BootCurrent: 0000
BootOrder: 0000
Boot0000* Linux Boot Manager	HD(2,GPT,1a2b...,0x800,0x100000)/File(\EFI\systemd\systemd-bootx64.efi)
```

Do this only after any old OS on another drive has actually been wiped (A5).

## A4. First contact: verify the NIC, find the web UI, get a real editor

You have a console. Now you need to reach the machine from your workstation — and to know whether the network is actually working rather than guessing. Do this before anything else, because the rest of Part A is much easier from an SSH session than from a physical keyboard.

### A4a. Is the link physically up?

```bash
ip -br link
```

Expected on a healthy wired host:

```
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
enp6s0           UP             a8:a1:59:xx:xx:xx <BROADCAST,MULTICAST,UP,LOWER_UP>
wlp5s0           DOWN           b4:6b:fc:xx:xx:xx <NO-CARRIER,BROADCAST,MULTICAST,UP>
vmbr0            UP             a8:a1:59:xx:xx:xx <BROADCAST,MULTICAST,UP,LOWER_UP>
```

Read it this way: **`LOWER_UP` means carrier detected** — a cable is plugged into a live switch port. **`NO-CARRIER` means nothing on the other end** (unplugged, dead port, dead cable). The WiFi card showing `DOWN`/`NO-CARRIER` is expected and fine; see A4e.

Then check the physical layer directly:

```bash
ethtool enp6s0
```

Expected (trimmed to what matters):

```
Settings for enp6s0:
        Supported ports: [ TP ]
        Speed: 1000Mb/s
        Duplex: Full
        Auto-negotiation: on
        Link detected: yes
```

**`Link detected: yes` is the line that matters.** If it says `no`, stop and fix cabling before touching software. If `Speed:` reads 100Mb/s on gigabit hardware, you have a bad cable, a bad port, or a negotiation problem — that will quietly halve your NAS throughput later and be maddening to diagnose then.

If the driver never loaded at all, the interface will not appear in `ip -br link`. Check with:

```bash
lspci -nnk | grep -A3 -i ethernet
```

Expected: the controller listed with a `Kernel driver in use:` line beneath it. If that line is missing, the NIC has no driver — note the PCI ID and search for it; a very new consumer NIC occasionally needs a backported driver.

### A4b. Where is my IP address?

```bash
ip -br a
```

Expected:

```
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp6s0           UP
vmbr0            UP             192.168.1.10/24 fe80::aaa1:59ff:fexx:xxxx/64
```

**This is the detail that confuses nearly everyone:** the physical NIC has *no IP address*, and that is correct. Proxmox enslaves the physical interface to the bridge `vmbr0`, and **the bridge holds the address**. Do not diagnose a perfectly healthy machine as broken because `enp6s0` looks empty.

Confirm the enslavement is real:

```bash
ip link show master vmbr0
```

Expected: your physical NIC listed as a member. If this returns nothing, the bridge has no port — the machine will have an IP configured but no path to the network, which looks bewildering until you check this one thing.

### A4c. The web UI address

The address on `vmbr0` is your host address. From the example above, the web interface is:

```
https://192.168.1.10:8006
```

Port 8006, HTTPS (plain HTTP will not answer), and a self-signed certificate warning you are expected to click through.

Confirm the host agrees with itself about its identity:

```bash
hostname -f
cat /etc/hosts
```

Expected `/etc/hosts` line:

```
192.168.1.10 lab1.hpc.internal lab1
```

Proxmox reads its own address from `/etc/hosts` rather than from the interface. If you ever change the host's IP, change it here too, or cluster services and the web UI will disagree about where the machine lives.

### A4d. If there is no IP at all

Read the config that produced (or failed to produce) it:

```bash
cat /etc/network/interfaces
```

Expected for a static setup:

```
auto lo
iface lo inet loopback

iface enp6s0 inet manual

auto vmbr0
iface vmbr0 inet static
        address 192.168.1.10/24
        gateway 192.168.1.1
        bridge-ports enp6s0
        bridge-stp off
        bridge-fd 0
```

**The single most common failure here:** `bridge-ports` names an interface that does not exist — because the NIC is named something else than you assumed at install time, or a typo crept in. Compare that name character-by-character against `ip -br link`. Fix it, then:

```bash
ifreload -a
```

Then test outward in three deliberate steps, because each one isolates a different layer:

```bash
ping -c3 192.168.1.1        # your router: local L2/L3 works
ping -c3 1.1.1.1            # the internet by IP: routing works
ping -c3 deb.debian.org     # by name: DNS works
```

If the first two succeed and the third fails, it is DNS — check `/etc/resolv.conf` for a sane `nameserver` line. If the first succeeds and the second does not, check the `gateway` line. This three-ping ladder is worth internalizing; it turns "the network is broken" into a specific answer in about ten seconds.

### A4e. About the WiFi card

Leave it unconfigured, and use the wired connection for the lab. Two reasons, one practical and one fundamental:

- Proxmox ships no WiFi management stack by default — no NetworkManager, no configured `wpa_supplicant` — so the card simply sits idle.
- More fundamentally, **you cannot bridge a WiFi client interface the way you bridge ethernet.** The 802.11 frame format used in client mode does not carry the additional address field needed to forward traffic on behalf of other devices, so `bridge-ports wlp5s0` will not give your VMs network access. This is a protocol limitation, not a Proxmox one, and no amount of configuration fixes it.

If you want the card to stop appearing in `ip` output, blacklist its driver — but ignoring it is perfectly fine.

### A4f. The remote access model

The chain for this lab is:

```
your phone/laptop  ──Parsec──▶  workstation (4090)  ──SSH / HTTPS──▶  Proxmox host
                                        │
                                        └──▶ web UI ──▶ VM consoles (noVNC)
```

From the workstation:

```bash
ssh root@192.168.1.10                  # shell
```

and `https://192.168.1.10:8006` in a browser for the UI.

Set up key authentication so you are not typing the root password dozens of times per evening:

```bash
# run this ON the workstation, not the Proxmox host
ssh-copy-id root@192.168.1.10
```

Two things worth being deliberate about:

**Do not expose port 8006 or SSH to the internet.** The Parsec hop into your workstation *is* your remote access path, and it keeps the lab entirely LAN-only. If you later want direct access without Parsec in the middle, add a VPN (WireGuard or Tailscale) rather than forwarding ports on your router. A Proxmox web UI reachable from the open internet is a genuinely bad idea.

**The web UI's noVNC console is your crash cart.** When a VM loses its network — which will happen, repeatedly, as you build the cluster — you do not need SSH into that VM to fix it. Open its console from the UI and you have a keyboard on the virtual machine's screen. This is why losing the *host's* physical console to GPU passthrough (A10) is survivable: every VM still has a console, delivered over the network.

### A4g. Get vim

Proxmox ships `vim.tiny`, which provides `vi` but not full vim.

```bash
apt update
apt install -y vim
```

If `apt update` throws a `401 Unauthorized` on `enterprise.proxmox.com`, that is expected on a fresh install without a subscription — it is fixed in A6. The Debian repositories still work, so vim installs regardless.

Survival set, enough to edit every config file in this guide:

| Key | Action |
|---|---|
| `i` | insert before cursor |
| `A` | append at **end of line** — the one you want for `/etc/kernel/cmdline` |
| `Esc` | return to normal mode |
| `:w` | write |
| `:q` / `:q!` | quit / quit discarding changes |
| `:wq` or `ZZ` | write and quit |
| `dd` | delete current line |
| `/text` then `n` | search, jump to next match |
| `gg` / `G` | jump to start / end of file |
| `u` | undo |

One gotcha specific to `vim.tiny`: arrow keys in insert mode may insert stray `A`/`B`/`C`/`D` characters instead of moving the cursor. Installing full vim fixes it, as does `:set nocompatible`.

## A5. Wiping and repurposing a second drive, safely

If a second local drive previously held another OS, here is the safe reclaim — and the order matters more than the commands do.

**Before install, from the installer's Debug Mode shell:** most Proxmox installers have an **Advanced Options** submenu (easy to scroll past) with a **Debug Mode** install variant. Booting it drops you into an early root shell; type `exit` once to reach a second, fuller shell with disk tools available; do the wipe there; `exit` again to continue into the normal installer.

**After install, from the running hypervisor shell:**
```bash
lsblk -o NAME,SIZE,MODEL,SERIAL   # identify the drive by size and model — never by letter alone
```

A drive still carrying an old Proxmox install looks like this — note the three partition children:

```
NAME     SIZE MODEL                SERIAL
sda    447.1G Samsung SSD 860 EVO  S3Zxxxxxxxxxxx
├─sda1  1007K
├─sda2     1G
└─sda3 446.1G
sdb    931.5G Samsung SSD 860 EVO  S5Gxxxxxxxxxxx
├─sdb1  1007K
├─sdb2     1G
└─sdb3 930.5G
```

That 1007K/1G/rest pattern is the Proxmox signature: BIOS boot partition, EFI system partition, then the ZFS pool filling the remainder — which is why the pool lives on **partition 3**.

```bash
zpool import                       # if an old pool shows as importable, note it — do NOT import
```

If a foreign pool is present:

```
   pool: rpool
     id: 1234567890123456789
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

        rpool       ONLINE
          sdb3      ONLINE
```

**This is the dangerous state** — two pools named `rpool`, one of which is your running system. If instead you see:

```
no pools available to import
```

that is success: either the disk is already clean, or you have not attached it yet. Either way there is nothing to collide with.

```bash
zpool labelclear -f /dev/sdX3      # ZFS label copies live at the END of the partition too;
                                   # a Proxmox-style install put its pool on partition 3
sgdisk --zap-all /dev/sdX          # destroy the partition table (whole disk, no partition number)
wipefs -a /dev/sdX                 # clear residual filesystem/RAID/ZFS signatures
```

`sgdisk --zap-all` prints:

```
Creating new GPT entries in memory.
GPT data structures destroyed! You may now partition the disk using fdisk or
other utilities.
```

`wipefs -a` prints one line per signature it erased (and nothing at all if there were none):

```
/dev/sdb: 8 bytes were erased at offset 0x00000200 (gpt): 45 46 49 20 50 41 52 54
/dev/sdb: 8 bytes were erased at offset 0xe8e0db5e00 (gpt): 45 46 49 20 50 41 52 54
/dev/sdb: 2 bytes were erased at offset 0x000001fe (PMBR): 55 aa
```

**Verify, and know what "clean" looks like:**

```bash
lsblk /dev/sdb
```

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sdb      8:16   0 931.5G  0 disk
```

A single line with **no indented partition children beneath it** — that is a bare disk with no partition table and no filesystem metadata. Confirm nothing lingers:

```bash
wipefs /dev/sdb        # no -a: report only
zpool import
```

Empty output from the first and `no pools available to import` from the second means the disk is genuinely clean and ready for A7.

Substitute your real device and confirm with `lsblk` every time; there is no undo. This is a **metadata wipe, not a full erase** — it completes in seconds because it only removes the structures that make the old layout discoverable (partition table, filesystem signatures, ZFS labels), not the underlying data. That is sufficient for reuse. For a forensic-grade wipe, use `blkdiscard` (fast on SSDs) instead.

**Why the order matters:** installing to drive A while an old pool still exists unwiped on drive B creates a window where two ZFS pools can claim the same pool name (`rpool`) at the *next* boot. Best case you get ambiguity errors; worst case an import fails and you land in an initramfs shell. Wiping drive B *before* first boot avoids the entire class. Physically disconnecting it during install is the simpler version of the same idea.

If you do end up at that initramfs prompt: `zpool import` with no arguments lists both pools with numeric IDs, then `zpool import -N <numeric-id-of-the-correct-pool> rpool` and `exit` continues the boot.

## A6. Post-install configuration

1. **Repositories.** The enterprise repository requires a paid subscription and will return `401 Unauthorized` without one, which aborts every `apt update`. Swap it for the no-subscription repository.

   The GUI path is *Datacenter → your host → Updates → Repositories* — but that assumes you can already reach the web UI, so here is the file-level method that always works. **Start by looking at what is actually there** rather than assuming:

   ```bash
   ls -la /etc/apt/sources.list.d/
   ```

   Proxmox VE 9 (Debian 13 "trixie") uses the deb822 `.sources` format, so expect something like:

   ```
   ceph.sources
   debian.sources
   pve-enterprise.sources
   ```

   A system upgraded from PVE 8 may *also* carry legacy `.list` files — check for those and for `/etc/apt/sources.list` itself, since a leftover enterprise line in either will keep throwing 401s no matter what you do to the `.sources` files.

   Look at the enterprise definition:

   ```bash
   cat /etc/apt/sources.list.d/pve-enterprise.sources
   ```

   Expected:

   ```
   Types: deb
   URIs: https://enterprise.proxmox.com/debian/pve
   Suites: trixie
   Components: pve-enterprise
   Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
   ```

   Disable it by adding a single line — deb822 supports an explicit toggle, which is cleaner than commenting or deleting because the file stays readable and reversible:

   ```bash
   vim /etc/apt/sources.list.d/pve-enterprise.sources
   # add:  Enabled: false
   ```

   Do the same for `ceph.sources` if its `URIs:` also points at `enterprise.proxmox.com`.

   Now add the no-subscription repository:

   ```bash
   vim /etc/apt/sources.list.d/pve-no-subscription.sources
   ```

   ```
   Types: deb
   URIs: http://download.proxmox.com/debian/pve
   Suites: trixie
   Components: pve-no-subscription
   Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
   ```

   **`Suites:` must match your Debian base**, or apt will fetch the wrong release's packages. Confirm it rather than trusting this document:

   ```bash
   cat /etc/os-release | grep VERSION_CODENAME
   ```

   Then update and verify:

   ```bash
   apt update
   ```

   Success looks like a clean run ending in `Reading package lists... Done` with **no** `401` and no `E:` lines. Before the fix you would have seen:

   ```
   E: Failed to fetch https://enterprise.proxmox.com/debian/pve/dists/trixie/InRelease
      401  Unauthorized [IP: ...]
   E: The repository '...' is not signed.
   ```

   Confirm which repositories are actually live:

   ```bash
   apt policy | grep -E "proxmox|debian"
   ```

   Expected: `download.proxmox.com` present, `enterprise.proxmox.com` absent. Then:

   ```bash
   apt full-upgrade -y
   ```

   The "no valid subscription" popup in the web UI is unrelated to this and harmless; it is a licensing notice, not an error.
2. **ZFS ARC cap**, if not set during install:
```bash
echo "options zfs zfs_arc_max=3221225472" > /etc/modprobe.d/zfs.conf   # 3 GB; adjust to taste
update-initramfs -u -k all        # required — root is on ZFS, so this must ride in the initramfs
```

After the next reboot, confirm the cap actually took:

```bash
arc_summary | grep -A4 "ARC size"
```

Expected — the `Max size` figure should equal what you set, not your total RAM:

```
ARC size (current):                                     4.1 %  126.2 MiB
        Target size (adaptive):                       100.0 %    3.0 GiB
        Min size (hard limit):                          6.2 %  192.0 MiB
        Max size (high water):                           16:1    3.0 GiB
```

If `Max size` still shows roughly half your RAM, the setting did not reach the initramfs — re-run `update-initramfs -u -k all` and reboot.
3. **Sanity checks — and what each should actually print.**

```bash
dmesg | grep -i -e AMD-Vi -e DMAR -e IOMMU
```

Expected on AMD with IOMMU enabled in BIOS:

```
[    0.512345] AMD-Vi: Using global IVHD EFR:0x246577efa2254afa, EFR2:0x0
[    0.534567] pci 0000:00:00.2: AMD-Vi: Found IOMMU cap 0x40
[    0.556789] AMD-Vi: Interrupt remapping enabled
[    0.578901] AMD-Vi: Virtual APIC enabled
```

**`Interrupt remapping enabled` is the line that matters.** *Empty output means IOMMU is off in firmware* — go back to BIOS before attempting A10, because passthrough cannot work without it. (On Intel the equivalent lines say `DMAR:` instead of `AMD-Vi:`.)

```bash
pvesm status
```

Expected on a fresh single-disk install:

```
Name             Type     Status           Total            Used       Available        %
local             dir     active        98497780         2841524        90621136    2.88%
local-zfs     zfspool     active       450000000            2400       449997600    0.00%
```

`Status: active` on every row is what you want. Once you add the NAS (A9) and second pool (A7), `nas-pve` and `vmdata` join this list — and an NFS storage showing `inactive` means the mount failed, not that the config is wrong.

```bash
zfs list
```

Expected on a fresh install:

```
NAME               USED  AVAIL  REFER  MOUNTPOINT
rpool             1.50G   448G    96K  /rpool
rpool/ROOT        1.49G   448G    96K  /rpool/ROOT
rpool/ROOT/pve-1  1.49G   448G  1.49G  /
rpool/data          96K   448G    96K  /rpool/data
```

`rpool/ROOT/pve-1` mounted at `/` is your running system; `rpool/data` is where VM disks would land if you used the default pool — which, per A7, you will not.

## A7. Claiming a second drive as VM storage

Dedicate the second drive entirely to VM disks rather than sharing the boot drive. This gives VMs their own bandwidth and puts the hypervisor's root filesystem in an independent failure domain — separate reinstall paths, separate blast radius.

```bash
zpool create -o ashift=12 vmdata /dev/sdX
zfs set compression=lz4 vmdata
pvesm add zfspool vmdata --pool vmdata --content images,rootdir --sparse 1
```

None of these print anything on success — silence is the expected result. Verify explicitly:

```bash
zpool status vmdata
```

```
  pool: vmdata
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        vmdata      ONLINE       0     0     0
          sdb       ONLINE       0     0     0

errors: No known data errors
```

`state: ONLINE` with zero error counters, and the whole disk (`sdb`, not `sdb1`) as the single vdev — which is what you want when ZFS owns the entire device.

```bash
pvesm status
```

`vmdata` should now appear alongside `local` and `local-zfs`, with `Status: active`:

```
Name             Type     Status           Total            Used       Available        %
local             dir     active        98497780         2841524        90621136    2.88%
local-zfs     zfspool     active       450000000            2400       449997600    0.00%
vmdata        zfspool     active       938000000              96       937999904    0.00%
```

From here on, explicitly choose this pool as the storage target for every VM you create, rather than the default pool on your boot drive.

## A8. Bridges

The LAN bridge exists from install. Add the isolated cluster bridge:

```
auto vmbr1
iface vmbr1 inet manual
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 2-4094
```

Apply with `ifreload -a`, then verify:

```bash
ip -br a show vmbr1
```

```
vmbr1            UP
```

**`UP` with no IP address listed is exactly right** — no host IP, no physical port, a pure virtual switch. It will look "empty" compared to `vmbr0` and that is the design: the head node VM, not the hypervisor, routes this network.

**Pin your interface names now, before adding any NICs later.** Predictable naming (`enp6s0`, etc.) is stable across reboots but keyed to PCI topology, so it can shift when you add or move a card. A shifted name means your bridge references a port that no longer exists — which looks like a network outage but is really an identity mismatch, the same category of mistake as trusting a drive letter. Fix it once with `.link` files keyed to MAC address:

```bash
ip -o link show                     # list interfaces
ip link show <iface>                # note the link/ether MAC

cat > /etc/systemd/network/10-lan0.link <<'EOF'
[Match]
MACAddress=aa:bb:cc:dd:ee:ff
[Link]
Name=lan0
EOF
```

Semantic names pay off later (`lan0`, `cluster0`, `rdma0`). Update `/etc/network/interfaces` to match and reboot once so `systemd-udevd` applies the rename.

## A9. NAS and ISOs

1. On the NAS, create two NFS exports — one for the hypervisor datastore, one held in reserve for the object-storage stage. Restrict them to the hypervisor's IP where the UI allows.
2. `pvesm add nfs nas-pve --server <NAS-IP> --export /nfs/pve --content backup,iso,vztmpl`
3. Pull ISOs (GUI: *→ ISO Images → Download from URL*): a recent Ubuntu Server LTS, and a Rocky/AlmaLinux minimal if you plan to run RPM-based identity tooling in Stage 6. Keep local copies too, so template work does not depend on NAS health.
4. Expectation setting: a NAS behind gigabit tops out near 112 MB/s. Every time that annoys you, it is the visceral argument for why production storage networks run at 100/200/400 GbE — and Stage 3 turns that annoyance into measurements.

## A10. GPU passthrough (its own evening)

If your lab host has no integrated graphics, the local console goes dark permanently once the GPU is claimed for passthrough. That is expected and is why the lab is designed to be managed over SSH and the web UI.

1. **Map the IOMMU groups:**
```bash
for g in /sys/kernel/iommu_groups/*; do
  echo "Group ${g##*/}:"
  for d in "$g"/devices/*; do echo -e "\t$(lspci -nns "${d##*/}")"; done
done
```
Modern NVIDIA cards are **multi-function devices** — VGA, HDMI audio, and on Turing and later often a USB-C controller and its UCSI companion. Ideally all functions sit alone in one IOMMU group. Consumer AMD boards are usually clean; if yours are not, look for an "ACS Enable" BIOS option before considering the ACS-override kernel patch — and understand why that patch is a security compromise (it asserts DMA isolation the hardware does not actually guarantee).

2. **Kernel cmdline:** append `iommu=pt` to the same file you used for `nomodeset` in A3, then re-apply (`proxmox-boot-tool refresh`, or `update-grub` on a GRUB system).

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
options vfio-pci ids=<PCI IDs from lspci -nn, comma-separated> disable_vga=1
softdep nouveau pre: vfio-pci
softdep nvidia pre: vfio-pci
EOF

update-initramfs -u -k all && reboot
```
(Older guides list `vfio_virqfd`; it merged into vfio core in kernel 6.2+.)

4. **Verify.** Before binding, the card shows a host graphics driver (or none at all):

```
0a:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU102 [GeForce RTX 2080 Ti] [10de:1e04] (rev a1)
	Subsystem: ... [1462:3715]
	Kernel driver in use: nouveau
	Kernel modules: nvidiafb, nouveau
```

After the reboot:

```bash
lspci -nnk -s 0a:00
```

```
0a:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU102 [GeForce RTX 2080 Ti] [10de:1e04] (rev a1)
	Subsystem: ... [1462:3715]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau
0a:00.1 Audio device [0403]: NVIDIA Corporation TU102 High Definition Audio Controller [10de:10f7] (rev a1)
	Kernel driver in use: vfio-pci
0a:00.2 USB controller [0c03]: NVIDIA Corporation TU106 USB 3.1 Host Controller [10de:1ad6] (rev a1)
	Kernel driver in use: vfio-pci
0a:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU106 USB Type-C UCSI Controller [10de:1ad7] (rev a1)
	Kernel driver in use: vfio-pci
```

Two things to read carefully. **`Kernel driver in use: vfio-pci` must appear for every function** (`.0` through `.3` here) — one function still bound to a host driver will block the passthrough. And **`Kernel modules:` still listing `nouveau` is normal and not a problem** — that line lists drivers that *could* bind, not what is bound.

5. **Know your way back.** If you need the local console again, edit the boot entry and append `modprobe.blacklist=vfio_pci` for that one boot: vfio never binds, the normal framebuffer driver takes over, console returns. Learn this before you need it.

## A11. The GPU VM and a reusable template

**GPU passthrough VM** (adjust IDs and paths):
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

Details that matter: `--balloon 0` with fixed memory is **mandatory** — passthrough pins guest memory, so ballooning accomplishes nothing and only confuses host accounting. Specifying the PCI address without a function suffix passes **all** functions of a multi-function card. Keep the default virtual display alongside the GPU so noVNC stays available as a fallback console; CUDA does not care which display is primary. The `tpmstate0` line gives this VM a **virtual** TPM 2.0 — this, not host fTPM, is what Stage 9's measured-boot work uses.

Inside the guest, install the vendor's recommended server-branch driver and confirm with `nvidia-smi`:

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.xx.xx              Driver Version: 570.xx.xx      CUDA Version: 12.8     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 2080 Ti     Off | 00000000:01:00.0 Off  |                  N/A |
|  0%   35C    P8              20W / 250W |      1MiB / 11264MiB  |      0%      Default |
+-----------------------------------------+------------------------+----------------------+
```

One detail that surprises people: **the PCI address inside the guest is the virtual one** (`01:00.0` here), not the host's `0a:00.0`. The guest has its own PCI topology; it does not inherit the host's addressing.

**Cloud-init template**, so future VMs clone instead of being hand-installed:
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

Test it:

```bash
qm clone 9000 150 --name scratch1 --full
```

```
create full clone of drive scsi0 (vmdata:base-9000-disk-0)
transferred 0.0 B of 20.0 GiB (0.00%)
transferred 210.0 MiB of 20.0 GiB (1.03%)
...
transferred 20.0 GiB of 20.0 GiB (100.00%)
```

Start it, and it should reach a login prompt in a few seconds and accept your SSH key with no password. Then clean up:

```bash
qm destroy 150
```

If the clone boots but refuses your key, the cause is almost always that `--sshkeys` pointed at a file that did not exist when you created the template — cloud-init silently proceeds with no key rather than failing loudly.

## A12. Backups

Schedule backups to the NAS datastore (weekly, zstd, snapshot mode is fine for a lab), plus an occasional manual dump to local storage as an on-box copy.

**Do one restore drill before moving on.** A backup you have never restored is a rumor, not a safety net. This is also a question you should hope an interviewer asks.

**Part A exit criteria:** hands-free cold boot to a console with no manual intervention; stale boot entries cleaned; `ethtool` reports `Link detected: yes` at full negotiated speed; the web UI answers on `:8006` and SSH key auth works from the workstation; repositories fixed so `apt update` runs clean with no 401; ARC capped and confirmed; second drive wiped to a bare device and dedicated to VM storage; isolated bridge up and VLAN-aware; interface names pinned; NAS datastore `active` in `pvesm status`; every GPU function on `vfio-pci`; `nvidia-smi` clean inside the passthrough VM; a template clone boots and accepts your key; one backup successfully restored.

---

# Part B — Study Stages

Each stage builds one coherent skill area and closes with **self-test questions** — the kind a technical interviewer actually asks. Several stages also include a **deliberate failure** exercise: breaking something on purpose and watching exactly how it fails is worth more than three successful runs, and it is the difference between "I installed this" and "I operate this."

## Stage 1 — Linux and Python Fundamentals  `[In progress — ongoing]`

The foundation everything else assumes. Needs no lab; start today.

Daily practice: pick a topic, explain it aloud as if to an interviewer, then verify. Core topics:

- **Boot flow on a UEFI Linux machine:** firmware → bootloader → kernel → init, and who verifies what at each step. (Part A's `nomodeset` and boot-entry cleanup give you a first-person story here.)
- **`ssh host` end to end:** DNS resolution, TCP handshake, key exchange, authentication, pty allocation, shell.
- **Process and memory internals:** `/proc` exploration, `strace` on a misbehaving command, signals, zombie vs orphan processes, page cache vs buffers in `free`, and the cgroup v2 hierarchy under `/sys/fs/cgroup`.
- **Concurrency in Python for systems work.** Three facts worth knowing precisely rather than approximately:
  1. **CPython's `hashlib` releases the GIL** for buffers above roughly 2 KB, so threaded hashing of multiple files genuinely parallelizes across cores. This is one of the handful of "CPU-bound work that threads handle fine" exceptions, because the heavy lifting happens in a C extension that drops the lock.
  2. **`Executor.map()` yields results in input order**, which creates head-of-line blocking during consumption — one slow item delays everything behind it. It also submits or collects work more eagerly than people expect (the input iterable is consumed up front unless you use the newer `buffersize` parameter), so it is not simply "lazy." `submit()` plus `as_completed()` gives you explicit per-task completion and per-task error handling, which is what you want for anything fleet-scale.
  3. **A second run over the same data being much faster is almost always the OS page cache.** That speedup disappears when the working set exceeds RAM, when data sits behind a cold tier (tape, object storage, a spun-down array), or when the second run lands on a different machine with a cold cache.
- **Practice katas:** a streaming log parser that never loads the whole file; a parallel file-checksum tool built both ways (thread pool and process pool) with an argument for which is right *given where the files actually live* — local NVMe and a gigabit NAS have different answers; a `subprocess` wrapper with real timeout and retry handling.

**Self-test:** Walk through everything between pressing the power button and a login prompt on a UEFI Linux machine. Why can threaded file-hashing outperform sequential hashing despite the GIL? What exactly differs between `Executor.map()` and `submit()` + `as_completed()`, and when does that difference bite? Why might a second identical run of a data-processing job be dramatically faster — and under what three conditions would it not be?

## Stage 2 — SLURM: Build It, Then Operate It  `[Designed]`

Clone VMs from your template rather than hand-installing — faster, and closer to how real fleets are built.

### 2a. Build the cluster

- **Head node:** a VM with a second NIC on the isolated bridge, acting as the cluster's router (NAT for outbound), DHCP/DNS server (`dnsmasq` is simple and sufficient), and NFS server exporting `/home` and `/apps`.
- Clone 3–4 lightweight compute node VMs.
- Install SLURM (`slurm-wlm` from your distro is fine for a lab): `slurmctld` plus `slurmdbd` and a small database on the head node.

**Resource enforcement — get this right, because the common version is incomplete.** `ProctrackType=proctrack/cgroup` tracks processes, but tracking is not enforcing. To actually constrain what a job can use you need the task plugin and the cgroup constraints together. In `slurm.conf`:

```ini
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
ProctrackType=proctrack/cgroup
TaskPlugin=task/cgroup,task/affinity
JobAcctGatherType=jobacct_gather/cgroup
```

And in `cgroup.conf`:

```ini
CgroupPlugin=autodetect
ConstrainCores=yes
ConstrainRAMSpace=yes
ConstrainDevices=yes
```

`ConstrainDevices=yes` is what makes `--gres=gpu:1` mean something — without it a job can see every GPU on the node regardless of what it requested.

- Add Lmod with a small module tree and Apptainer for containerized workloads.
- A `gpu` partition once your GPU VM exists. Note that distro SLURM builds are frequently **not** compiled against NVML, so `AutoDetect=nvml` in `gres.conf` may silently fail; a manual `gres.conf` entry (`Name=gpu File=/dev/nvidia0`) is the honest fallback, and knowing that build-flag nuance is itself a good interview answer.

**Deliberate failure — memory enforcement.** Submit a job with `--mem=512M` that allocates 2 GB. Then observe the full picture: `dmesg` for the kernel OOM message naming the cgroup, `sacct -j <id> --format=JobID,State,ExitCode,MaxRSS` for how SLURM records it, and the cgroup's own `memory.events` counters. Then do the more interesting version: run the same overallocation inside a **multi-rank step** and watch whether the entire step dies or just the offending rank — that behavior, and how to change it, is a real production conversation.

### 2b. Operate the scheduler

Building SLURM proves jobs can run. Operating it is what separates "I installed a scheduler" from "I run a shared compute service," and it is where senior interviews go. Work through:

- **Accounting structure:** `sacctmgr` accounts, users, and associations. Build a small hierarchy (two labs, several users) and understand what an association actually is.
- **Fairshare:** set differing shares between accounts, submit competing workloads, and watch priority evolve with `sprio` and `sshare`. Explain why a heavy recent user drops in priority.
- **QOS:** create two or three (e.g. `normal`, `high`, `debug`) with different limits, priorities, and preemption behavior. Enforce per-user and per-account limits (`MaxJobs`, `MaxTRES`, `GrpTRES`).
- **Preemption:** configure a high-priority QOS that preempts a low-priority one. Then the realistic wrinkle: make the low-priority jobs checkpoint-friendly so preemption suspends or requeues rather than destroying work.
- **Backfill:** understand how `sched/backfill` fills gaps ahead of large pending jobs, why accurate `--time` limits are what make it work, and what happens to throughput when everyone requests the maximum walltime.
- **Reservations:** create a maintenance reservation, watch it drain scheduling, and use a reservation to hold nodes for a specific account.
- **Job arrays:** submit a 500-task array; understand throughput differences vs 500 individual submissions, and how array tasks appear in accounting.
- **Node lifecycle:** `scontrol update NodeName=... State=DRAIN Reason="..."`, resume, and the DOWN/DRAINED/FAIL states. Configure a `HealthCheckProgram` that marks a node DOWN on a synthetic failure, then recover it.
- **GRES failure handling:** simulate a GPU disappearing (stop the VM, or unbind the device) and watch what happens to queued and running GPU jobs.
- **Accounting reports:** `sacct` for jobs, `sreport` for utilization by account and user over a window. Answer "who used what last month" without hand-editing anything.
- **Controller and database recovery:** know what `StateSaveLocation` holds, then practice losing it — stop `slurmctld`, move the directory aside, restart, and watch what happens to the running and queued workload. Separately, stop `slurmdbd` while jobs run and observe how accounting catches up when it returns.

**Self-test:** What does cgroup v2 actually do when a job exceeds its memory limit — mechanism, not outcome? Trace a job from `sbatch` to running process across `slurmctld`, `slurmd`, and `slurmstepd`. Why is tracking (`proctrack`) insufficient without `task/cgroup`? Explain fairshare to a researcher who thinks the scheduler is broken because their job is behind someone else's. Why do accurate walltime requests improve *everyone's* throughput under backfill? What breaks on a shared NFS home directory under `root_squash`, and why does that setting exist at all? You lost `StateSaveLocation` — what is recoverable and what is not?

## Stage 3 — HPC Storage: Metadata, Failure Modes, and Parallel Filesystems  `[Designed]`

A guide with HPC in the title cannot treat storage as "we exported `/home` over NFS." Storage is where most real HPC support tickets originate, where the least intuitive failures live, and where senior candidates separate themselves — because almost everyone can describe a filesystem, and very few can explain what happens to 500 running jobs when the metadata server stops answering.

This stage is deliberately placed right after the scheduler, because Stage 2 just gave you shared `/home` and `/apps` and you should understand what you actually built.

### 3a. The distinction that explains everything: metadata vs data

Every filesystem operation is either **data** (read/write bytes in a file) or **metadata** (create, stat, open, rename, unlink, chmod, directory listing). They scale completely differently, they bottleneck on different hardware, and conflating them is the single most common source of "the filesystem is slow" misdiagnoses.

Make it concrete on your NFS share:

```bash
# data path: one big file, sequential
fio --name=seq --rw=write --bs=1M --size=4G --numjobs=1 --directory=/shared/bench

# metadata path: many tiny files
fio --name=meta --rw=randwrite --bs=4k --size=64M --nrfiles=20000 \
    --file_service_type=random --directory=/shared/bench
```

Watch throughput collapse on the second even though total bytes moved are trivial. Then explain, out loud, why: the first is bandwidth-bound on the link, the second is round-trip-bound on metadata operations, and no amount of extra network bandwidth fixes the second.

### 3b. Inode exhaustion — a filesystem full of nothing

A filesystem can be 3% full by bytes and completely unwritable. Build one deliberately:

```bash
# a small filesystem with deliberately few inodes
truncate -s 1G /tmp/small.img
mkfs.ext4 -N 1024 /tmp/small.img      # only 1024 inodes
mkdir -p /mnt/small && mount -o loop /tmp/small.img /mnt/small

# now exhaust them
for i in $(seq 1 2000); do touch /mnt/small/file_$i 2>/dev/null || { echo "died at $i"; break; }; done

df -h  /mnt/small     # plenty of space
df -i  /mnt/small     # zero inodes free
```

The error is `ENOSPC` — "No space left on device" — while `df -h` shows the disk nearly empty. Every storage administrator meets this eventually, usually via a user who ran a pipeline that wrote ten million tiny intermediates. Know it before it wakes you up. Then know the follow-up: on ext4, inode count is fixed at `mkfs` time and cannot be grown; on XFS it is dynamic but bounded by `imaxpct`; on ZFS and most parallel filesystems the concept is different again.

### 3c. Small-file storms and why they hurt shared storage

A pipeline that opens 40,000 small files per second is not moving much data — it is issuing 40,000 metadata operations per second, and on a shared filesystem those queue behind everyone else's. Simulate one with `mdtest` (part of the IOR suite) or a simple parallel `find`/`stat` loop, and watch what it does to a *second* client's interactive responsiveness on the same share.

This is why real sites push users toward: tar-and-stage patterns, local scratch for intermediates, container image files instead of unpacked trees, and per-directory file-count limits. It is also why "just add more bandwidth" is the wrong answer and why parallel filesystems put metadata on dedicated servers.

### 3d. Local scratch vs shared storage

Give each compute node a local scratch disk and export a shared filesystem, then run the same workload against both and measure. The lesson is not "local is faster" — it is understanding the tradeoff a scheduler has to mediate:

- **Local scratch:** fast, no contention with other nodes, but invisible to other nodes, lost at job end, and requires an explicit stage-in/stage-out step.
- **Shared:** visible everywhere, survives the job, but every node's I/O contends with every other node's.

Wire this into SLURM properly: use a prolog/epilog to create and clean a per-job scratch directory, expose it as an environment variable, and write a job script that stages in, computes on local scratch, and stages results out. That workflow *is* the answer to a very common interview question about how you would speed up an I/O-bound cluster workload.

### 3e. Quotas

Set user and group quotas on your shared filesystem, exceed them deliberately, and observe how the failure surfaces to a running job (usually as a write error at an arbitrary point, not a friendly message). Then look at the difference between:

- **User/group quotas** — per-identity limits.
- **Project or fileset quotas** — per-directory-tree limits, which is what most HPC sites actually want ("this lab gets 50 TB" rather than "this user gets 50 TB").
- **Soft vs hard limits and grace periods**, and why a soft limit with a grace period is kinder to long-running jobs.

### 3f. Mount failure behavior — the most underrated topic here

Stop the NFS server while a client job is writing. What happens depends entirely on mount options, and this is worth experiencing rather than reading:

- **`hard` (the default):** the client retries **forever**. Your job hangs in uninterruptible sleep (`D` state). It does not fail, it does not error, it does not respond to `SIGKILL`. When the server comes back, the write completes as if nothing happened. This is the right default for data integrity and the reason a storage outage produces a cluster full of unkillable processes.
- **`soft`:** the client gives up after a timeout and returns an I/O error to the application — which usually means silent data corruption in applications that do not check write return codes. Fast failure, real risk.
- **`intr`** (largely a no-op on modern kernels, which allow killing on hard mounts under some conditions) and `retrans`/`timeo` tuning shape the middle ground.

Run the experiment both ways. Watch `ps` show `D`-state processes. Try to kill one. Bring the server back and watch the job resume mid-write. You will never again be confused about why a storage blip produces a wave of stuck jobs.

### 3g. NFS consistency semantics

Two clients, one shared file. Write from A, read from B, and measure how long B takes to see the change. That delay is **attribute caching** (`acregmin`/`acregmax`/`actimeo`), and it is why NFS is described as **close-to-open consistent**: guarantees exist between one client closing a file and another opening it, not on arbitrary concurrent access.

Then meet **`ESTALE`**: delete and recreate a file on the server while a client holds it open, and watch "Stale file handle" appear. Understand that an NFS file handle references an inode, not a path — which is exactly why the error exists and why it is not fixable by re-running `ls`.

The practical consequence for HPC: applications that assume POSIX-strict semantics across nodes (many MPI programs writing to one shared output, naive lock files) behave differently on NFS than on a local filesystem or a true parallel filesystem. That difference is a legitimate interview conversation.

### 3h. Parallel filesystems — concepts, then a real one

The concepts first, because they transfer across products:

- **Separated metadata and data paths** — dedicated metadata servers/targets vs object/data servers, precisely because of 3a.
- **Striping** — one file distributed across many targets so a single file's bandwidth exceeds one server's. Stripe width and stripe size are tunable per file or directory and matter enormously for large sequential I/O.
- **Failure domains** — what happens when one storage target, one metadata server, or one network path dies; which of those is a full outage and which is a degraded read.
- **Client-side caching and lock management** — how the filesystem keeps many clients coherent, and what that costs.

Then build one. **BeeGFS runs perfectly well in VMs** and is by far the most approachable parallel filesystem for a lab: separate `mgmtd`, `meta`, and `storage` services (put them on different VMs so the roles are real), plus clients on your compute nodes. Create a directory, set a stripe pattern across two storage targets, write a large file, and confirm with BeeGFS's own tools that it is actually striped. Then kill one storage service and observe exactly which operations still work.

**Lustre** is the other major open option but is substantially heavier to stand up; **IBM Storage Scale (GPFS)** is not practical to distribute for a home lab, but its concepts — filesets, policy-driven ILM, storage pools, NSD servers and failure groups, `mmhealth` — are worth studying because they appear constantly in research-computing job descriptions. Know the vocabulary and the failure-domain model even where you cannot run the product.

### 3i. Benchmark like you mean it

Three tools, three different questions:

- **`fio`** — flexible single-node I/O patterns. Best for "what can this device or mount actually do" under a specific pattern.
- **`IOR`** — the HPC standard for parallel bandwidth. Runs under MPI across nodes, so it measures aggregate filesystem throughput, not one client's.
- **`mdtest`** (ships with IOR) — the metadata counterpart: creates, stats, and removals per second across ranks.

Run all three against both your NFS share and (once built) your BeeGFS mount. Record the numbers. The comparison — where NFS holds up fine, and exactly where it falls off — is the empirical backing for "why do parallel filesystems exist," and having your own measurements makes that answer yours rather than recited.

### 3j. Where storage meets the scheduler

Finally, connect the two halves of Stages 2 and 3:

- Nodes with local scratch as a SLURM feature/constraint, so jobs that need it request it.
- `--tmp` resource requests and what happens when scratch fills.
- Why staging data close to compute changes job placement decisions, and how a scheduler could (and mostly does not) reason about data locality.
- What a filesystem outage does to your queue, and how a `HealthCheckProgram` that tests the mount can drain a node *before* jobs land on broken storage.

**Self-test:** Explain the difference between the metadata and data paths, and give an example of a workload that saturates one while barely touching the other. A user reports "disk full" but `df -h` shows 60% free — what do you check and what is your fix? Walk through what happens on a client with a `hard` NFS mount when the server disappears mid-write, and contrast with `soft` — including which one risks data corruption and why. What is close-to-open consistency, and what class of application breaks under it? Why does `ESTALE` happen and why can the application not simply retry the path? What does striping buy you, and what does it cost when a single storage target fails? Given a 40,000-small-files-per-second pipeline, what would you actually change?

## Stage 4 — RDMA, MPI, and GPU Collective Communication  `[Designed]`

This converts "I have heard of RDMA and InfiniBand" into something you have measured. You will not have real RDMA hardware in a typical home lab, but you can build genuine intuition for the concepts and real fluency with the software stack.

1. **OpenMPI under SLURM.** Run the OSU microbenchmarks (`osu_latency`, `osu_bw`, `osu_allreduce`) between compute nodes over plain TCP and record the numbers as a baseline.

2. **Soft-RoCE** — the trick that makes verbs programming possible without special hardware. Any Ethernet NIC can host a software RoCEv2 implementation:
```bash
sudo apt install -y rdma-core ibverbs-utils perftest
sudo modprobe rdma_rxe
sudo rdma link add rxe0 type rxe netdev <your-nic>
ibv_devinfo                          # a real verbs device now exists
rping -s                             # then ib_send_bw / ib_write_bw between two nodes
```
Performance will be **worse** than plain TCP — this is software emulation, not acceleration — and that is fine. The point is hands-on fluency with the verbs API: queue pairs, completion queues, memory regions, GIDs, and the `rdma`/`ibv_*` tooling. Afterward you can speak specifically to what real hardware changes: kernel bypass, NIC-offloaded transport, true zero-copy.

3. **Multi-node GPU collectives.** With two GPU-capable machines (a passthrough VM and your workstation, for instance), boot both into Linux and run NCCL's `all_reduce_perf` between them:
```bash
NCCL_SOCKET_IFNAME=<nic> NCCL_DEBUG=INFO \
mpirun -np 2 -H hostA,hostB ./build/all_reduce_perf -b 8 -e 1G -f 2 -g 1
```
Mixed GPU generations work — NCCL does not require identical hardware, though a real training job would be paced by the slower card. Read the output properly: **algorithm bandwidth (algbw) vs bus bandwidth (busbw)**, where for all-reduce `busbw = algbw × 2(N−1)/N`. On ordinary gigabit you will watch it pin near 110 MB/s — a felt demonstration of why GPU training clusters need RDMA fabrics. Finish with a small PyTorch DDP run across both hosts: a real all-reduce inside a real training loop, with `NCCL_DEBUG=INFO` showing ring construction and chosen transport.

4. **Theory worth knowing cold:** RoCEv2 encapsulates InfiniBand transport semantics in UDP; why that historically wants a lossless fabric; Priority Flow Control and its failure modes (head-of-line blocking, pause storms, deadlock); ECN and DCQCN as the modern congestion-control answer; why a single large flow defeats ECMP hashing and how RDMA transports work around it; ring vs tree all-reduce; in-network reduction on InfiniBand.

5. **One nuance worth stating precisely:** GPUDirect RDMA — letting a NIC read and write GPU memory directly — requires data-center-class GPUs. Consumer GeForce cards route that traffic through host system memory instead. Knowing this distinction, and why it exists, marks real depth.

6. **Optional spend (~$60–90):** a pair of used older-generation RDMA-capable NICs and a direct-attach cable, connected point-to-point, gets you genuine hardware verbs performance instead of software emulation. The single best money this lab can absorb.

**Self-test:** Walk through what happens between posting an RDMA send and its completion being signaled. Contrast InfiniBand's credit-based flow control with RoCE's PFC/ECN approach — mechanisms and failure modes for each. Given a link speed and node count, compute the theoretical bus bandwidth of a ring all-reduce. Why can a consumer GPU not do GPUDirect RDMA, and what changes on a data-center card?

## Stage 5 — Kubernetes with Real GPU Scheduling  `[Designed]`

Kubernetes "familiarity" and "I run a GPU-enabled cluster and understand batch scheduling on it" are very different claims in an interview. This stage earns the second.

- Build with **kubeadm**, not a simplified distribution, at least once — touch every component: container runtime, CNI, control-plane init, node joining, and recovery from things you break on purpose. A CNI like **Cilium** (eBPF-based, can replace kube-proxy) is worth choosing specifically because it generates good "how does this actually work" conversations.
- **Bring a real GPU in:** join your passthrough VM as a worker and install the **NVIDIA GPU Operator** via Helm (with the in-guest driver already present, disable the operator's driver management). Schedule a pod requesting a GPU resource. Then configure **time-slicing** to oversubscribe one GPU across pods, and be ready to explain the isolation tradeoffs versus MIG (hardware-partitioned, data-center GPUs only) and MPS (a different sharing model again).

> **Hardware compatibility checkpoint.** The GPU Operator and DCGM are built, documented, and tested against **data-center GPUs and data-center driver branches**. On consumer GeForce hardware (a 2080 Ti or 4090, as in this lab) parts of the stack work and parts may not — the device plugin generally functions, while DCGM's richer telemetry and diagnostics are neither guaranteed nor supported. Treat this path as an experiment with a fallback, not a prescription. **If the operator or DCGM misbehaves on your card, do not lose the stage to it:** fall back to installing the NVIDIA device plugin alone, and gather metrics from `nvidia-smi`/NVML through a lightweight exporter instead. You will still learn the scheduling, admission, and quota lessons, which are the transferable part. Being able to *explain* this consumer-vs-data-center split is itself a strong interview signal.

- **Batch and gang scheduling** — the conceptual bridge from HPC schedulers to Kubernetes: install **Kueue**, define a ClusterQueue with CPU/memory/GPU quotas and a LocalQueue, submit jobs, and watch suspension and admission. Know **Volcano** by name and be able to contrast its PodGroup gang model with Kueue's quota-and-admit model.
- **Observability:** a Prometheus and Grafana stack with node and GPU metrics gives you a real dashboard of utilization, queue depth, and node health — useful in itself and a good demonstration centerpiece.
- **GPU fleet operations, worth an evening:** `nvidia-smi topo -m` and what the interconnect codes mean; running a GPU stress tool while watching thermals; reading vendor GPU error codes (NVIDIA's XID taxonomy) well enough to distinguish "the application misbehaved" from "reset the GPU" from "replace the hardware"; cordon and drain patterns for planned GPU maintenance.

**Self-test:** Trace a packet from one pod to a pod on another node under an eBPF-based CNI — what replaced the traditional kube-proxy path? How does the device plugin API let kubelet discover GPUs? Why does a naive scheduler deadlock a two-pod MPI job, and how does gang scheduling prevent it? Contrast bin-packing and preemption between an HPC scheduler and the default Kubernetes scheduler. A GPU pod is stuck Pending — walk your full debugging tree out loud.

## Stage 6 — Identity, Access Control, and Secure Provisioning  `[Designed]`

Making compute available "securely and conveniently" is a real requirement anywhere shared infrastructure exists. This stage builds the identity layer everything else should depend on.

- Stand up a directory service — **FreeIPA** suits a Linux-centric estate, bundling Kerberos, LDAP, DNS, host-based access control, and sudo rules behind a scriptable API. (Samba as an AD-compatible domain controller is the alternative if you want Windows-style semantics, or want to practice a trust relationship between the two.)
- Enroll every node; use **automount** maps for home directories rather than static mounts, since that is closer to real HPC practice.
- Build access rules deliberately and **test the negative case** — confirm that a user outside the right group is actually denied, not merely that authorized users succeed. That habit separates people who configure access control from people who verify it.
- Tie identity to scheduling: sync directory groups into SLURM accounting, so who someone is determines what they can run and how it is tracked.
- A useful capstone: a deliberately boring **onboarding pipeline** — a small service that takes a structured "new user" request, maps a role to groups and a scheduler account via a policy table, creates the user, and writes an audit record. It does not need to be clever; it needs to show you understand identity as the control plane rather than an afterthought.
- Prove the chain end to end: authenticate with Kerberos, SSH in with no password using that ticket, submit a job, and confirm accounting attributes it correctly.

**Self-test:** Walk a Kerberos exchange from initial request to usable service ticket — what is inside a ticket-granting ticket? Trace how `id` resolves a username through the name service switch and any caching layer back to the directory. How does ticket-based SSH differ operationally from key-based? What is the failure mode when clocks drift between client and KDC? Contrast an LDAP/Kerberos directory's model with Active Directory's.

## Stage 7 — Network Boot and Infrastructure as Code  `[Designed]`

- Build a real **PXE boot chain**, ideally one that respects UEFI Secure Boot rather than disabling it: a signed shim, a signed network-bootable GRUB, and kernel/initrd/image served over HTTP rather than TFTP (which chokes on anything large). One gotcha worth knowing in advance: autoinstall datasource URLs often contain a semicolon, which GRUB's parser treats as a statement separator unless the whole argument is quoted.
- Pair it with **Ansible** roles so a freshly network-installed node ends up fully configured — networking, identity enrollment, scheduler agent, monitoring — with no manual steps.
- Add **OpenTofu** (or Terraform) to define VMs declaratively against your hypervisor, so a single apply creates or destroys lab nodes reproducibly. This is real infrastructure-as-code practice against real infrastructure, which reads very differently from a cloud tutorial.
- If you dual-home any node across both networks, watch for the classic multi-homing trap: two default routes competing. Suppressing the second is a small, specific lesson worth internalizing.

**Self-test:** Walk the PXE process end to end — DHCP options, TFTP or HTTP handoff, the shim-to-GRUB-to-kernel chain, and where cloud-init takes over. Why must your lab's DHCP server never be reachable from your home network? What changes about PXE when Secure Boot is enforced rather than disabled?

## Stage 8 — Self-Hosted Object Storage  `[Planned]`

Most research and ML infrastructure eventually needs an S3-compatible object store alongside file storage. Build one rather than only consuming a cloud provider's.

- Deploy **MinIO** (or a lighter alternative like Garage) backed by real block storage rather than a network filesystem — object stores generally want the stronger consistency and locking guarantees block storage provides, and knowing *why* that recommendation exists is the interview-grade part. Know **Ceph RGW** by name as the large-scale answer.
- Practice the concepts that do not map onto file storage: bucket versioning, lifecycle rules that expire old versions, presigned URLs, bucket policies.
- Point any data-transfer tooling you have built at your own endpoint. Running your own tools against infrastructure you operate is a different kind of understanding than running them against a managed service.

**Self-test:** What is S3's actual consistency model today, versus the "eventually consistent" reputation it carries from years ago? Why does multipart upload exist and what do per-part checksums protect against? Contrast erasure coding and replication in storage overhead, rebuild cost, and failure-domain tolerance. What breaks when an application assumes POSIX semantics against an object store?

## Stage 9 — Boot and Firmware Security  `[Planned]`

The best return on unusual effort: almost nobody builds hands-on experience here, so it converts vague "familiar with Secure Boot and TPM concepts" into something you have actually done.

- **Secure Boot and MOK enrollment.** In a VM with a virtual TPM and OVMF firmware, try to load an unsigned kernel module under Secure Boot and watch it be refused; then generate your own key, enroll it with `mokutil`, sign the module, and watch it load. This is exactly the workflow real fleets hit whenever a vendor driver is not signed by a recognized authority.
- **Measured boot with a TPM.** Read PCR values and the boot event log; then bind a LUKS volume's unlock key to a PCR with `systemd-cryptenroll`, watch it auto-unlock, then deliberately change something that affects that PCR and watch it fall back to a passphrase. Understand why binding to a PCR that reflects **Secure Boot policy state** survives routine updates while binding to exact boot-binary hashes does not — choosing wrong means bricking your own unlock on the next kernel patch.
- **State the two mechanisms precisely**, because loose phrasing gets picked apart in interviews:
  - **Verified boot (Secure Boot)** validates signatures along the **boot chain** — firmware, bootloader, kernel, and (subject to kernel lockdown policy) kernel modules — and refuses to execute components that fail validation. It does **not** cryptographically approve every user-space binary you later run.
  - **Measured boot** records **defined boot events and component measurements** into TPM PCRs together with an event log. It blocks nothing; it produces an audit trail that supports remote attestation and sealed secrets. It does not hash everything the machine ever executes.
- **Firmware-level Linux, safely.** Do not attempt to flash alternative firmware onto a consumer motherboard — real brick risk, no board support, and saying exactly that is the senior answer. But you can build coreboot for QEMU's emulated hardware with zero risk, and pair it with a minimal Linux-based initramfs (the LinuxBoot approach) that `kexec`s into a full OS — a faithful demonstration of replacing most of a traditional firmware stack with auditable Linux plus `kexec`.
- Know the standard boot-stage vocabulary, and know where the hardware root of trust actually begins on your platform before any general-purpose CPU code runs.

**Self-test:** Narrate power-on to login prompt again, this time naming every verification step and being precise about what is and is not verified. Why does a shim bootloader exist, and what problem does MOK enrollment solve that vendor signing does not? Why bind a LUKS unlock to Secure Boot policy state rather than exact binary hashes? What does a LinuxBoot-style stack replace, and what is the security argument for it?

## Stage 10 — Routing and Fabric Simulation  `[Planned]`

Build a router VM between two virtual network segments; carve VLANs and practice inter-VLAN routing; add a second router and bring up OSPF between them. A tool like containerlab lets you spin up a disposable multi-node routing topology on your workstation for practicing ECMP hashing behavior, which connects directly back to Stage 4's congestion-control material.

**Self-test:** Why did BGP become dominant for large datacenter fabrics rather than a purely IGP-based design? Where do the Layer 2 and Layer 3 boundaries sit in a leaf-spine topology, and why there specifically? What does PFC do to a leaf-spine under incast?

## Stage 11 — Optional: A Heterogeneous ARM Node  `[Planned]`

Real fleets are increasingly multi-architecture, and "our software ran fine until the ARM nodes arrived" is a common operational story. Any ARM box on your network can teach that whole lesson class. The worked example here is an Apple Silicon Mac mini that keeps its day job as a home-theater machine.

**Prepare it so you never touch it physically again.** On macOS, enable Remote Login and Screen Sharing. Then stop it sleeping out from under your cluster — the single most likely way a part-time node sabotages a lab:
```bash
sudo pmset -a sleep 0
sudo pmset -a autorestart 1
```

**A Linux VM does the real work** — macOS cannot be a SLURM or Kubernetes node itself (no Linux kernel, no cgroups). Use **UTM** (GUI, Apple Virtualization backend, bridged networking is a dropdown) or **Lima** (scriptable, but its default user-mode networking leaves the VM unreachable from the rest of the LAN — install `socket_vmnet` or the exercise fails at step one). Bridge it to your LAN and reserve its address.

**Route it to the cluster network** with a static route via your head node, and make sure the head node's NAT rule only masquerades traffic actually leaving for the internet — not east-west traffic between the cluster and your LAN. If it NATs everything, every cluster node appears to the ARM VM as the head node's address, which breaks direct addressing and any CNI tunnel expecting real node IPs. "Only NAT what is leaving the building" is a real router discipline, learned here in miniature.

**In SLURM:** install the scheduler agent, match versions with your controller, and define the node with an architecture feature and its own partition. Then run the most instructive demo of the stage: compile a hello-world on an x86 node, submit it to the ARM partition, and watch it die with **`Exec format error`**. That single error is the entire reason heterogeneous sites maintain per-architecture software trees — then extend your module path to include the architecture so each resolves its own builds.

**In Kubernetes:** a standard join adds it as a worker. Then: schedule onto it deliberately with a `nodeSelector` on architecture; push an amd64-only image at it and observe the failure (either a pull-time "no matching manifest" or a container that starts and immediately exec-format-errors, depending on how the image was built); fix it properly with a multi-architecture manifest via `docker buildx`; and optionally taint the node so only workloads that tolerate ARM land there — the polite pattern for a part-time node.

**Honest limits:** an Apple GPU speaks Metal, not CUDA, so this node adds *architecture* diversity, not GPU capacity, and never participates in Stage 4's NCCL work. Throughput is modest and availability is part-time. None of that matters for what it teaches.

**Self-test:** At exactly what layer does an amd64-only workload fail on an arm64 node, and why are there two different failure modes depending on how the image was built versus pulled? What is a multi-architecture manifest list and how does a runtime resolve it? Compare SLURM's feature/constraint mechanism with Kubernetes node labels and selectors — same idea, different ecosystems; where do they diverge?

---

# Part C — The Signature End-to-End Demonstration

Once several stages exist, wire them into one coherent chain. This is worth far more than the sum of its parts, because it proves you understand how the pieces fit rather than each piece in isolation — and because it is the thing you can narrate in an interview while someone interrupts you with questions.

A reasonable shape:

| # | Beat | What it proves |
|---|---|---|
| 1 | An infrastructure-as-code apply builds a new compute node from nothing; it network-boots and joins the cluster unattended | provisioning and IaC (Stage 7) |
| 2 | An onboarding request creates a real directory user with correct group memberships and a scheduler account | identity as control plane (Stage 6) |
| 3 | That user authenticates with a Kerberos ticket and SSHes in with no password | authentication chain (Stage 6) |
| 4 | They submit a job that stages data to local scratch, computes, and stages results back to shared storage | storage-aware workflow (Stage 3) |
| 5 | The job lands on the GPU node, correctly fenced by cgroups | scheduling and enforcement (Stages 2, 5) |
| 6 | A dashboard shows utilization; accounting shows correct attribution | observability and accounting (Stages 2, 5) |
| 7 | The same tooling tears the node back down | full lifecycle, cattle not pets |

Rehearse narrating it end to end until you can hold the thread under questioning. If an interviewer stops you at any beat and digs, the corresponding stage's self-test questions are your preparation for exactly that.

---

# Appendices

## A. Pacing

No single correct pace — go faster where a stage is close to your existing experience, slower where it is genuinely new. A rough shape for evenings and weekends:

| Stage | Time |
|---|---|
| Part A — hypervisor build (A1–A12) | 1 week, including a dedicated evening for A10 (GPU passthrough) |
| — of which A4 (network first contact) | 30 minutes, and it makes everything after it easier |
| Stage 1 — fundamentals | ongoing throughout; never really "done" |
| Stage 2 — SLURM build and operations | 2–3 weeks (2a builds it; 2b is where the depth is) |
| Stage 3 — storage | 2 weeks |
| Stage 4 — RDMA/MPI/NCCL | 2–3 weeks |
| Stage 5 — Kubernetes and GPU | 2–3 weeks |
| Stage 6 — identity | 1–2 weeks |
| Stage 7 — PXE and IaC | 1–2 weeks |
| Stage 8 — object storage | a few evenings |
| Stage 9 — boot and firmware security | ongoing background |
| Stage 10 — routing | ongoing background |
| Stage 11 — ARM node (optional) | a weekend, best after Stages 2 and 5 |

## B. Break-Glass Cheatsheet

Recovery moves worth knowing *before* you need them:

- **Black screen right after kernel handoff** → at the boot menu, `e`, append `nomodeset`; once confirmed, persist it in your bootloader's config and re-apply (A3).
- **Console lost to GPU passthrough** → boot-menu edit, append `modprobe.blacklist=vfio_pci` for one boot; the normal display driver takes back over (A10).
- **A cmdline edit "did not take"** → you edited the wrong bootloader's file, or skipped the apply step. `proxmox-boot-tool status` settles which one you are on; remember it reports `uefi` for systemd-boot.
- **Cannot reach the web UI** → work the ladder in order rather than guessing: `ethtool <nic>` for `Link detected: yes`; `ip -br a` for an address **on `vmbr0`, not on the physical NIC**; `ip link show master vmbr0` to confirm the NIC is actually enslaved to the bridge; then `ping` gateway → `1.1.1.1` → a hostname to separate local, routing, and DNS failures (A4).
- **`bridge-ports` names an interface that does not exist** → the classic cause of "configured but unreachable." Compare `/etc/network/interfaces` against `ip -br link` character by character, fix, `ifreload -a` (A4d).
- **`apt update` fails with 401 Unauthorized** → the enterprise repository without a subscription. Disable it and add no-subscription (A6). Check for leftover legacy `.list` files as well as `.sources`.
- **Arrow keys typing `A`/`B`/`C`/`D` in the editor** → you are in `vim.tiny`. `apt install vim`, or `:set nocompatible` (A4g).
- **Two ZFS pools with the same name importable at boot** → the reason a second drive gets label-cleared (not merely partition-deleted) *before* it is ever powered on beside a fresh install (A5). If you are already at the initramfs prompt: `zpool import` to list numeric IDs, then `zpool import -N <id> rpool` and `exit`.
- **Stale UEFI boot entries** → `efibootmgr -v` to list, `-b XXXX -B` to delete, `-o` to reorder (A3).
- **Stuck at a GRUB rescue prompt** → `set prefix=(hd0,gpt2)/boot/grub`, `insmod normal`, `normal`.
- **Jobs hung in `D` state after a storage blip** → expected behavior on a `hard` NFS mount; they are waiting, not dead, and will resume when the server returns (Stage 3f). Killing them is not the fix; restoring the server is.
- **"No space left on device" with free space showing** → inodes, not bytes. `df -i` (Stage 3b).
- **A VM locked after a failed backup** → unlock it explicitly rather than fighting it manually.
- **Anything genuinely unrecoverable** → this is why you rehearsed a restore in A12. A backup that has never been tested is a hope, not a plan.

## C. Reading List

LLNL's MPI tutorials remain an excellent free resource. The OSU microbenchmark and NCCL/`nccl-tests` documentation are worth reading directly — the bus-bandwidth math in particular is explained clearly in the primary sources. For storage: the IOR and mdtest documentation, the BeeGFS administration guide, and any vendor's parallel-filesystem architecture overview (they are more candid about failure domains than you might expect). SchedMD's own `slurm.conf` and `cgroup.conf` documentation is the authoritative source for Stage 2 and repays close reading. NVIDIA's GPU Operator platform-support page is where to check hardware compatibility claims for Stage 5. Kubernetes and Kueue documentation, and FreeIPA/RHEL Identity Management documentation, cover Stages 5 and 6. On networking theory, the DCQCN congestion-control paper (SIGCOMM 2015) and "RDMA over Commodity Ethernet at Scale" (SIGCOMM 2016) are both approachable and foundational. The coreboot project documentation and the u-root/LinuxBoot pages are the best primary sources for Stage 9.

## D. Cost

Everything here is achievable with free and open-source software on hardware you likely already own. If you want to spend anything, the best value by a wide margin is a pair of used older-generation RDMA-capable network cards and a direct-attach cable — often well under $100 secondhand — which upgrades Stage 4 from software emulation to genuine hardware verbs testing. A spare SSD for dual-booting your workstation is the second-best purchase, since multi-node GPU work needs bare-metal Linux.

---

*This guide is a living document. Stages marked Designed or Planned will be promoted to Validated as they are built and their failure paths verified.*
