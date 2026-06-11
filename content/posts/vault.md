---
title: "Running Keycloak at Home: SSO for Your Homelab"
date: 2026-06-10
draft: false
tags: ["homelab", "keycloak", "iam", "sso", "oidc", "identity", "proxmox"]
series: ["Homelab Journey"]
description: "Installing Keycloak 26 on Proxmox, connecting it to FreeIPA via LDAP, and wiring up Grafana and Proxmox SSO — plus the FreeIPA OTP incident that took down everything."
cover:
  image: "/images/banners/banner-post-20.svg"
  alt: "Running Keycloak at Home: SSO for Your Homelab"
  relative: false
---

Static passwords are a liability. They don't expire, they get shared, they show up in dotfiles and shell history, and when something gets compromised you have to hunt down every place that credential was used. HashiCorp Vault solves this — it's a secrets management platform that stores credentials encrypted, logs every access, enforces policy-based access control, and can generate short-lived dynamic credentials on demand.

For a homelab targeting IAM engineering roles, Vault is one of the most valuable things you can run. It's the same technology used in enterprise environments to manage secrets at scale, and understanding how it works — initialization, unsealing, policies, auth methods, secrets engines — is directly relevant to roles that involve secrets management, PAM, and zero-trust access.

This post covers the full Vault setup: installation, initialization, LDAP auth via FreeIPA, a KV secrets store, audit logging, and the SSH Certificate Authority that replaced every static authorized_keys file in the lab.

---

## The container

CT 107 on `leviathan`, Debian 12, VLAN 40 (midnight):

```
Hostname: vault
IP: 192.168.40.32/24
Storage: wreck-lvm (iSCSI), 10GB
RAM: 1GB
```

This was the first container provisioned with SSH key injection via Terraform cloud-init — no manual `authorized_keys` step needed. Covered in the Terraform post.

---

## Installation

Vault is available via HashiCorp's apt repository:

```bash
apt update && apt install -y gpg curl lsb-release

curl -fsSL https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  tee /etc/apt/sources.list.d/hashicorp.list

apt update && apt install -y vault
```

One gotcha: install `lsb-release` before running the `echo` command. If it's missing, `$(lsb_release -cs)` returns an empty string and the repo entry is malformed — Vault won't install and the error message won't tell you why.

---

## SSL certificate

Same two-step process via Mac as every other service:

```bash
# On Mac — pull from pfSense
scp admin@192.168.1.1:/var/etc/cert.crt /tmp/vault.crt
scp admin@192.168.1.1:/var/etc/cert.key /tmp/vault.key

# Push to vault container
scp /tmp/vault.crt root@192.168.40.32:/etc/ssl/vault.crt
scp /tmp/vault.key root@192.168.40.32:/etc/ssl/vault.key

# On vault — set permissions
chown root:vault /etc/ssl/vault.crt /etc/ssl/vault.key
chmod 640 /etc/ssl/vault.crt /etc/ssl/vault.key
```

---

## Configuration

`/etc/vault.d/vault.hcl`:

```hcl
ui = true

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-1"
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/etc/ssl/vault.crt"
  tls_key_file  = "/etc/ssl/vault.key"
}

api_addr     = "https://vault.jordanaperry.com:8200"
cluster_addr = "https://vault.jordanaperry.com:8201"
disable_mlock = true
```

`disable_mlock = true` is required in Vault 2.0+ — it must be explicitly set or Vault fails to start.

Enable and start:

```bash
chown -R vault:vault /opt/vault/data
systemctl enable --now vault
```

---

## Initialization and unsealing

```bash
export VAULT_ADDR="https://vault.jordanaperry.com:8200"
vault operator init
```

This outputs 5 unseal keys and 1 root token. All 6 go straight into Bitwarden — this is the only time they're shown.

**How unsealing works** — Vault starts sealed after every restart. The data is encrypted with a master key that Vault doesn't hold in memory. The master key is split into 5 pieces using Shamir's Secret Sharing — any 3 of the 5 reconstruct it. Until you unseal, Vault rejects every request.

```bash
vault operator unseal  # run 3 times with 3 different keys
vault status           # Sealed: false ✅
```

In production you'd configure auto-unseal using AWS KMS, Azure Key Vault, or an HSM so Vault unseals automatically after a restart. For a homelab, manual unseal is fine — it's a good excuse to understand the mechanism.

![Vault UI showing initialized and unsealed status](/images/vault.png)
*Vault web UI at vault.jordanaperry.com:8200 — unsealed and ready*

---

## LDAP auth via FreeIPA

Rather than managing Vault users separately, Vault authenticates against FreeIPA via LDAP — the same directory that feeds Keycloak.

```bash
vault auth enable ldap

vault write auth/ldap/config \
  url="ldap://192.168.40.31" \
  binddn="uid=keycloak-bind,cn=users,cn=accounts,dc=jordanaperry,dc=com" \
  bindpass="<from-bitwarden>" \
  userdn="cn=users,cn=accounts,dc=jordanaperry,dc=com" \
  userattr="uid" \
  groupdn="cn=groups,cn=accounts,dc=jordanaperry,dc=com" \
  groupattr="cn" \
  groupfilter="(&(objectClass=groupofnames)(member={{.UserDN}}))" \
  insecure_tls=true
```

