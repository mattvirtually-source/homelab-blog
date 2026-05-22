---
title: "Network Changes, Cloud-Init Templates, and the First Terraform Config"
date: 2026-05-21
draft: false
weight: 5
series: ["Series 2 — Automation and IaC"]
tags: ["homelab", "opnsense", "terraform", "proxmox", "cloud-init", "iac", "kubernetes"]
summary: "Replacing Proxmox SDN SNAT with a proper OPNsense firewall, building a reusable Ubuntu cloud-init template, and writing the first Terraform configuration to provision Kubernetes nodes."
showToc: true
TocOpen: false
---

In the last post I had the lab workstation set up with Git, a GitHub strategy in place, and Terraform installed. The homelab-terraform repo existed but was essentially empty. This post covers what happened next: deploying a real firewall, building a cloud-init VM template, and writing the Terraform configuration that will be used to provision Kubernetes nodes.

There is a lot of ground in this one.

---

## Replacing SNAT with OPNsense

When I set up the VM10 network in the previous series, I used Proxmox SDN's built-in SNAT to give VMs internet access. SNAT works, but it is a blunt instrument. There is no visibility into what traffic is flowing, no way to write granular firewall rules between networks, and no clean path to more advanced routing scenarios as the lab grows.

For a lab that is heading toward Kubernetes, where pod-to-pod traffic, ingress controllers, and network policies all matter, the right answer is a proper firewall and router. I deployed OPNsense.

### Deploying the OPNsense VM

OPNsense is deployed as a VM on node01 with two network interfaces:

| Interface | Connected To | IP Address |
|-----------|-------------|------------|
| WAN (vtnet0) | vmbr0 (management network) | DHCP from home router |
| LAN (vtnet1) | VM10 SDN VNet | 10.10.86.1/24 |

The WAN interface sits on the same 192.168.86.0/24 network as the Proxmox management interfaces and the home router. It gets internet access through the home router, just like everything else on that network. The LAN interface becomes the default gateway for anything on the VM10 (10.10.86.0/24) network.

This is the same logical topology as a typical edge firewall: WAN faces upstream, LAN faces the environment it protects.

### The Initial Access Problem

OPNsense's default firewall behavior presented the first obstacle. By default, the management web interface is reachable on the LAN side but locked down on the WAN side. Since no VMs existed on the LAN yet, and I needed to configure OPNsense from my laptop on the management network, I temporarily disabled the firewall to get initial access to the console.

Once in, I created a permanent firewall rule to allow HTTPS (port 443) inbound from the management network to the OPNsense WAN interface IP. That made the management console reachable without disabling the firewall entirely.

Additional rules configured after initial setup:

| Rule | Direction | From | To | Purpose |
|------|-----------|------|----|---------|
| Allow HTTPS | WAN in | 192.168.86.0/24 | WAN IP | Management console access |
| Allow SSH | WAN in | 192.168.86.0/24 | 10.10.86.0/24 | SSH into VMs from management network |
| Allow ICMP | WAN in | 192.168.86.0/24 | 10.10.86.0/24 | Ping for troubleshooting |

With OPNsense handling routing and NAT, I disabled SNAT on the Proxmox SDN subnet. OPNsense is now the sole path between the VM network and the outside world.

---

## Preparing Proxmox for Terraform

Terraform communicates with Proxmox through the Proxmox REST API. Before writing a single resource block, the Proxmox side needs two things: a dedicated service account and an API token scoped to what Terraform actually needs.

### Creating the Terraform User and Role

The right approach here is least privilege. Terraform does not need root access to Proxmox. It needs enough permissions to create, configure, clone, and destroy VMs. Nothing more.

I created a dedicated user (`terraform@pve`) in Proxmox's built-in PVE authentication realm, along with a custom role (`TerraformHomelab`) that holds only the specific privileges required.

A script to automate this lives in the repo at `scripts/setup-proxmox-terraform-user.sh`. Run it as root on any Proxmox node:

```bash
bash scripts/setup-proxmox-terraform-user.sh
```

The script creates the user, defines the role with a scoped privilege set, assigns the role at the datacenter level, generates an API token, and prints the token value. The token is shown once. Copy it immediately.

The privilege set covers what Terraform needs for VM lifecycle management:

