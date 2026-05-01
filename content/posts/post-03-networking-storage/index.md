---
title: "The Network and Storage Stack: VLANs, Proxmox SDN, and Ceph"
date: 2026-04-22
draft: false
weight: 3
series: ["Series 1 — Building the Foundation"]
tags: ["homelab", "proxmox", "networking", "ceph", "sdn", "vlans", "ubiquiti"]
summary: "Configuring the Ubiquiti switch, Proxmox Linux bridges, and SDN for VM networking, then standing up a three-node Ceph cluster with two OSDs per node."
showToc: true
TocOpen: false
---

With three Proxmox nodes up and reachable on the home network, the next step was to stop treating the lab like three isolated boxes and turn it into something that actually behaves like a cluster. That means a deliberate network design and shared storage. This setup will mirror many production environments and allow for live migration and high availability.

This post covers everything from switch VLAN configuration through Ceph OSD creation. It's a long one, but these two pieces are foundational to everything that comes after.

---

## The Physical Layout

Before any software configuration, the cabling follows a simple pattern. Each Nucbox has two NICs, plugged into the Ubiquiti switch in sequential order:

| Node | NIC | Switch Port |
|------|-----|-------------|
| node01 | nic0 | Port 1 |
| node01 | nic1 | Port 2 |
| node02 | nic0 | Port 3 |
| node02 | nic1 | Port 4 |
| node03 | nic0 | Port 5 |
| node03 | nic1 | Port 6 |

The uplink to the home router occupies the 10 GbE SFP+ port, leaving all eight 2.5 GbE ports available for the cluster.

This also leaves ports 7 and 8 available for future expansion, likely a NAS I plan to build.

---

## Switch Configuration

All VLAN configuration is handled through the Ubiquiti UniFi Network Controller.

### VLAN Design

Three networks, one purpose each:

| VLAN | Name | Subnet | Purpose |
|------|------|--------|---------|
| 1 | Management / Home | 192.168.86.0/24 | Proxmox management interfaces, home network access |
| 10 | VM Traffic | 10.10.86.0/24 | Guest VM network, handled by Proxmox SDN |
| 100 | Storage | 172.20.86.0/24 | Ceph replication and client traffic, isolated to nic1 |

### Port Profiles

**nic0 ports (1, 3, 5): Trunk**

These ports carry all VLANs tagged, with VLAN 1 as the native (untagged) VLAN. The intention is to handle additional VLAN definitions entirely inside Proxmox rather than needing to return to the switch every time a new network is needed. The switch just passes everything through. Proxmox SDN takes it from there.

**nic1 ports (2, 4, 6): VLAN 100 Access**

These ports are set with VLAN 100 as the native VLAN and all other VLANs disallowed. They behave as standard access ports. The storage network is completely isolated at the switch level. No VM traffic, no management traffic, and nothing from the home network can reach the storage interfaces without a deliberate routing decision.

Keeping storage on its own physical NIC and its own switch ports eliminates the possibility of Ceph replication traffic competing with VM workloads for bandwidth, even under heavy load on nic0.

---

## Proxmox Network Configuration

Proxmox represents network interfaces as Linux bridges. VMs and containers attach to these bridges rather than directly to physical NICs. Each bridge can carry one VLAN or many, depending on its configuration.

### vmbr0: Management Bridge (nic0)

This bridge was created automatically during the Proxmox installer and attached to nic0. The management IP for each node lives here. No changes were needed beyond confirming the configuration was correct after install.

```
# /etc/network/interfaces (node01 example)
auto lo
iface lo inet loopback

iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.86.11/24
    gateway 192.168.86.1
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0
```

Node IPs on the management network follow a consistent pattern, with the final octet matching the node number: `192.168.86.11`, `192.168.86.12`, `192.168.86.13`.

### vmbr1: Storage Bridge (nic1)

The storage bridge was created manually through the Proxmox web UI after install. This bridge attaches to nic1, which connects to the VLAN 100 access ports on the switch. No VLAN tagging is needed on this bridge since the switch is already handling that at the port level.

```
iface nic1 inet manual

auto vmbr1
iface vmbr1 inet static
    address 172.20.86.11/24
    bridge-ports nic1
    bridge-stp off
    bridge-fd 0
```