Create the admin policy and map the FreeIPA group to it:

```bash
vault policy write homelab-admin - << 'EOF'
path "*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
EOF

vault write auth/ldap/groups/homelab-admins policies=homelab-admin
vault write auth/ldap/users/jordanp policies=homelab-admin
```

Test it:

```bash
vault login -method=ldap username=jordanp
# Enter FreeIPA password
# Login successful — token issued
```

One gotcha: if you see 403 errors after logging in successfully, the group lookup probably failed — Vault authenticated the user but couldn't map the group to the policy. Check `insecure_tls` is set and the `groupdn`/`groupattr` values match your FreeIPA schema.

---

## KV secrets engine

The KV (key-value) secrets engine is the simplest engine — it's an encrypted store for static credentials. All homelab passwords now live here instead of in scattered notes.

```bash
vault secrets enable -path=secret kv-v2

vault kv put secret/keycloak admin_password="..."
vault kv put secret/freeipa admin_password="..." dir_manager_password="..."
vault kv put secret/grafana admin_password="..."
```

KV v2 supports versioning — every write creates a new version and old versions are retained. You can roll back to a previous value if needed.

![Vault KV secrets engine showing secret paths](/images/kv-engine.png)
*Vault KV secrets engine — homelab credentials stored and version-controlled*

---

## Audit logging

Every request to Vault — reads, writes, auth attempts, everything — can be logged to a file. This is one of the features that makes Vault genuinely useful from an IAM perspective: a complete audit trail of who accessed what secret and when.

```bash
mkdir -p /opt/vault/logs
chown vault:vault /opt/vault/logs
vault audit enable file file_path=/opt/vault/logs/audit.log
```

The audit log is JSON format, one entry per line. The actual secret values are hashed — you can see that a secret was read but not what the value was.

---

## SSH Certificate Authority

This is the most interesting part of the setup and the most directly relevant to enterprise IAM. Instead of static `authorized_keys` files on every server, Vault acts as an SSH Certificate Authority. You sign your public key with Vault, getting a short-lived certificate that's trusted by every server. No static keys, automatic expiry, full audit trail.

### Configure the CA

```bash
vault secrets enable ssh

vault write ssh/config/ca generate_signing_key=true

# Save the public key — this goes on every server
vault read -field=public_key ssh/config/ca
```

### Create the signing role

The role defines what the CA is allowed to sign. Extensions matter here — without `permit-pty`, the signed cert will authenticate successfully but give you a connection with no interactive terminal (no prompt, no output). Not obvious from the error.

The CLI has JSON parsing issues with the extensions field — use a file:

```bash
cat > /tmp/role.json << 'EOF'
{
  "key_type": "ca",
  "allow_user_certificates": true,
  "allowed_users": "root,jordanp",
  "allowed_extensions": "permit-pty,permit-port-forwarding",
  "default_extensions": {
    "permit-pty": "",
    "permit-port-forwarding": ""
  },
  "ttl": "8h",
  "max_ttl": "24h"
}
EOF

vault write ssh/roles/homelab-admin @/tmp/role.json
```

Certificates expire after 8 hours — sign a new cert each working session. This is intentional: short-lived credentials are better than permanent ones.

### Deploy the CA public key to every server

On each server that should trust Vault-signed certs:

```bash
echo "<vault-ca-public-key>" > /etc/ssh/vault_ca.pub
echo "TrustedUserCAKeys /etc/ssh/vault_ca.pub" >> /etc/ssh/sshd_config
systemctl restart ssh    # Debian
systemctl restart sshd   # Rocky Linux (FreeIPA)
```

Deployed to: bastion, keycloak, leviathan, monitor, ipa, vault.

### Sign a cert and SSH

On the Mac:

```bash
export VAULT_ADDR="https://vault.jordanaperry.com:8200"

# Log in with FreeIPA credentials
vault login -method=ldap username=jordanp

# Sign your public key
vault write -field=signed_key ssh/sign/homelab-admin \
  public_key=@$HOME/.ssh/id_ed25519.pub \
  valid_principals="root,jordanp" > ~/.ssh/id_ed25519-cert.pub

# SSH using the cert
ssh -i ~/.ssh/id_ed25519 -i ~/.ssh/id_ed25519-cert.pub keycloak
```

![Vault SSH role showing certificate TTL and allowed users](/images/vaultsshrole.png)
*Vault SSH secrets engine — homelab-admin role with 28800s (8h)*

---

## Vault CLI on Mac

```bash
brew install vault
export VAULT_ADDR="https://vault.jordanaperry.com:8200"
```

Add the `export` to `~/.zshrc` so it persists across terminal sessions.

---

## What's next

Vault is running and doing real work: LDAP auth against FreeIPA, credentials stored in KV, and an SSH CA eliminating static keys across the midnight VLAN. Auto-unseal and Keycloak OIDC integration are on the roadmap for when cloud resources get added to the lab.

That wraps up the core IAM stack — FreeIPA as the identity directory, Keycloak for SSO, and Vault for secrets. The lab now has an enterprise-grade identity and access management layer running entirely on-premises.
