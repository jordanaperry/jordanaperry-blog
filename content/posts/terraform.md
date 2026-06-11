---
title: "Infrastructure as Code with Terraform"
date: 2026-06-09
draft: false
tags: ["homelab", "terraform", "iac", "proxmox", "devops", "iam"]
series: ["Homelab Journey"]
description: "Managing every Proxmox container with Terraform — importing existing infrastructure, cloud-init SSH key injection, and why IaC matters for IAM engineering roles."
cover:
  image: "/images/banners/banner-post-18.svg"
  alt: "Infrastructure as Code with Terraform"
  relative: false
---

Every container in this lab was built manually at first. The configuration lived in my head and in Proxmox's database. If `leviathan` died and I had to rebuild from scratch, I'd be working from notes and memory — slow, error-prone, and incomplete.

Terraform fixes this. Every container is now defined in code: its IP, VLAN, storage, CPU, RAM, startup order, and SSH keys. Rebuilding the entire lab from zero is `terraform apply`. The code lives in Gitea, mirrors to GitHub, and serves as both documentation and the authoritative definition of what should exist.

This post covers the setup, the `bpg/proxmox` provider, the import workflow for containers that already existed, and the cloud-init pattern for injecting SSH keys automatically.

---

## Why IaC matters for IAM engineering

Before getting into the technical setup — this is worth calling out explicitly because it's relevant to the job search.

IAM engineers at modern companies don't provision access through GUIs. They manage identity resources — Okta applications, AWS IAM roles, Azure AD groups, Vault policies — as code using Terraform. A candidate who can say "I manage my homelab infrastructure with Terraform and the same pattern applies to Okta's provider" is more compelling than one who only knows the web UI.

The homelab Terraform config is public on GitHub at `github.com/jordanaperry/homelab-terraform`. Every container, every resource, reviewable by a hiring manager in five minutes.

---

## Setup

Terraform is installed on the Mac — the lab control plane is `manta`, not a server.

```bash
# Install via Homebrew
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Verify
terraform version
# Terraform v1.15.4 on darwin_arm64
```

Project lives at `~/homelab-terraform/` with a VS Code workspace. Two files are gitignored and never committed:
- `terraform.tfvars` — contains the Proxmox API token
- `terraform.tfstate` — contains the current state

---

## The bpg/proxmox provider

The official HashiCorp Proxmox provider is limited. The `bpg/proxmox` community provider is what people actually use — it supports LXC containers, VLAN tags, cloud-init, and most of what Proxmox exposes through its API.

`versions.tf`:

```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "0.106.0"
    }
  }
}
```

`provider.tf`:

```hcl
provider "proxmox" {
  endpoint  = "https://leviathan.jordanaperry.com:8006"
  api_token = var.proxmox_api_token
  insecure  = false
}
```

The API token is created in Proxmox at **Datacenter → Permissions → API Tokens**. It needs `PVEAdmin` role on the node. Store it in `terraform.tfvars`:

```hcl
proxmox_api_token = "root@pam!terraform=your-token-here"
```

---

## A typical container definition

Here's what a container resource looks like — this is CT 105 (`monitor`):

```hcl
resource "proxmox_virtual_environment_container" "monitor" {
  node_name    = "leviathan"
  vm_id        = 105
  description  = "Prometheus + Grafana monitoring"
  started      = true
  unprivileged = true

  features {
    nesting = true
  }

  cpu {
    cores = 2
  }

  memory {
    dedicated = 1024
  }

  disk {
    datastore_id = "wreck-lvm"
    size         = 20
  }

  network_interface {
    name     = "eth0"
    bridge   = "vmbr0"
    vlan_id  = 40
    firewall = false
  }

  initialization {
    hostname = "monitor"
    ip_config {
      ipv4 {
        address = "192.168.40.40/24"
        gateway = "192.168.40.1"
      }
    }
    dns {
      servers = ["192.168.40.1"]
    }
  }

  operating_system {
    template_file_id = "local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst"
    type             = "debian"
  }
}
```

