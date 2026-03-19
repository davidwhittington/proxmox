# Deploying Ubuntu 24.04 LTS on Proxmox VE via Cloud-Init

Fully automated VM provisioning — no interactive installer, SSH-ready in under 2 minutes.

---

## Why Cloud Image over ISO

The Ubuntu cloud image skips the interactive installer entirely. Cloud-init handles everything on first boot: user creation, SSH key injection, DHCP network configuration, and hostname assignment. The image is reusable as a template base — download once, clone as many VMs as needed.

**Benefits:**
- No interactive installer — fully automated
- Cloud-init handles user creation, SSH key injection, network (DHCP), hostname
- Boots to an SSH-accessible system in under 2 minutes
- Image is reusable as a template base for future VMs

---

## Prerequisites

- Proxmox VE host with at least one bridge (e.g. `vmbr0`)
- Storage pool that supports disk images (e.g. `local-lvm` lvmthin)
- SSH access to the Proxmox host as root
- An SSH public key to inject

---

## Step 1 — Gather Environment Info

Before creating anything, confirm what's available on the host.

```bash
# Next available VM ID
pvesh get /cluster/nextid

# Available storage pools
pvesm status

# Network bridges
cat /etc/network/interfaces | grep -E '^iface|^auto'

# Node name
hostname
```

---

## Step 2 — Download the Ubuntu 24.04 Cloud Image

```bash
# Fetch cloud image to Proxmox ISO repository (skips download if already present)
wget -O /var/lib/vz/template/iso/noble-cloudimg-amd64.img \
    https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
```

The image is approximately 600MB. Check for an existing copy before downloading to avoid pulling it again unnecessarily.

---

## Step 3 — Create the VM

> **Note — VM ID 100 is illustrative.** The commands below use `100` as the VM ID throughout this guide. If you already have VMs running, use an ID that isn't taken. To check:
> ```bash
> # List existing VMs and their IDs
> qm list
>
> # Get the next available ID from Proxmox
> pvesh get /cluster/nextid
> ```

```bash
# Create VM shell — hardware profile only, no disk or drives yet
qm create 100 \
    --name ollama \
    --memory 16384 \
    --cores 6 \
    --sockets 1 \
    --cpu host \
    --net0 virtio,bridge=vmbr0 \
    --ostype l26 \
    --machine q35 \
    --scsihw virtio-scsi-pci \
    --onboot 0
```

**Parameter notes:**
- `--cpu host` — exposes host CPU features to the VM; best for performance, avoid if live migration is needed
- `--machine q35` — modern chipset; required for PCIe passthrough (Phase 2)
- `--onboot 0` — VM does not auto-start with Proxmox; change to `1` if desired
- `--scsihw virtio-scsi-pci` — required for `scsi0` disk attachment

---

## Step 4 — Import and Attach the Disk

```bash
# Import cloud image as VM disk
qm importdisk 100 /var/lib/vz/template/iso/noble-cloudimg-amd64.img local-lvm

# Attach as primary SCSI disk with discard and SSD flags
qm set 100 --scsi0 local-lvm:vm-100-disk-0,discard=on,ssd=1

# Resize to desired capacity (cloud image is ~3.5GB raw)
qm resize 100 scsi0 100G
```

---

## Step 5 — Add Cloud-Init Drive

```bash
# Attach cloud-init ISO to VM
qm set 100 --ide2 local-lvm:cloudinit
```

This creates a small ISO attached as a CD-ROM. The VM reads it on first boot to configure itself. Without this, cloud-init has no source for its user data.

---

## Step 6 — Configure Cloud-Init

The `--sshkeys` flag injects your public key into the VM's `authorized_keys` on first boot, enabling key-based SSH login without a password. Don't have a key pair yet? See [SSH Public Key Authentication](ssh-pubkey-auth.md).

```bash
# Write SSH public key to temp file
echo "{YOUR_PUBKEY}" > /tmp/vm-key.pub

# Apply cloud-init settings
qm set 100 \
    --ciuser {username} \
    --cipassword '{YOUR_PASSWORD}' \
    --sshkeys /tmp/vm-key.pub \
    --ipconfig0 ip=dhcp \
    --nameserver 8.8.8.8 \
    --searchdomain local

# Clean up
rm /tmp/vm-key.pub
```

