# SSH Public Key Authentication — Proxmox & Guest VMs

Password-free access across your lab. This guide covers three connection paths:

- **Client → Proxmox host** — manage the hypervisor from your Mac without a password
- **Client → Guest VMs** — direct access from your Mac to any VM, via cloud-init injection or manually after the fact
- **Proxmox → Guest VMs** — for automation, scripts, or backups running on the Proxmox host that SSH into guests

---

## Prerequisites

- A client machine (Mac or Linux) with SSH installed
- SSH access to the Proxmox host (password-based initially, to bootstrap)
- At least one guest VM running and reachable on the network

> **Confirm key access before disabling passwords.** Keep the current session open, open a second terminal, and verify key-based login works before locking out passwords. Don't lock yourself out.

---

## Step 1 — Generate a Key Pair

Run this on your **client machine** (Mac). Skip if you already have a key at `~/.ssh/id_rsa` or `~/.ssh/id_ed25519`.

```bash
# Check for an existing key pair first
ls ~/.ssh/id_*.pub

# Generate a new Ed25519 key (preferred — smaller and faster than RSA)
ssh-keygen -t ed25519 -C "{your_email_or_label}"

# Or RSA 4096 if you need broader compatibility
ssh-keygen -t rsa -b 4096 -C "{your_email_or_label}"
```

```bash
# Print your public key — this is what you copy to remote hosts
cat ~/.ssh/id_ed25519.pub
```

The output looks like: `ssh-ed25519 AAAA... your_label`. This is the string you'll distribute. The private key (`id_ed25519`, no `.pub`) never leaves your machine.

---

## Step 2 — Client → Proxmox Host

```bash
# Copy your public key to the Proxmox root account
ssh-copy-id root@{PROXMOX_IP}
```

You'll be prompted for the root password once. Then verify:

```bash
# Should connect without a password prompt
ssh root@{PROXMOX_IP}
```

**Manual method (if ssh-copy-id isn't available):**

```bash
# Append your public key over SSH in one line
cat ~/.ssh/id_ed25519.pub | ssh root@{PROXMOX_IP} "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

---

## Step 3 — Disable Password Auth (Optional Hardening)

Run this on the **Proxmox host** after confirming key-based login works:

```bash
# Harden SSH — disable password and root password auth
sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config

# Reload SSH daemon to apply changes
systemctl reload sshd

# Verify the settings took effect
grep -E 'PasswordAuthentication|PermitRootLogin' /etc/ssh/sshd_config
```

`PermitRootLogin prohibit-password` keeps root login available via key but rejects password attempts.

---

## Step 4 — Client → Guest VMs

### New VM — Inject via Cloud-Init (preferred)

```bash
# Write your public key to a temp file on the Proxmox host
echo "{YOUR_PUBKEY}" > /tmp/vm-key.pub

# Inject the key during cloud-init configuration (run on Proxmox host)
qm set {VMID} --sshkeys /tmp/vm-key.pub

# Clean up
rm /tmp/vm-key.pub
```

See [ubuntu-cloudinit-vm.md](ubuntu-cloudinit-vm.md) for the full provisioning workflow.

### Existing VM — Push Key Manually

```bash
# Copy your client public key to a running guest VM
ssh-copy-id {username}@{VM_IP}

# Verify
ssh {username}@{VM_IP}
```

---

## Step 5 — Proxmox → Guest VMs

For automation, scripts, or health checks running on the Proxmox host.

### Generate a Key Pair on Proxmox

```bash
# Run on the Proxmox host as root
ssh-keygen -t ed25519 -C "proxmox-host" -f ~/.ssh/id_ed25519 -N ""

# Print the Proxmox host's public key
cat ~/.ssh/id_ed25519.pub
```

The `-N ""` sets an empty passphrase — appropriate for automated/non-interactive use.

### Push the Proxmox Key to a Guest VM

```bash
# Push Proxmox host key to a guest VM (run on Proxmox host)
ssh-copy-id {username}@{VM_IP}

# Test the connection from Proxmox to the guest
ssh {username}@{VM_IP} "hostname && uptime"
```

### Relay via Client (if guest has no password auth)

```bash
# From your client Mac — fetch Proxmox's pubkey and push it to the guest
PROXMOX_PUBKEY=$(ssh root@{PROXMOX_IP} "cat ~/.ssh/id_ed25519.pub")
ssh {username}@{VM_IP} "echo '$PROXMOX_PUBKEY' >> ~/.ssh/authorized_keys"
```

---

## Step 6 — SSH Config Aliases

### Client ~/.ssh/config

```
# Add to ~/.ssh/config on your client Mac

Host proxmox
    HostName {PROXMOX_IP}
    User root
    IdentityFile ~/.ssh/id_ed25519

Host ollama
    HostName {VM_IP}
    User {username}
    IdentityFile ~/.ssh/id_ed25519
```

```bash
# Test your aliases
ssh proxmox "pveversion"
ssh ollama "hostname"
```

### Proxmox ~/.ssh/config (for Proxmox → VM access)

```
# Add to ~/.ssh/config on the Proxmox host

Host ollama
    HostName {VM_IP}
    User {username}
    IdentityFile ~/.ssh/id_ed25519
```

After this, scripts running on Proxmox can call `ssh ollama "command"` without any IP or credential management.

---

## Troubleshooting

### Permission Denied (publickey)

Almost always a file permissions issue. SSH silently rejects keys if `authorized_keys` or `.ssh` is world-writable.

```bash
# Fix permissions on the remote host
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chown -R $USER:$USER ~/.ssh
```

### Wrong Key Being Offered

```bash
# Verbose output shows which key is being tried
ssh -v {username}@{HOST_IP}

# Force a specific identity file
ssh -i ~/.ssh/id_ed25519 {username}@{HOST_IP}
```

### Key Not in authorized_keys

```bash
# Check what keys are authorized on the remote host
cat ~/.ssh/authorized_keys

# Compare against your local public key
cat ~/.ssh/id_ed25519.pub
```

### Check Auth Log

```bash
# Detailed rejection reasons on the remote host
sudo tail -50 /var/log/auth.log | grep sshd
```

---

## Changelog

| Date | Entry |
|------|-------|
| 2026-03-19 | Initial guide: key generation, client-to-Proxmox, client-to-VM, Proxmox-to-VM, SSH config aliases, troubleshooting |
