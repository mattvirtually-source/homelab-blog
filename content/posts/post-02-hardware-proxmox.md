---
title: "The Hardware: Three Mini PCs, a Switch, and a Plan"
date: 2026-04-14
draft: false
series: ["Series 1 — Building the Foundation"]
tags: ["homelab", "hardware", "proxmox", "gmktec", "ceph", "ubiquiti"]
summary: "Why I chose mini PCs over rack servers, how I configured Proxmox across three nodes with Ceph in mind from day one, and what the hardware stack looks like before networking enters the picture."
showToc: true
TocOpen: false
---

The first decision you make in a home lab tends to be the one that constrains everything else: what hardware are you running this on? For me, that decision sat at the intersection of three competing constraints — compute density, power consumption, and budget. The answer ended up being three small boxes on a small rack, and I don't regret it.

---

## Why Not Rack Servers?

The traditional home lab path is second-hand enterprise gear, Dell PowerEdge R720s, HPE ProLiant DL380s, SuperMicro 2U nodes. You can buy these for a few hundred dollars each on eBay, and the specs are genuinely impressive: dual-socket Xeons, 256+ GB RAM, redundant power supplies, iDRAC/iLO for out-of-band management, 10 GbE onboard.

Basically a few generation old versions of servers I sell every day.

The problem is what you pay for those specs in ways that don't show up on the eBay listing. A single R730 under load draws 300–400 watts. Three of them, running 24/7, adds roughly $150–200/month to the power bill depending on where you live. I have solar power, and while the sun shines in Phoenix most days, it doesn't at night. Add the noise that makes them incompatible with anything resembling a living space, and the total cost of ownership climbs fast.

What I wanted was a cluster I could run continuously, in my home office, without an electricity bill that my household would notice. That shifted the calculus toward mini PCs.

---

## The GMKtec NucBox M5 Ultra