A few things worth noting:

**`firewall = false`** on every container — enabling it causes VLAN tagging issues regardless of what the firewall rules say. This was learned the hard way across multiple containers.

**`vlan_id = 40`** — this is how VLAN tagging is applied at the container level. Combined with `vmbr0` (the VLAN-aware bridge), the container gets traffic on the correct VLAN.

**`nesting = true`** in features — required for some containers (Grafana, in particular) to function correctly.

**`wreck-lvm`** as `datastore_id` for all new containers — iSCSI-backed storage on the NAS.

---

## Importing existing containers

The first eight containers were built manually before Terraform was set up. Importing them into state without destroying and recreating them requires the `terraform import` command plus a `lifecycle` block that tells Terraform to ignore fields it can't read back from the API.

Import workflow:

```bash
# Write the resource block in main.tf first, then import
terraform import proxmox_virtual_environment_container.helm leviathan/100
terraform import proxmox_virtual_environment_container.blog leviathan/101
terraform import proxmox_virtual_environment_container.bastion leviathan/102
# ... etc
```

Imported containers need this lifecycle block to prevent Terraform from trying to update fields that were set at creation time and can't be retrieved from the API:

```hcl
lifecycle {
  ignore_changes = [
    initialization,
    operating_system,
    network_interface[0].mac_address,
  ]
}
```

New containers created by Terraform (CT 104 onwards) don't need this — Terraform has the full creation context in state.

---

## Cloud-init SSH key injection

Starting with CT 107 (Vault), new containers get SSH keys injected automatically via Terraform. No more logging into the Proxmox console to manually add `authorized_keys`.

```hcl
initialization {
  hostname = "vault"
  ip_config {
    ipv4 {
      address = "192.168.40.32/24"
      gateway = "192.168.40.1"
    }
  }
  dns {
    servers = ["192.168.40.1"]
  }
  user_account {
    keys = [
      trimspace(file("~/.ssh/id_ed25519_vault.pub"))
    ]
  }
}
```

The `trimspace()` call is important — SSH key files often have a trailing newline that causes authentication failures if included verbatim.

This pattern will be applied to all future containers. The SSH key lives on the Mac, gets embedded in the Terraform config at apply time, and the container comes up with key auth already configured.

---

## Common commands

```bash
terraform init        # initialise providers
terraform plan        # preview changes — always run before apply
terraform apply       # create or update resources
terraform destroy     # destroy a resource (use carefully)
terraform state list  # show all resources in state
terraform import      # import an existing resource into state
```

---

## Handling drift

Occasionally Proxmox state drifts from Terraform state — a container is modified manually (like bumping CT 100's RAM to fix the Unifi controller issue) and Terraform doesn't know about it.

The fix is to update the Terraform resource to match reality, then run `terraform plan` to confirm it shows no changes, then `terraform apply` if needed.

```bash
terraform plan
# Should show: No changes. Infrastructure is up-to-date.
```

If you see unexpected changes, review them carefully before applying. Terraform will try to reconcile state with config, which can mean restarting containers.

---

## The repo

`homelab-terraform` on Gitea, mirrored to `github.com/jordanaperry/homelab-terraform`. The full container inventory, provider config, and variables file (without secrets) are all there.

![VS Code showing homelab-terraform repo with main.tf open](/images/terraform.png)
*homelab-terraform in VS Code — all 8 containers defined as code*

---

## What's next

Infrastructure is fully codified. That wraps up the core homelab build series — network, compute, storage, monitoring, access control, and IaC are all in place.

The next chapter is the IAM stack: Keycloak for SSO, FreeIPA as the identity directory, HashiCorp Vault for secrets management, and eventually Okta. That's where the lab gets interesting from a career perspective — and it's where the next set of posts picks up.