```
Datastore.Allocate, Datastore.AllocateSpace,
VM.Allocate, VM.Audit, VM.Clone,
VM.Config.Cloudinit, VM.Config.CPU, VM.Config.Disk,
VM.Config.HWType, VM.Config.Memory, VM.Config.Network,
VM.Config.Options, VM.GuestAgent.Audit, VM.PowerMgmt
```

### Setting the API Token

The Proxmox Terraform provider reads the API token from an environment variable. Set it before running any Terraform commands:

```bash
export PROXMOX_VE_API_TOKEN='terraform@pve!terraform=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
```

The token format is `user@realm!tokenid=secret`. This stays out of any config file and out of Git entirely.

---

## Building the Cloud-Init Template

Terraform can provision VMs, but it needs something to clone from. Rather than installing Ubuntu manually every time a node is needed, the approach is to create a single cloud-init-enabled template and let Terraform clone it with per-VM customization passed at provision time.

Cloud-init is a standard for configuring Linux instances at first boot. You pass in hostname, user accounts, SSH keys, network configuration, and optional packages or scripts. The image boots, cloud-init runs, and the VM is configured and ready. No interactive installer, no manual steps per node.

### What the Template Script Does

The template is built from Ubuntu 24.04's official cloud image, which is a minimal disk image pre-configured for cloud-init use. The script I wrote to create the template runs on node01:

```bash
# Variables
TEMPLATE_ID=9000
TEMPLATE_NAME=ubuntu-2404-cloudinit-template
STORAGE=vm-pool
BRIDGE=vm10
UBUNTU_RELEASE=24.04

cd /var/lib/vz/template/iso

# Download the Ubuntu cloud image
wget -O ubuntu-cloud.img \
  "https://cloud-images.ubuntu.com/releases/${UBUNTU_RELEASE}/release/ubuntu-${UBUNTU_RELEASE}-server-cloudimg-amd64.img"

# Create an empty VM shell
qm create ${TEMPLATE_ID} \
  --name "${TEMPLATE_NAME}" \
  --memory 2048 \
  --cores 2 \
  --net0 virtio,bridge=${BRIDGE} \
  --agent enabled=1 \
  --ostype l26 \
  --serial0 socket \
  --vga serial0

# Import the cloud image as a disk
qm importdisk ${TEMPLATE_ID} ubuntu-cloud.img ${STORAGE}

# Attach the disk and cloud-init drive
qm set ${TEMPLATE_ID} \
  --scsihw virtio-scsi-pci \
  --scsi0 ${STORAGE}:vm-${TEMPLATE_ID}-disk-0 \
  --ide2 ${STORAGE}:cloudinit \
  --boot order=scsi0 \
  --ipconfig0 ip=dhcp

# Convert to template
qm template ${TEMPLATE_ID}
```

A few things worth noting in that script:

**`--serial0 socket` and `--vga serial0`**: Cloud images often have no graphical console configured. Setting the serial device and redirecting VGA output to it gives you a working console in the Proxmox web UI and through `qm terminal`. Without this, a non-booting VM is very difficult to troubleshoot.

**`--agent enabled=1`**: Enables the QEMU guest agent. This is what allows Proxmox (and Terraform) to read the VM's IP address after boot and confirm the agent is running before considering provisioning complete.

**Storage target `vm-pool`**: The disk is imported into the Ceph pool, making the template available to all three nodes. Terraform can provision new VMs on any node using this template.

**`qm template`**: This final step converts the VM into a template. A template cannot be started directly, only cloned. It locks the base image in place.

Template ID 9000 is a convention, not a requirement. I use IDs above 9000 for templates and keep VM IDs below 900 for provisioned machines.

---

## The Terraform Configuration