After a few weeks of research, I landed on the [GMKtec NucBox M5 Ultra](https://www.gmktec.com). Three units, each configured with:

| Component | Spec |
|-----------|------|
| CPU | AMD Ryzen 7 7730U (8C/16T, Zen 3, up to 4.5 GHz) |
| RAM | 32 GB DDR4-3200 |
| Primary Storage | 500 GB M.2 NVMe (PCIe Gen 3) |
| Secondary Storage | 512 GB M.2 NVMe (added, see below) |
| Networking | Dual 2.5 GbE (Realtek RTL8125B) |
| Wireless | WiFi 6E + Bluetooth 5.2 (unused in this context) |
| TDP | 15W base |

The spec that closed the deal was the dual 2.5 GbE NICs. Most mini PCs at this price point give you a single NIC. Having two is what makes the networking design I had in mind actually workable — one NIC for general cluster and management traffic, one dedicated to storage. More on that in the next post.

### The CPU: What Matters for Virtualization

The Ryzen 7 7730U is an 8-core, 16-thread Zen 3 processor. It supports AMD-V (AMD's hardware virtualization extension, equivalent to Intel VT-x), and IOMMU for PCI passthrough when enabled in BIOS.

One thing worth knowing before you boot a fresh Proxmox install on any AMD consumer CPU: **SVM Mode is disabled by default in the BIOS**. Without it, KVM virtualization won't work. The setting is buried under Advanced → CPU Configuration on the M5 Ultra's firmware. It took me longer to find it than it should have — it's labeled "SVM Mode" rather than anything that immediately reads as "virtualization."

Enable SVM Mode before you do anything else. Then verify it from the Proxmox shell:

```bash
grep -e 'svm' /proc/cpuinfo | head -1
```

If the output includes `svm` in the flags, you're good. If the line comes back empty, go back into BIOS.

### Power Draw

At idle, each M5 Ultra draws approximately 8–12 watts. Under moderate virtualization load, expect 20–35W. At full bore, you might see 45–55W briefly. Three nodes together at sustained load land somewhere around 90–120W, a fraction of what rack servers would draw. This thing can run continuously without being a second thought on the power bill.

### The Trade-offs

I'll be honest about what I gave up:

- **No out-of-band management.** There's no IPMI, iDRAC, or iLO equivalent here. If a node becomes completely unresponsive, I need physical access. For a home lab, that's fine. For anything resembling production, it would be a problem.
- **DDR4, not DDR5.** The 7730U is a Barcelo-R chip (Zen 3 refresh on a 6nm node), which predates the Zen 4 transition to DDR5. Not a limitation for this use case, but worth knowing.
- **PCIe Gen 3, not Gen 4.** NVMe throughput is capped accordingly. In practice, sequential reads around 3,500 MB/s and writes around 2,600 MB/s, which is more than sufficient for a Ceph-backed lab cluster.

---

## Adding the Second Drive

Each Nucbox M5 Ultra ships with one occupied M.2 slot and one open. I ordered three additional 512 GB NVMe drives, one per node, and installed them before the first boot.

The purpose of that second drive will become clear in the networking and storage post, but the short version: Ceph. When I configure Ceph as the shared storage backend across these three nodes, each node contributes one or more OSDs (Object Storage Daemons). I partitioned the boot drive to give ~400 GB of space for the first OSD, but I wanted to give myself plenty of room for future growth hence the added 512 GB M.2 as a second OSD per node.

---

## The Switch: Ubiquiti 8-Port 2.5 GbE

Compute without appropriate networking is just three isolated boxes. The switch I chose was the [Ubiquiti UniFi Switch Flex 2.5G 8-port](https://store.ui.com). Relevant specs:

- 8x 2.5 GbE RJ45 ports
- 1x 10 GbE SFP+ uplink
- Managed, VLAN-aware
- Fanless

The 2.5 GbE match with the Nucbox NICs is intentional, there's no bottleneck between the nodes and the switch, and 2.5 GbE gives Ceph replication traffic enough headroom to operate without competing with VM traffic for bandwidth. The 10 GbE uplink is currently connected to my home router for internet access.

The switch is configured through UniFi Network Controller, which I plan on running as a VM on one of the Proxmox nodes. That's slightly chicken-and-egg on the first boot, but in practice you just configure the VLANs from a locally connected laptop first, adopt the switch, and then migrate controller management to the VM once the cluster is up.

---

## Installing Proxmox 9.1

With hardware in hand, the next step was getting Proxmox VE 9.1 installed on all three nodes. I used the standard ISO installer, booted via USB on each machine in sequence.

### The Partition Decision

This is the part where I made a deliberate choice that deviates from a default install, and it's worth explaining why.

By default, the Proxmox installer will consume the entire disk. On a 500 GB drive (~476 GB shown usable), that means creating an LVM volume group (`pve`) that spans nearly all 500 GB, with logical volumes for the OS root, swap, and a data pool for VM disk images (`local-lvm`). That's a reasonable default if local storage is all you'll ever use, but it makes carving out space for a Ceph OSD later more painful than it needs to be.

Instead, I used the installer's advanced disk options to set `hdsize` to **76 GB**. This tells the installer to only partition and use the first 76 GB of the drive, leaving the remaining ~400 GB as unallocated free space on the disk. That free space will be partitioned separately when I configure Ceph OSDs, keeping the Proxmox LVM volume group clean and isolated from the Ceph stack.

The Proxmox installer's `hdsize` setting is found in the advanced options on the disk configuration screen — it's not prominently advertised, but it's exactly the right tool for this.

After setting `hdsize=76`, the installer creates:
- EFI partition: ~512 MB
- Boot partition: ~1 GB
- LVM VG `pve`: ~74 GB
  - `root` LV: ~26 GB (OS)
  - `swap` LV: ~8 GB
  - `data` LV (thin pool): remaining ~40 GB

The 40 GB local thin pool is enough for ISO images and a few lightweight containers. Everything else will land on Ceph once that's configured.

```bash
# Verify the partition layout after install
lsblk

# Confirm free space on the drive is unallocated
parted /dev/nvme0n1 print free
```

The second M.2 drive (512 GB) was left completely unpartitioned at this stage. It'll be handed to Ceph as a whole-disk OSD.

### Post-Install Basics

After the three-node install, a few immediate housekeeping steps before touching anything else:

```bash
# Disable the enterprise repo (requires paid subscription) and enable no-subscription
sed -i 's/^deb/# deb/' /etc/apt/sources.list.d/pve-enterprise.list
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-sub.list

# Update
apt update && apt full-upgrade -y

# Install useful tools
apt install -y htop iotop net-tools ethtool vim
```

Repeat on all three nodes. I keep them named simply: `node01`, `node02`, `node03` — with matching hostnames and sequential IPs on the management network. Simple naming goes a long way in a lab where you're SSHing into things constantly.

---

## Where Things Stand

At the end of this phase, here's what exists:

```
node01 (192.168.x.11) — Proxmox 9.1, 76 GB partitioned, 424 GB free, 512 GB free
node02 (192.168.x.12) — Proxmox 9.1, 76 GB partitioned, 424 GB free, 512 GB free
node03 (192.168.x.13) — Proxmox 9.1, 76 GB partitioned, 424 GB free, 512 GB free
```

Three nodes, all reachable on the management network, none yet clustered. Storage sits idle. The switch is physically connected but VLANs are not configured. The NIC bonding and bridge configuration that Proxmox needs for a multi-VLAN network is a blank page.

That's the next post.

---

## What's Next

Post 3 covers everything about the network and storage configuration:

- VLAN design on the Ubiquiti switch (management, storage, VM traffic)
- Proxmox bridge and VLAN interface configuration on both NICs
- Ceph cluster initialization, OSD configuration, and pool creation
- Standing up a virtual pfSense router on the VM traffic VLAN

The networking setup is where most of the complexity lives — and where most of my initial config attempts failed before I got it right. That made for good notes.

---

*This is post 2 of an ongoing series. [Start from the beginning](https://mattvirtually-source.github.io/homelab-blog/post-01-the-vision/) or [view all posts in this series](https://mattvirtually-source.github.io/homelab-blog/).*
