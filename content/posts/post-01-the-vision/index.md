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

## The Why?

I've worked with virtualization my entire IT career. At my first IT job, shoutout to Western Title Company, I leveraged Citrix XenServer on a few servers that had previously hosted everything on bare metal and a Netgear NAS with NFS for storage. And yes, for those who didn't know, Netgear made a NAS... and I was crazy (or dumb) enough to leverage one for the backend of a company's entire virtualization stack!

When Western Title was acquired by a multistate holding company and I became that company's Infrastructure Manager I leveraged XenServer to virtualize the majority of their server stack, later progressing to VMware as the environment grew and we needed the capabilities and support of a truly enterprise solution. There's a story from this time in my career that deserves its own post: the trials and tribulations of building a DR datacenter with application-based replication over a T3.

As a Professional Services Engineer I ran VMware best practice reviews and services work to bring companies' environments into alignment with those reviews. For smaller customers I would leverage Hyper-V to virtualize without the additional cost for VMware. It's in this role that I started deploying infrastructure as code, primarily scripting-based automation, while still building environments friendly to small IT and managed services teams that ran on a click-ops model.

In my Enterprise Engineer role at a large law firm I managed the hybrid cloud consisting of VMware in the datacenter and Azure. I further embraced DevOps practices like Infrastructure as Code, automating processes using Azure DevOps to call and run the scripts I would build. There was a lot of figuring things out as I went. I didn't leverage a lot of the tools that would later become the standard for such efforts.

After progressing out of operational IT in 2019, leaving my Enterprise Engineer role for a Technical Account Manager role at VMware, there were just technologies that I did not get a chance to touch. A lot of the DevOps I had practice with was scripting-based, leveraging languages like PowerShell and Python, and automating processes using scheduling or triggers. I didn't get to experience technologies like Terraform, Chef, or Ansible. And as more applications make the shift from VMs to containers I have yet to gain experience on Kubernetes. While at VMware I was exposed to these technologies through the customer I supported, but I lacked hands-on experience and I only experienced these technologies through the lens of VMware.

And now, as an Enterprise Solution Engineer at SHI, I make it a point to understand these technologies at a level where I can be genuinely useful in technical conversations with customers. I know their place in the tech stack. I have a high-level understanding of how these technologies are leveraged. But I've always been a hands-on learner, and my hands-on experience in these areas is lacking. 

That gap bothered me enough to do something about it. So I'm building a home lab.

This blog is the paper trail.

---

## What Is a Home Lab?

If you already know, or if you're already building one, skip this. If you landed here from a search: a home lab is a personal, self-managed infrastructure environment. Mine will live in my home office, directly behind where I work every day. It's used for the purpose of learning, experimentation, and breaking things without consequence. No change control tickets. No production impact. No explaining to anyone why the cluster is down at 2am.

The hardware is usually repurposed enterprise gear (cheap on eBay once it's a generation or two old), and the whole point is to run real workloads, real platforms, and real configurations, not simulations.

Previously in my career, I never needed one as I always had access to equipment or environments that I could leverage as a lab. That changed when I left operational IT in 2019.

---

## What I'm Building

The plan has three phases.

### Phase 1: The Foundation

Before anything interesting happens, there's hardware to rack and cable, a network to design, and a management plane to set up. This is the boring necessary work, and I'm going to document it anyway, because "boring necessary work" is exactly what breaks people's labs six months in when they skip it. It's also the work I often found skipped in environments I had to clean up as a Professional Services Engineer. Better to start from a solid base.

Phase 1 covers:
- Hardware selection and physical setup
- Networking: VLANs, a proper management network, storage networking, DNS, virtual routing/firewalling
- A repeatable base I can rebuild from if something goes catastrophically wrong

### Phase 2: Hypervisor

I've spent time with VMware ESXi/vSphere, Microsoft Hyper-V, and Citrix XenServer. The obvious next step would be to lean on one of those. I can probably get VMUG licensing for VMware (just renewed my VCP), and my XenServer experience maps directly to XCP-ng. But the point of this lab is to learn things I haven't touched, not to get comfortable with things I already know.

Proxmox VE is one major hypervisor I haven't run. It's KVM under the hood, which matters as I move into automation and container orchestration. Terraform has a solid Proxmox provider, Ansible integrates cleanly, and every Kubernetes platform I plan to evaluate has been documented running on Proxmox extensively. It gives me a stable, well-documented foundation without locking me into a commercial licensing model.

This isn't a hypervisor evaluation series. In the future, I might look at other hypervisors but for now, Proxmox is the platform, and everything else gets built on top of it.


### Phase 3: Automation, IaC, and Kubernetes

This is where I'm spending the most time. Automation platforms like Ansible and Terraform are standard practice in any hybrid cloud environment, and container orchestration at scale is where enterprise infrastructure is going. I want hands-on experience with these platforms.

I'll be evaluating infrastructure automation platforms and various opinionated flavors (and management layers) of Kubernetes. I'll likely dip my toes into various monitoring solutions as well. 

The longer goal behind all this goes beyond the lab itself. I've been playing with AI tools like Claude Code and Cursor to build small pet projects for fun. My goal is to take the knowledge I learn using this lab and expand those pet projects to some fun coding projects that I can use this new environment to deploy, test, and then break. Over and over! This will also help me expand my knowledge in modern development practices.

---

## Why Document It?

A few reasons.

Writing forces clarity. If I can't explain what I'm doing well enough to write it down, I probably don't understand it well enough to trust it. The act of writing is part of the learning, not just a record of it.

I've benefited enormously from other people's lab write-ups. The various people who documented their Proxmox lab builds have already helped me a ton. I want to add to that body of work, if for no other reason than to prove an old dog can still learn new tricks.

And honestly, having a public record of what I'm learning keeps me accountable to finishing it. Nothing like knowing someone might read this to make sure I actually finish the Kubernetes series instead of stalling out after the first install.

---

## What's Next

The hardware is racked. The next post covers the physical setup: what I'm working with, how it's configured, and the decisions I made on hardware and networking before a single VM was provisioned.

If you've done this before, [reach out](mailto:matt.virtually@gmail.com). I'm interested in what you'd do differently and what I'm probably about to get wrong.

---

*This is post 1 of an ongoing series. [View all posts in this series](/homelab-blog/posts/).*
