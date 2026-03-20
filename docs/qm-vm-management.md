# qm VM Management — Proxmox Command Line

> `qm` is the Proxmox command-line tool for managing QEMU/KVM virtual machines.
> Everything you can do in the web UI you can do with `qm` — faster, and scriptable.

---

## Listing VMs

```bash
qm list
```

Sample output:

```
      VMID NAME                 STATUS     MEM(MB)    BOOTDISK(GB) PID
       100 giraffe              running       49152        200.00 2847301
       200 flamingo             running        2048         32.00 2847618
       304 pangolin             running        2048         32.00 136623
       900 tapir                running        4096         32.00 2848012
       910 okapi                running        4096         32.00 2848199
```

Column guide:

| Column | Meaning |
|--------|---------|
| VMID | Numeric ID used in all `qm` commands |
| NAME | Friendly name set at creation time |
| STATUS | `running`, `stopped`, `paused` |
| MEM(MB) | RAM allocated to the VM |
| BOOTDISK(GB) | Size of the primary disk |
| PID | Host kernel PID of the QEMU process (running VMs only) |

---

## Serial Console — `qm terminal`

Once a VM has a serial port configured (see setup section below), you can open a direct console without needing SSH.

```bash
qm terminal <vmid>
```

Example:

```bash
qm terminal 100
```

Output:

```
starting serial terminal on interface serial0 (press Ctrl+O to exit)

giraffe login: david
Password:
Last login: Fri Mar 20 08:14:22 UTC 2026 from 192.168.6.1 on pts/0

david@giraffe:~$
```

**To exit:** press `Ctrl+O`

The terminal is a raw serial connection — no SSH required, works even if the network is down or SSH is misconfigured. Useful for debugging a broken network stack, editing `/etc/netplan/`, or recovering a locked-out VM.

---

## Setting Up Serial Console (if `qm terminal` is not working)

### Why it might not work

- Serial port not added to the Proxmox VM config
- Guest OS not configured to use `ttyS0` as a console
- Guest started before the serial port was added (needs a stop/start cycle)

### Step 1 — Add serial port in Proxmox

Run on the Proxmox host:

```bash
qm set <vmid> --serial0 socket
```

> This takes effect on the next full stop/start — a `qm reset` is not enough.

### Step 2 — Configure the guest OS

SSH into the VM and run:

```bash
# Update grub to output to serial
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT="console=tty0 console=ttyS0,115200n8"/' /etc/default/grub
sudo update-grub

# Enable serial getty so you get a login prompt
sudo systemctl enable serial-getty@ttyS0.service
```

### Step 3 — Restart the VM from Proxmox

```bash
qm stop <vmid>
qm start <vmid>
```

### Verify

```bash
qm terminal <vmid>
# Should show: starting serial terminal on interface serial0 (press Ctrl+O to exit)
```

---

## Starting, Stopping, and Restarting

```bash
# Start a stopped VM
qm start <vmid>

# Graceful shutdown (sends ACPI power signal — OS shuts down cleanly)
qm shutdown <vmid>

# Force stop immediately (like pulling the power plug)
qm stop <vmid>

# Graceful reboot
qm reboot <vmid>

# Hard reset (like hitting the reset button)
qm reset <vmid>
```

For bulk operations across multiple VMs:

```bash
for id in 200 304 900 910; do qm stop $id --timeout 15 & done; wait
```

---

## Checking VM Status

```bash
# One VM
qm status <vmid>

# All VMs
qm list
```

Example:

```bash
qm status 100
# status: running
```

---

## Viewing VM Configuration

```bash
qm config <vmid>
```

Shows the current config including CPU, memory, disks, network interfaces, serial ports, cloud-init settings, and agent status.

```bash
qm config 100
# agent: enabled=1,type=virtio
# boot: order=scsi0
# cores: 6
# machine: q35
# memory: 49152
# name: giraffe
# net0: virtio=BC:24:11:C1:2E:67,bridge=vmbr0
# serial0: socket
# vga: serial0
# ...
```

---

## Modifying VM Configuration

```bash
# Change memory (takes effect on next boot)
qm set <vmid> --memory 8192

# Add a serial port
qm set <vmid> --serial0 socket

# Enable QEMU guest agent
qm set <vmid> --agent enabled=1,type=virtio

# Change core count
qm set <vmid> --cores 4

# Set boot order
qm set <vmid> --boot order=scsi0
```

---

## QEMU Guest Agent

The guest agent enables Proxmox to communicate with the VM — reporting IP addresses, flushing filesystem buffers before snapshots, and executing graceful shutdowns.

### Check agent status from Proxmox

```bash
# Ping the agent (no output = success)
qm agent <vmid> ping

# Get IP addresses reported by the guest
qm agent <vmid> network-get-interfaces

# Get agent info (version, capabilities)
qm agent <vmid> info
```

### Enable agent in VM config

```bash
qm set <vmid> --agent enabled=1,type=virtio
```

### Install agent in guest (Ubuntu/Debian)

```bash
sudo apt-get install -y qemu-guest-agent
sudo systemctl start qemu-guest-agent
```

The service starts automatically when the virtio-serial device is present — no `systemctl enable` needed.

---

## Snapshots

```bash
# Create a snapshot
qm snapshot <vmid> <name> --description "before upgrade"

# List snapshots
qm listsnapshot <vmid>

# Roll back to a snapshot
qm rollback <vmid> <name>

# Delete a snapshot
qm delsnapshot <vmid> <name>
```

> If the QEMU guest agent is running, Proxmox will freeze the filesystem before snapshotting for a consistent state. Without the agent, the snapshot is crash-consistent.

---

## Cloning

```bash
# Full clone (independent copy)
qm clone <vmid> <new-vmid> --name <new-name> --full

# Linked clone (shares base storage — faster, less disk space, tied to source)
qm clone <vmid> <new-vmid> --name <new-name>
```

---

## Sending Commands via Guest Agent

```bash
# Run a command in the guest without SSH
qm guest exec <vmid> -- <command>

# Examples
qm guest exec 100 -- uptime
qm guest exec 100 -- bash -c "df -h /"
```

---

## Quick Reference

| Task | Command |
|------|---------|
| List all VMs | `qm list` |
| Open serial console | `qm terminal <vmid>` |
| Exit serial console | `Ctrl+O` |
| Start VM | `qm start <vmid>` |
| Graceful shutdown | `qm shutdown <vmid>` |
| Force stop | `qm stop <vmid>` |
| Reboot | `qm reboot <vmid>` |
| Hard reset | `qm reset <vmid>` |
| View config | `qm config <vmid>` |
| Modify config | `qm set <vmid> --<option> <value>` |
| Ping guest agent | `qm agent <vmid> ping` |
| Get IPs from guest | `qm agent <vmid> network-get-interfaces` |
| Create snapshot | `qm snapshot <vmid> <name>` |
| List snapshots | `qm listsnapshot <vmid>` |
| Roll back snapshot | `qm rollback <vmid> <name>` |
| Clone VM | `qm clone <vmid> <new-vmid> --name <name> --full` |
| Run command in guest | `qm guest exec <vmid> -- <command>` |

---

## Changelog

| Date | Note |
|------|------|
| 2026-03-20 | Initial document |
