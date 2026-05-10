---
title: "Setting Up the Workstation: Git, GitHub, and Terraform"
date: 2026-05-10
draft: false
weight: 4
series: ["Series 2 — Automation and IaC"]
tags: ["homelab", "terraform", "git", "github", "iac", "devops"]
summary: "Before writing a single line of Terraform, the workstation needs to be ready. Setting up Git, building a GitHub strategy for the whole lab, and initializing the first Terraform project."
showToc: true
TocOpen: false
---

Before anything in the lab gets automated, something has to run the automation. That means getting my laptop into a state where it can serve as a real workstation for infrastructure work, not just a browser and a terminal I use to SSH into things.

This post is a shorter one. Life has been busy and my lab time has been limited lately, but I made enough progress to document it. Sometimes a post is a sprint and sometimes it's a few productive hours on a weekend. Both count.

---

## The GitHub Strategy

I've been using GitHub already to publish this blog. Every post goes through a Git commit and push, and GitHub Actions handles the Hugo build and deployment automatically. That workflow forced me to actually learn Git rather than just use it as a file upload mechanism, and it's been a good foundation.

The decision to put everything lab-related in GitHub rather than just the blog was a deliberate one. Here's the reasoning:

**One platform, one workflow.** Blog posts, Terraform configurations, Kubernetes manifests, application code for future projects — all of it lives in GitHub. Same tools, same mental model, same version history regardless of what I'm working on.

**Version control for infrastructure is not optional.** When a Terraform config breaks something, I want to be able to see exactly what changed and roll it back. When I iterate on a cloud-init template, I want a history of every version. Git makes this free and automatic.

**GitHub Actions as a future automation layer.** The same GitHub Actions pipeline that builds this blog can eventually run `terraform plan` on pull requests, lint Kubernetes manifests, or trigger deployments. The foundation is already there.

**A portfolio that builds itself.** Every repo is a record of what I've actually built, not just what I claim to know. That matters when talking to customers about these technologies.

The current repo structure, and what each one will eventually hold:

| Repository | Purpose |
|------------|---------|
| `homelab-blog` | This blog, Hugo source, GitHub Pages deployment |
| `homelab-terraform` | All Proxmox infrastructure definitions as code |
| Future repos | Kubernetes manifests, application projects, CI/CD experiments |

---

## Learning Git Beyond the Blog

Using Git for the blog taught me the basics: `git add`, `git commit`, `git push`, and how to recover when a push fails because of a history conflict. Useful, but narrow.

Moving into infrastructure work means Git becomes a more active tool. Branches for experimenting with config changes without touching the main state. Pull requests as a forcing function to review what's actually changing before it applies. Commit messages that explain the why, not just the what, because three months from now I won't remember why I made a particular networking decision.

A few habits I'm building from the start:

```bash
# Descriptive commit messages over lazy ones
git commit -m "Add Proxmox provider config with API token auth"
# Not: git commit -m "updates"

# Check what's staged before committing
git status
git diff --staged

# Branch for experiments
git checkout -b feature/cloud-init-template
```

None of this is groundbreaking for someone who's been using Git professionally for years. For someone coming from a world of change tickets and static configs, it's a different way of thinking about infrastructure changes entirely.

---

## Installing Terraform

Terraform is the first automation tool going into the stack. The Proxmox provider for Terraform is mature and well-documented, and it maps cleanly to how Proxmox exposes its API. More on that in the next post.

On Windows, the simplest install path is through winget:

```powershell
winget install HashiCorp.Terraform
```

Close and reopen the terminal after install, then verify:

```bash
terraform version
```

That should return the installed version. If it doesn't, the PATH update from the installer didn't take — restarting the terminal usually resolves it.

---

## Setting Up the homelab-terraform Repository

The repo was created on GitHub first, then cloned locally. Same pattern as the blog repo.

```bash
git clone https://github.com/mattvirtually-source/homelab-terraform.git
cd homelab-terraform
```

Initial folder structure inside the repo:

```
homelab-terraform/
├── main.tf          # Provider config and core resources
├── variables.tf     # Input variables
├── outputs.tf       # Output values
└── .gitignore       # Keeps credentials and state files out of the repo
```

The `.gitignore` is not optional. Terraform generates a state file (`terraform.tfstate`) that can contain sensitive information including API credentials. It should never be committed to a public repository.

A minimal Terraform `.gitignore` for this project:

```
# Terraform state files
*.tfstate
*.tfstate.backup
.terraform/
*.tfvars
crash.log
```

With the folder structure in place and the `.gitignore` configured, Terraform gets initialized:

```bash
terraform init
```

On a fresh directory with no provider configuration yet, `terraform init` won't do much beyond confirming the working directory is valid. The provider block and API connection come next, once the Proxmox side is ready.

---

## What's Next

The workstation is ready. The repo is in place. The next step moves back to the Proxmox environment to prepare it for Terraform-managed provisioning.

Two things need to happen on the Proxmox side before a single VM can be created with code:

1. **An API token.** Terraform communicates with Proxmox through its REST API. Creating a dedicated service account with a scoped API token is the right approach — no root credentials, least privilege, token can be revoked without touching anything else.

2. **A cloud-init template.** Rather than installing an OS manually every time a VM is needed, cloud-init allows a base image to be configured at first boot with hostname, user accounts, SSH keys, and network settings passed in at provision time. The template gets created once, cloned as many times as needed. That template is what the Kubernetes node VMs will be built from.

Those two pieces, along with the first working Terraform configuration to provision a VM, are the next post.

---

*This is post 4 of an ongoing series. [Start from the beginning](/posts/post-01-the-vision/) or [view all posts in this series](/posts/).*
