---
title: "The Vision: Why I'm Building a Home Lab"
date: 2026-04-09
draft: false
series: ["Series 1 — Building the Foundation"]
tags: ["homelab", "intro", "virtualization", "kubernetes"]
summary: "The what, the why, and the roadmap for a home lab built to learn virtualization and container orchestration from the ground up."
showToc: true
TocOpen: false
---

I've worked with enterprise virtualization for a while now. VMware, Hyper-V, XenServer — the usual suspects. I know how to operate them. I can provision a VM, configure a network, troubleshoot a storage issue. But there's a difference between *operating* a platform and *understanding* it deeply, and for the platforms I haven't touched yet, I don't have that depth.

That gap bothered me enough to do something about it. So I'm building a home lab.

This blog is the paper trail.

---

## What Is a Home Lab?

If you're already building one, skip this. If you landed here from a search: a home lab is a personal, self-managed infrastructure environment — usually sitting in a spare room, a basement, or a closet — that you run for the purpose of learning, experimentation, and breaking things without consequence. No change control tickets. No production impact. No explaining to anyone why the dev cluster is down at 2am.

The hardware is usually repurposed enterprise gear (cheap on eBay once it's a generation or two old), and the whole point is to run real workloads, real platforms, and real configurations — not simulations.

---

## What I'm Building

The plan has three phases.

### Phase 1: The Foundation

Before anything interesting happens, there's hardware to rack and cable, a network to design, and a management plane to set up. This is the boring necessary work, and I'm going to document it anyway, because "boring necessary work" is exactly what breaks people's labs six months in when they skip it.

Phase 1 covers:
- Hardware selection and physical setup
- Networking: VLANs, a proper management network, OOB access (IPMI/iDRAC)
- A repeatable base you can rebuild from if something goes catastrophically wrong

### Phase 2: Hypervisors

I've spent time with VMware ESXi/vSphere, Microsoft Hyper-V, and Citrix XenServer. Each one has shaped how I think about virtualization — how VMs are managed, how storage gets abstracted, how networking is modeled.

For this lab, I'm going in a different direction. The three platforms I'll be evaluating are:

**Proxmox VE** — A Debian-based open source hypervisor that's become the go-to choice for home labs and small businesses. Built on KVM and LXC, with a polished web UI, ZFS storage integration, and a clustering model that's surprisingly capable. I want to understand it well enough to have a real opinion about it.

**KVM + QEMU on Ubuntu** — The manual path. Instead of a pre-packaged hypervisor distro, I'll set up KVM directly on Ubuntu, configure libvirt, and manage everything through virsh and virt-manager. The goal is to understand what Proxmox is actually built on top of — removing the abstraction layer.

**XCP-ng** — The open source fork of Citrix XenServer. Free, community-maintained, and backed by Vates (the team behind Xen Orchestra). If you've used XenServer, this is the continuation of that lineage without the licensing headache.

Each platform gets its own posts: installation, initial configuration, storage setup, networking, and anything interesting that comes up along the way. After running all three for a few weeks, I'll do a shootout post comparing them across the dimensions that actually matter for a real deployment.

### Phase 3: Kubernetes

This is where I'm spending the most time and the most money. Container orchestration at scale is where enterprise infrastructure is going, and I want hands-on experience with the platforms that run it in production.

The three I'll be evaluating:

**OpenShift** — Red Hat's Kubernetes distribution. The enterprise standard for a lot of large organizations. It adds a significant layer on top of upstream Kubernetes: an Operator framework, a build pipeline, a web console with real UX investment, and a security model that is — depending on who you ask — either admirably opinionated or aggressively annoying. I expect it to be both.

**Rancher** — Rancher Labs' (now SUSE's) Kubernetes management platform. The pitch is multi-cluster management: you can spin up RKE2 clusters, import existing clusters, and manage them all from one place. More operator-friendly than OpenShift, less opinionated. I'm particularly interested in Fleet (GitOps at scale) and how it compares to OpenShift's approach.

**VMware VCF with Tanzu** — This one is conditional. VMware Cloud Foundation is the full enterprise stack: vSphere, vSAN, NSX-T, and Aria (formerly vRealize) bundled together with Tanzu for Kubernetes. It's expensive and complex, and the NFR (not-for-resale) license path is the only way to run it in a home lab legally. If I can get those licenses, this gets its own series. If not, I'll document the attempt and move on.

---

## Why Document It?

A few reasons.

Writing forces clarity. If I can't explain what I'm doing well enough to write it down, I probably don't understand it well enough to trust it. The act of writing is part of the learning, not just a record of it.

I've benefited enormously from other people's lab write-ups. The person who documented their Proxmox cluster setup three years ago saved me hours. The blog post about XCP-ng SDN behavior that showed up in a search at exactly the right moment — same. I want to add to that body of work.

And honestly, having a public record of what I'm learning keeps me accountable to finishing it. Nothing like knowing someone might actually read this to make sure I actually finish the Kubernetes series instead of stalling out after the first install.

---

## What's Next

The hardware is either racked or arriving shortly, depending on when you're reading this. Next post covers the physical setup: what I'm working with, how it's configured, and the decisions I made on hardware and networking before a single VM was provisioned.

If you've done this before, [reach out](#) — I'm interested in what you'd do differently and what I'm probably about to get wrong.

---

*This is post 1 of an ongoing series. Start here or jump to the [series index](#).*
