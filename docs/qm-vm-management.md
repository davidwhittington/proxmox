# VM Management CLI Guide

> `qm` is the Proxmox command-line tool for managing **QEMU/KVM virtual machines** — full VMs
> with their own kernel, hardware emulation, and optional GPU passthrough.
> For LXC containers, see the [Container Management CLI Guide](pct-container-management.md).

---

## TL;DR — The Commands You Actually Use Every Day

| What you want to do | Command |
|---------------------|---------|
| See all VMs and their status | `qm list` |
| Open a console (no SSH needed) | `qm terminal <vmid>` — exit with `Ctrl+O` |
| Start a stopped VM | `qm start <vmid>` |
| Cleanly shut down a VM | `qm shutdown <vmid>` |
| Kill a VM that won't respond | `qm stop <vmid>` |
| Reboot a VM | `qm reboot <vmid>` |
| See a VM's config | `qm config <vmid>` |
| Change a setting | `qm set <vmid> --memory 4096` |
| Verify guest agent is talking | `qm agent <vmid> ping` |
| Take a snapshot before changes | `qm snapshot <vmid> pre-upgrade` |
| Roll back a snapshot | `qm rollback <vmid> pre-upgrade` |
| Run a command in the guest | `qm guest exec <vmid> -- uptime` |