---

## Step 7 — Boot Configuration

```bash
# Set boot order to primary disk
qm set 100 --boot order=scsi0

# Add serial console (required for cloud-init serial output + noVNC serial tab)
qm set 100 --serial0 socket --vga serial0
```

---

## Step 8 — Start the VM

```bash
# Start the VM — cloud-init runs on first boot and configures the system
qm start 100
```

---

## Step 9 — Find the VM's IP Address

The VM gets its IP via DHCP from the network router — not from Proxmox. Proxmox does not track DHCP leases. To find the assigned IP:

```bash
# Ping-sweep the subnet to populate the ARP table
for i in $(seq 1 254); do ping -c1 -W1 192.168.X.$i > /dev/null 2>&1 & done
wait

# Look up the VM's MAC address (visible in qm config output)
arp -an | grep -i "BC:24:11:XX:XX:XX"
```

The MAC address is shown in `qm config 100` under `net0`.

---

## Step 10 — Connect via SSH

```bash
# First connection — suppress host key prompt since this is a fresh VM
ssh -o StrictHostKeyChecking=no {username}@{VM_IP}
```

Add a permanent alias to `~/.ssh/config` on the source Mac:

```
# Add to ~/.ssh/config for a persistent alias — then just: ssh ollama
Host ollama
    HostName {VM_IP}
    User {username}
    IdentityFile ~/.ssh/id_rsa
```

Then connect with `ssh ollama`. For a full walkthrough of key generation, distribution, and disabling password auth, see [SSH Public Key Authentication](ssh-pubkey-auth.md).

---

## Step 11 — Create Your User Account (if using default ubuntu user)

If cloud-init created a default `ubuntu` user and you want a named account, this block creates the user, sets a password, installs your SSH public key, locks down `.ssh` directory permissions, and grants passwordless sudo:

```bash
# Create the user with a home directory and bash shell, add to sudo group
sudo useradd -m -s /bin/bash -G sudo {username}

# Set the user's password
echo '{username}:{YOUR_PASSWORD}' | sudo chpasswd

# Create .ssh directory and install your public key for key-based login
sudo mkdir -p /home/{username}/.ssh
echo "{YOUR_PUBKEY}" | sudo tee /home/{username}/.ssh/authorized_keys

# Lock down .ssh permissions — SSH will reject keys if these are too open
sudo chmod 700 /home/{username}/.ssh
sudo chmod 600 /home/{username}/.ssh/authorized_keys
sudo chown -R {username}:{username} /home/{username}/.ssh

# Grant passwordless sudo
echo '{username} ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/{username}
```

---

## Final VM Configuration Reference

| Setting | Value |
|---------|-------|
| VM ID | 100 |
| Name | ollama |
| CPU | 6 cores, host type |
| RAM | 16384 MB |
| Disk | 100G, local-lvm, discard+ssd |
| Machine | q35 |
| Network | virtio, vmbr0, DHCP |
| Cloud-init drive | ide2 |
| Serial console | serial0 socket |
| Auto-start | disabled (onboot 0) |

---

## Notes

**Proxmox MOTD:** The default Proxmox installation outputs large ANSI art on login. When scripting SSH commands from a Mac, strip it:

```bash
ssh root@{PROXMOX_IP} "command" | python3 -c "import sys,re; [print(re.sub(r'\x1b\[[0-9;]*[A-Za-z]','',l).strip()) for l in sys.stdin if l.strip()]"
```

**DHCP discovery:** If the VM's IP isn't showing up via ARP, the lease is held by the network router, not Proxmox. A subnet ping-sweep populates the ARP table so you can grep for the MAC.

**Re-running cloud-init:** Cloud-init only runs on first boot. To re-run it after changing settings:

```bash
sudo cloud-init clean && sudo reboot
```

**Phase 2:** GPU passthrough (AMD Vega 56 via Thunderbolt 4 eGPU) — see `proxmox-egpu-spec.md` in this directory.

---

## Changelog

- **2026-03-18:** Initial deployment documented