No gateway is set on vmbr1. Storage traffic stays on the 172.20.86.0/24 network and never needs to route anywhere. The final octet again matches the node number: `172.20.86.11`, `172.20.86.12`, `172.20.86.13`.

---

## VM Networking: Proxmox SDN

This is where the configuration gets interesting.

The obvious approach for VM networking in Proxmox is to create another Linux bridge, tag a VLAN on it, and assign VMs to that bridge. That works, but it requires touching the network config on every node individually, and there's no centralized way to manage subnets, DHCP, or routing across the cluster.

Proxmox SDN is a better answer. It's a cluster-aware networking layer built into Proxmox that handles VLANs, IP management, DHCP, and NAT from a single interface, then pushes the config to every node in the cluster automatically. The comparison that clicked for me: it's like distributed vSwitch had a baby with a lite version of NSX, but it's integrated directly into the hypervisor and significantly simpler to operate.

For anyone coming from VMware, a quick mapping:

| VMware Concept | Proxmox SDN Equivalent |
|----------------|----------------------|
| Distributed vSwitch | Zone |
| dVs NIC Assignment | Underlying vmbr interface |
| Distributed Port Group | VNet |

The reason I compare it to NSX is that inter-VLAN routing, NAT, and firewalling/segmentation are all handled directly within Proxmox SDN, without needing a separate appliance.

### Creating the Zone

Navigate to **Datacenter → SDN → Zones → Add → VLAN**.

Configuration:
- **ID:** VM
- **Type:** VLAN
- **Bridge:** vmbr0
- **Nodes:** node01, node02, node03

The VLAN zone type creates VNets as tagged VLANs on top of the specified bridge. Since vmbr0 is a trunk port receiving all VLANs from the switch, any VLAN tag defined in SDN will work without additional switch changes.

### Creating the VNet

Navigate to **Datacenter → SDN → VNets → Add**.

Configuration:
- **Name:** VM10
- **Zone:** VM
- **Tag:** 10

This creates a VLAN 10 network available across all three nodes. VMs assigned to VM10 will communicate over VLAN 10 on vmbr0.

### Creating the Subnet

Under VM10, add a subnet:

- **Subnet:** 10.10.86.0/24
- **Gateway:** 10.10.86.1
- **DHCP:** Enabled
- **DHCP Range:** 10.10.86.30 to 10.10.86.200
- **SNAT:** Enabled

SNAT (Source NAT) is what gives VMs on the 10.10.86.0/24 network outbound internet access without needing a dedicated router VM. Proxmox translates outbound traffic from the VM network to the management IP of whichever node the VM is running on, then routes it through the home network to the internet. It's simple, functional, and appropriate for lab use.

A dedicated firewall VM is a future addition for more granular policy and routing between networks. For now, SNAT covers the requirement.

### Applying the Config

SDN configuration is staged before it takes effect. After creating the zone, VNet, and subnet, click **Apply** in the SDN panel. Proxmox pushes the configuration to all nodes in the cluster. No manual network changes needed per node.

> Coming in a later post: the entire SDN configuration can be defined in Terraform using the Proxmox provider. Codifying the current setup will be the first infrastructure-as-code experiment in this lab.

---

## Storage: Ceph

With networking in place, the next piece is shared storage. Ceph is a distributed storage system that pools disks across multiple nodes and presents that capacity as a single storage cluster. In a Proxmox context, it's the standard choice for a hyperconverged setup: the same nodes that run VMs also provide storage, and Proxmox manages Ceph directly from the web UI.

Each node contributes two OSDs (Object Storage Daemons) to the cluster, for six total:

| Node | OSD | Source |
|------|-----|--------|
| node01 | osd.0 | 400 GB partition on 476 GB usable boot drive |
| node01 | osd.1 | Full 512 GB second M.2 drive |
| node02 | osd.2 | 400 GB partition on 476 GB usable boot drive |
| node02 | osd.3 | Full 512 GB second M.2 drive |
| node03 | osd.4 | 400 GB partition on 476 GB usable boot drive |
| node03 | osd.5 | Full 512 GB second M.2 drive |