That covers 90% of day-to-day use. Read on for deep dives, or jump to [Advanced Operations](#advanced-operations).

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

## Sending Keystrokes — qm sendkey



Injects a keypress directly into a running VM. Useful for interacting with GRUB, BIOS, or a
locked console without opening the noVNC viewer.

```bash
qm sendkey <vmid> <key>

# Send Ctrl+Alt+Delete
qm sendkey 100 ctrl-alt-delete

# Press Enter (advance through a bootloader prompt)
qm sendkey 100 ret

# Send Escape
qm sendkey 100 esc

# Send a character
qm sendkey 100 y
```

Key names follow QEMU conventions: `ret`, `esc`, `spc`, `ctrl-c`, `shift-a`, etc.

---

## Advanced Operations

Less common commands for disk management, cloud-init, migrations, diagnostics, and VM import.

### Disk Operations

```bash
# Grow disk by 20GB
qm disk resize <vmid> scsi0 +20G

# Move disk to different storage (--delete removes source after move)
qm disk move <vmid> scsi0 ceph-pool --delete

# Import a disk image file as a new disk
qm disk import <vmid> /path/to/disk.qcow2 local-lvm

# Detach a disk (--purge also deletes the storage volume)
qm disk unlink <vmid> --idlist scsi1
qm disk unlink <vmid> --idlist scsi1 --purge

# Force rescan of storage
qm disk rescan
```

### Cloud-init Management

```bash
# Show the rendered cloud-init config (user / network / meta)
qm cloudinit dump <vmid> user
qm cloudinit dump <vmid> network

# Show pending cloud-init changes not yet written to the ISO
qm cloudinit pending <vmid>

# Regenerate the cloud-init ISO after config changes
qm cloudinit update <vmid>
```

### VM Lifecycle

```bash
# Suspend VM to disk / resume
qm suspend <vmid>
qm resume <vmid>

# Convert a stopped VM to a template (irreversible)
qm template <vmid>

# Delete a VM and all its disks (must be stopped)
qm destroy <vmid>
qm destroy <vmid> --purge   # also remove from jobs and replication

# Show config changes that require a reboot
qm pending <vmid>

# Change a guest user's password via the guest agent
qm guest passwd <vmid> <username>
```

### Migration

```bash
# Live migrate to another node in the same cluster
qm migrate <vmid> <target-node>
qm migrate <vmid> <target-node> --targetstorage local-lvm

# Migrate to a different Proxmox cluster entirely
qm remote-migrate <vmid> [<target-vmid>] <target-endpoint> \
  --target-bridge vmbr0 \
  --target-storage local-lvm
```

### Diagnostics & Recovery

```bash
# Print the exact QEMU command line Proxmox uses to start the VM
qm showcmd <vmid>

# Open a raw QMP monitor session (low-level QEMU interface)
qm monitor <vmid>

# Remove a stuck lock after a failed migration or snapshot
qm unlock <vmid>

# Block in a script until the VM reaches a state (running/stopped)
qm wait <vmid> --timeout 60

# Check the result of a previously started async guest exec
qm guest exec-status <vmid> <pid>
```

### Importing VMs

```bash
# Import a disk image to Proxmox storage
qm import <vmid> /path/to/disk.img --storage local-lvm

# Import an OVF/OVA from VMware or VirtualBox
tar -xf vm.ova
qm importovf <vmid> vm.ovf local-lvm

# Enroll Secure Boot EFI keys (required for new UEFI VMs with SB enabled)
qm enroll-efi-keys <vmid>
```

### Full Command Reference

| Command | What it does |
|---------|-------------|
| `qm list` | List all VMs |
| `qm status <vmid>` | Show status of a single VM |
| `qm start <vmid>` | Start a stopped VM |
| `qm shutdown <vmid>` | Graceful shutdown via ACPI |
| `qm stop <vmid>` | Force stop |
| `qm reboot <vmid>` | Graceful reboot |
| `qm reset <vmid>` | Hard reset |
| `qm suspend <vmid>` | Suspend to disk |
| `qm resume <vmid>` | Resume from suspend |
| `qm terminal <vmid>` | Serial console (exit: Ctrl+O) |
| `qm sendkey <vmid> <key>` | Inject a keypress |
| `qm monitor <vmid>` | Open QMP monitor |
| `qm config <vmid>` | Show config |
| `qm set <vmid> [OPTIONS]` | Modify config |
| `qm pending <vmid>` | Show reboot-pending changes |
| `qm showcmd <vmid>` | Print raw QEMU command line |
| `qm snapshot <vmid> <name>` | Create snapshot |
| `qm listsnapshot <vmid>` | List snapshots |
| `qm rollback <vmid> <name>` | Roll back to snapshot |
| `qm delsnapshot <vmid> <name>` | Delete snapshot |
| `qm clone <vmid> <newid>` | Clone a VM |
| `qm create <vmid>` | Create a new VM |
| `qm destroy <vmid>` | Delete VM and disks |
| `qm template <vmid>` | Convert to template |
| `qm migrate <vmid> <node>` | Migrate to another node |
| `qm remote-migrate <vmid> ...` | Migrate to different cluster |
| `qm unlock <vmid>` | Remove stuck lock |
| `qm wait <vmid>` | Wait for VM state in scripts |
| `qm disk resize <vmid> <disk> <size>` | Resize a disk |
| `qm disk move <vmid> <disk> <storage>` | Move disk to storage |
| `qm disk import <vmid> <src> <storage>` | Import disk image |
| `qm disk unlink <vmid> --idlist <disk>` | Detach/delete disk |
| `qm disk rescan` | Force storage rescan |
| `qm cloudinit dump <vmid> <type>` | Show rendered cloud-init |
| `qm cloudinit pending <vmid>` | Pending cloud-init changes |
| `qm cloudinit update <vmid>` | Regenerate cloud-init ISO |
| `qm guest exec <vmid> -- <cmd>` | Run command via agent |
| `qm guest exec-status <vmid> <pid>` | Check async exec status |
| `qm guest passwd <vmid> <user>` | Set guest user password |
| `qm import <vmid> <source>` | Import disk image |
| `qm importovf <vmid> <ovf> <storage>` | Import OVF/OVA |
| `qm enroll-efi-keys <vmid>` | Enroll Secure Boot keys |
| `qm cleanup <vmid> ...` | Clean up after failed op |
| `qm vncproxy <vmid>` | Start VNC proxy |
| `qm nbdstop <vmid>` | Stop NBD server |
| `qm help` | Show help |

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
| Send keystrokes to VM | `qm sendkey <vmid> <key>` |

---

## Changelog

| Date | Note |
|------|------|
| 2026-03-20 | Initial document |
| 2026-03-20 | Retitled to VM Management CLI Guide; added qm sendkey; linked to container guide |
| 2026-03-20 | Added TL;DR section; added full Advanced Operations section (disk, cloud-init, migration, diagnostics, import, full command reference) |