With the Proxmox side ready, the Terraform configuration is straightforward. The full source is in the [homelab-terraform repo](https://github.com/mattvirtually-source/homelab-terraform). Here is a walkthrough of each file and the decisions behind it.

### Provider Choice: bpg/proxmox

There are two actively maintained Terraform providers for Proxmox. I went with the `bpg/proxmox` provider rather than the original `Telmate/proxmox` provider. The bpg provider has more complete coverage of the Proxmox API, better support for cloud-init configuration, and more active maintenance. If you are starting a new Proxmox Terraform project, it is the better starting point.

### versions.tf

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.73"
    }
  }
}
```

Pinning the provider version to `~> 0.73` means Terraform will accept any `0.73.x` patch release but won't automatically upgrade to `0.74.x` or beyond. Provider APIs can introduce breaking changes, and infrastructure code should not break silently because of an upstream update.

### providers.tf

```hcl
provider "proxmox" {
  endpoint = var.proxmox_endpoint
  insecure = var.proxmox_insecure

  # Set PROXMOX_VE_API_TOKEN in the environment (recommended)
  # Format: user@realm!tokenid=secret
}
```

The endpoint and TLS verification flag come from variables. The API token comes from the environment variable set earlier. No credentials in any file that goes near Git.

`insecure = true` is necessary for a homelab running a self-signed certificate on the Proxmox web interface. In a production environment, this would point to a proper CA-signed certificate.

### variables.tf

The variables file defines the inputs with types and descriptions. Rather than listing everything here, the key inputs are:

| Variable | Purpose |
|----------|---------|
| `proxmox_endpoint` | API URL, e.g. `https://192.168.86.11:8006/` |
| `proxmox_node` | Which node provisions the VMs |
| `template_vm_id` | The VM ID of the cloud-init template (9000) |
| `disk_datastore` | Storage ID for VM disks (`vm-pool` for Ceph) |
| `network_bridge` | Host bridge for VM NICs |
| `cloud_init_user` | Linux user created by cloud-init |
| `ssh_public_key` | Public SSH key injected via cloud-init |

All actual values live in `terraform.tfvars`, which is excluded from Git via `.gitignore`. The repo includes a `terraform.tfvars.example` to document what the file should look like.

### locals.tf: Defining the Nodes

Rather than writing a separate resource block for each VM, all node definitions live in a single `locals` map:

```hcl
locals {
  k8s_nodes = {
    control-1 = {
      vm_id      = 201
      cores      = 2
      memory_mb  = 4096
      ip_mode    = "dhcp"
      ipv4_cidr  = null
      dns_domain = null
    }
    worker-1 = {
      vm_id      = 211
      cores      = 4
      memory_mb  = 8192
      ip_mode    = "dhcp"
      ipv4_cidr  = null
      dns_domain = null
    }
  }
}
```

Adding a new node to the cluster means adding an entry to this map. The resource block handles the rest. This is the kind of pattern that makes infrastructure code maintainable as it grows.

### vms.tf: The Resource Definition

```hcl
resource "proxmox_virtual_environment_vm" "k8s_node" {
  for_each = local.k8s_nodes

  name      = each.key
  node_name = var.proxmox_node
  vm_id     = each.value.vm_id
  tags      = ["homelab", "terraform", "k8s"]

  cpu {
    cores = each.value.cores
    type  = "x86-64-v2-AES"
  }

  memory {
    dedicated = each.value.memory_mb
  }

  clone {
    vm_id        = var.template_vm_id
    datastore_id = var.disk_datastore
    full         = true
  }

  network_device {
    bridge = var.network_bridge
  }

  initialization {
    datastore_id = var.disk_datastore

    ip_config {
      ipv4 {
        address = each.value.ip_mode == "dhcp" ? "dhcp" : each.value.ipv4_cidr
        gateway = each.value.ip_mode == "dhcp" ? null : var.ipv4_gateway
      }
    }

    user_account {
      username = var.cloud_init_user
      keys     = [trimspace(var.ssh_public_key)]
    }
  }
}
```

`for_each = local.k8s_nodes` iterates over the map defined in locals.tf and creates one VM per entry. Each VM is a full clone of the template with its own disk, not a linked clone that shares the base image. Full clones are slower to provision but independent, which is what you want for nodes that will be rebooted, reconfigured, and eventually destroyed and rebuilt.

The `initialization` block is where cloud-init configuration happens. IP mode is determined per node. If `ip_mode = "dhcp"`, the VM requests an address from OPNsense's DHCP server on the VM10 network. If static, `ipv4_cidr` and the gateway variable are used instead. The SSH public key is injected here, so the first SSH connection to a newly provisioned node works immediately.

---

## What's Next

The configuration is written and reviewed. The next step is the first real `terraform apply`, which will provision the control plane and worker nodes that the Kubernetes cluster will run on.

That applies a pattern that will repeat throughout this lab: write the code, run it, see what breaks. Then fix it and run it again.

---

*This is post 5 of an ongoing series. [Start from the beginning](/posts/post-01-the-vision/) or [view all posts in this series](/posts/).*
