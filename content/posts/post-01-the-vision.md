---
title: "The Vision: Why I'm Building a Home Lab"
date: 2026-04-09
draft: false
weight: 1
series: ["Series 1 — Building the Foundation"]
tags: ["homelab", "intro", "virtualization", "kubernetes"]
summary: "The what, the why, and the roadmap for a home lab built to learn virtualization and container orchestration from the ground up."
showToc: true
TocOpen: false
---

I've worked with enterprise virtualization my entire IT career. Starting with Citrix XenServer in the small company I started in, later progressing to VMware, and even a stint with Hyper-V as a professional services engineer. I even know how to manage these environments adhering to modern DevOps practices (IaC). 

After progressing out of operational IT in 2019, leaving my Enterprise Engineer role at a large law firm for a Technical Account Manager role at VMware, there were just technologies that I did not get a chance to touch. A lot of the DevOps I had practice with was scripting (PowerShell, Python) based leveraging Azure DevOps to automate the process. I didn't get to experience technologies like Terraform, or Ansible. And as more applications make the shift from VMs to Containers I have yet to gain experience on Kubernetes.

As an Enterprise Solution Engineer at SHI I make it a point to understand these technologies and their use at a level that I'm beneficial to my customers in more technical conversations. I know their place in the tech stack. I have a high level understanding of how these technologies are leveraged. But I've always been a hands on learner, and my experience in these areas is lacking.

That gap bothered me enough to do something about it. So I'm building a home lab.

This blog is the paper trail.

---

## What Is a Home Lab?

If you're already building one, skip this. If you landed here from a search: a home lab is a personal, self-managed infrastructure environment. Mine will live in my home office, directly behind where I work every day. It's used for the purpose of learning, experimentation, and breaking things without consequence. No change control tickets. No production impact. No explaining to anyone why the cluster is down at 2am.

The hardware is usually repurposed enterprise gear (cheap on eBay once it's a generation or two old), and the whole point is to run real workloads, real platforms, and real configurations not simulations.

---

## What I'm Building

The plan has three phases.

### Phase 1: The Foundation

Before anything interesting happens, there's hardware to rack and cable, a network to design, and a management plane to set up. This is the boring necessary work, and I'm going to document it anyway, because "boring necessary work" is exactly what breaks people's labs six months in when they skip it.

Phase 1 covers:
- Hardware selection and physical setup
- Networking: VLANs, a proper management network, storage networking, dns, virtual routing/firewalling
- A repeatable base I can rebuild from if something goes catastrophically wrong

### Phase 2: Hypervisor

I've spent time with VMware ESXi/vSphere, Microsoft Hyper-V, and Citrix XenServer. The obvious next step would be to lean on one of those — I have VMUG licensing for VMware and XenServer experience that maps directly to XCP-ng. But the point of this lab is to learn things I haven't touched, not to get comfortable with things I already know.

Proxmox VE is the one major hypervisor I haven't run. It's KVM under the hood, which matters as I move into automation and container orchestration — Terraform has a solid Proxmox provider, Ansible integrates cleanly, and every Kubernetes platform I plan to evaluate has been documented running on Proxmox extensively. It gives me a stable, well-documented foundation without locking me into a commercial licensing model.

This isn't a hypervisor evaluation series. Proxmox is the platform — everything else gets built on top of it.


### Phase 3: Automation, IaC, and Kubernetes

This is where I'm spending the most time. Playing with automation platforms like Ansible and Terraform is common practice for any hybrid cloud, and container orchestration at scale is where enterprise infrastructure is going, and I want hands-on experience with the platforms that run it in production.

I'll be evaluating infrastructure automation platforms, various opinionated flavors (and management layers) of Kubernetes, such as Rancher, and OpenShift. I'll likely dip my toes into things like Prometheus + Grafana for monitoring. 

The longer goal behind all this goes beyond the lab itself. I have been playing with AI tools like Claude Code and Cursor to build small pet projects for fun, my goal is to take the knowledge I learn using this Lab and expand those pet projects to some fun coding projects that I can use this new environment to deploy, test, and then break. Over and over! This will also help me expand my knowledge in modern development practices.

---

## Why Document It?

A few reasons.

Writing forces clarity. If I can't explain what I'm doing well enough to write it down, I probably don't understand it well enough to trust it. The act of writing is part of the learning, not just a record of it.

I've benefited enormously from other people's lab write-ups. The various people who documented their Proxmox lab builds have already helped me a ton. I want to add to that body of work, if for no other reason than to prove an old dog can still learn new tricks.

And honestly, having a public record of what I'm learning keeps me accountable to finishing it. Nothing like knowing someone might actually read this to make sure I actually finish the Kubernetes series instead of stalling out after the first install.

---

## What's Next

The hardware is racked and the switch is on the way, next post covers the physical setup: what I'm working with, how it's configured, and the decisions I made on hardware and networking before a single VM was provisioned.

If you've done this before, [reach out](mailto:matt.virtually@gmail.com) — I'm interested in what you'd do differently and what I'm probably about to get wrong.

---

*This is post 1 of an ongoing series. Start here or jump to the [series index](https://mattvirtually-source.github.io/homelab-blog/posts/).*
