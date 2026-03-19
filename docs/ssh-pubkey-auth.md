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

## Post-Quantum KEX & Legacy Hosts

OpenSSH 8.5 introduced `sntrup761x25519-sha512@openssh.com` as its default key exchange algorithm — a hybrid combining classical elliptic-curve with a post-quantum algorithm. OpenSSH 9.9 (shipped with macOS Sequoia and current Linux distributions) promoted `mlkem768x25519-sha256` (ML-KEM, NIST FIPS 203) to the top of the preference list.

When your client is newer than the server, you may see warnings, negotiation failures, or silent fallback depending on the version gap.

---

### What the Warnings Look Like

A full failure — no common algorithm found:

```
Unable to negotiate with {HOST_IP} port 22: no matching key exchange method found.
Their offer: diffie-hellman-group14-sha1,diffie-hellman-group-exchange-sha256
```

Silent fallback is more common. The connection succeeds but you're not getting the algorithm you expected. Run `ssh -v` to see what was actually negotiated:

```bash
ssh -v {username}@{HOST_IP} 2>&1 | grep -i "kex\|key exchange"
```

### Check What the Server Supports

```bash
# Query supported KEX algorithms on the server
sshd -T | grep kexalgorithms

# Or check the OpenSSH version
sshd --version
```

---

### How to Suppress the Warnings

> **Read the risk section below before applying any of these.** Suppression forces a permanent cryptographic downgrade for that connection. It's not a neutral change.

Per-host in `~/.ssh/config` — the right approach for a specific legacy machine:

```
# Target only the host that needs it
Host old-server
    HostName {HOST_IP}
    User {username}
    # Remove post-quantum algorithms from the offer for this host only
    KexAlgorithms -mlkem768x25519-sha256,-sntrup761x25519-sha512@openssh.com
```

Per-connection, one-off:

```bash
# Force classical KEX for a single connection
ssh -o KexAlgorithms=curve25519-sha256,ecdh-sha2-nistp521 {username}@{HOST_IP}
```

Global suppression in `~/.ssh/config` — affects every host, almost never the right answer:

```
# Global override — think carefully before using this
Host *
    KexAlgorithms -mlkem768x25519-sha256,-sntrup761x25519-sha512@openssh.com
```

---

### Why You Should Think Carefully Before Suppressing

**Harvest now, decrypt later.** Nation-state actors and well-resourced adversaries are known to collect encrypted traffic today with the intent of decrypting it once cryptographically relevant quantum computers become viable. This is not a theoretical future concern — it's a documented present-day practice. Anything sent over a classical-only KEX session today could be in someone's archive waiting for that moment.

The specific risks:

- **Long-lived secrets become vulnerable.** SSH sessions carry credentials, private keys, and configuration data. If the session is recorded and later decrypted, everything in it is exposed.
- **The fallback is permanent for that host.** Once you add a `KexAlgorithms` override, it stays in place until someone removes it and upgrades the server. You're committing to weaker cryptography on every future connection.
- **The warning is useful signal.** It's telling you the server is running outdated software. Suppressing it doesn't fix that — it just removes the reminder.
- **Global suppression spreads the risk everywhere.** A `Host *` override disables post-quantum protection for every host, including future ones that do support it. You've downgraded your entire SSH posture.
- **Lab habits become production habits.** If you routinely silence crypto warnings in your home lab, that reflex carries over. Warnings exist to be acted on.

---

### The Right Fix: Update the Server

Post-quantum KEX is available in OpenSSH 8.5+ (sntrup761) and 9.9+ (ML-KEM). Ubuntu 24.04 ships OpenSSH 9.6. Proxmox VE 8.x on Debian 12 ships OpenSSH 9.2. Neither should generate KEX warnings against a current macOS client. If you're seeing them, update the server first.

```bash
# Update OpenSSH on Ubuntu/Debian
sudo apt update && sudo apt upgrade openssh-server
```

For genuinely legacy hosts that can't be updated — old network equipment, embedded systems, inherited infrastructure — scoped per-host suppression is the pragmatic choice. Apply it to the minimum set of hosts that actually need it, document why, and review it periodically. Don't let a temporary workaround quietly become permanent policy.

> **In the context of this lab:** Proxmox VE 8+ on Debian 12 and Ubuntu 24.04 both support `sntrup761x25519-sha512`. If you're following the cloud-init provisioning guide and connecting from macOS Sequoia or later, KEX negotiation should succeed without any overrides. If it doesn't, update the server before reaching for a config workaround.

---

## Changelog

| Date | Entry |
|------|-------|
| 2026-03-19 | Add post-quantum KEX section: warnings, risks, suppression methods, and why to update the server instead |
| 2026-03-19 | Initial guide: key generation, client-to-Proxmox, client-to-VM, Proxmox-to-VM, SSH config aliases, troubleshooting |