With three nodes and a default replication factor of 3, every piece of data is written to one OSD on each node. Usable capacity is roughly one-third of raw capacity. Six OSDs averaging roughly 450 GB each gives about 2.7 TB raw, and approximately 900 GB usable. That's more than sufficient for a lab running VMs and containers.

### Installing Ceph

Proxmox includes Ceph installation directly in the web UI. Navigate to any node, click **Ceph**, and follow the installation wizard. When prompted for a repository, select the no-subscription repository to match the package source already configured for Proxmox itself.

Run the installation on each node.

### Monitors and Managers

A Ceph cluster requires at least one monitor (MON) and one manager (MGR). For a three-node cluster, placing one of each on every node is the correct configuration: it ensures the cluster can maintain quorum and continue operating if a single node goes offline.

Add a monitor and manager for each node through **Ceph → Monitor → Add** and **Ceph → Monitor → Add Manager**, selecting each node in turn.

### Partitioning the Boot Drive

This is the step that required the most care. The 500 GB boot drive reports 476 GB usable, and was installed with Proxmox using only 76 GB, leaving roughly 400 GB of unallocated space. That space needs to be a partition before Ceph can use it as an OSD.

First, identify the correct disk. On the Nucbox M5 Ultra, the boot drive is the NVMe device:

```bash
lsblk
```

Confirm which device is the boot disk. It will show existing partitions for the EFI, boot, and LVM volumes. On these nodes it is `nvme0n1`. The second M.2 drive is `nvme1n1` and should show as completely unpartitioned.

Then create the new partition on the boot drive:

```bash
fdisk /dev/nvme0n1
```

Inside fdisk:
- `p` to print the current partition table and confirm free space is available after the existing partitions
- `n` to create a new partition
- Accept the defaults for partition number and start sector
- Accept the default end sector to use all remaining space
- `w` to write and exit

Verify the new partition is visible:

```bash
lsblk /dev/nvme0n1
```

The new partition (`nvme0n1p4` or similar, depending on how many partitions already exist) should show the remaining capacity. Repeat on node02 and node03.

### Adding OSDs

With the partition created and the second drive unpartitioned, add OSDs through the web UI. Navigate to **Ceph → OSD → Create OSD** on each node.

For the boot drive partition: select the new partition device (e.g., `/dev/nvme0n1p4`).

For the second drive: select the full device (`/dev/nvme1n1`). Ceph will partition and format it automatically.

Add both OSDs per node, all three nodes. Once all six OSDs are added and the cluster finishes its initial peering, the Ceph dashboard should show the cluster in a healthy state with the full pool capacity available.

---

## Where Things Stand

At the end of this phase, the cluster looks like this:

```
node01 (192.168.86.11) - Proxmox 9.1
  Management: 192.168.86.11 (vmbr0 / nic0)
  Storage:    172.20.86.11 (vmbr1 / nic1)
  OSDs:       nvme0n1p4 (~400 GB) + nvme1n1 (512 GB)

node02 (192.168.86.12) - Proxmox 9.1
  Management: 192.168.86.12 (vmbr0 / nic0)
  Storage:    172.20.86.12 (vmbr1 / nic1)
  OSDs:       nvme0n1p4 (~400 GB) + nvme1n1 (512 GB)

node03 (192.168.86.13) - Proxmox 9.1
  Management: 192.168.86.13 (vmbr0 / nic0)
  Storage:    172.20.86.13 (vmbr1 / nic1)
  OSDs:       nvme0n1p4 (~400 GB) + nvme1n1 (512 GB)

Ceph: 6 OSDs, ~2.7 TB raw, ~900 GB usable (3x replication)
SDN:  Zone VM / VNet VM10 / VLAN 10 / 10.10.86.0/24
```

Three nodes, fully clustered, shared storage healthy, VM networking available across all nodes. The foundation is done.

---

## What's Next

The next series shifts focus from infrastructure to automation. Before deploying anything manually, I want to codify what I have. The plan is to use Terraform with the Proxmox provider to define the SDN configuration and VM templates as code. If I ever need to rebuild this cluster, it should be a repeatable process rather than a set of notes I have to follow by hand.

---

*This is post 3 of an ongoing series. [Start from the beginning](/posts/post-01-the-vision/) or [view all posts in this series](/posts/).*
