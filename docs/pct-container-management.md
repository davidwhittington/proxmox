# Container Management CLI Guide

> `pct` is the Proxmox command-line tool for managing **LXC containers** — lightweight OS-level
> virtualization that shares the host kernel. Containers start faster, use less memory, and have
> near-native performance, but they're Linux-only and share the host kernel namespace.
>
> For full QEMU/KVM virtual machines, see the [VM Management CLI Guide](qm-vm-management.md).

---

## TL;DR — The Commands You Actually Use Every Day

| What you want to do | Command |
|---------------------|---------|
| See all containers and status | `pct list` |
| Get a root shell inside a container | `pct enter <ctid>` — exit with `exit` or `Ctrl+D` |
| Open console with login prompt | `pct console <ctid>` — exit with `Ctrl+A` then `q` |
| Start a stopped container | `pct start <ctid>` |
| Cleanly shut down a container | `pct shutdown <ctid>` |
| Kill a container that won't respond | `pct stop <ctid>` |
| Reboot | `pct reboot <ctid>` |
| See a container's config | `pct config <ctid>` |
| Change a setting | `pct set <ctid> --memory 2048` |
| Run a command without entering | `pct exec <ctid> -- uptime` |
| Take a snapshot before changes | `pct snapshot <ctid> pre-upgrade` |
| Roll back a snapshot | `pct rollback <ctid> pre-upgrade` |

That covers 90% of day-to-day use. Read on for deep dives, or jump to [Advanced Operations](#advanced-operations).

---

## Listing Containers

```bash
pct list
```

Sample output:

```
VMID       Status     Lock         Name
201        running                 zebra
305        stopped                 meerkat
901        running                 warthog
911        running                 lemur
```

---

## Entering a Container

The primary way to get a shell inside a running container:

```bash
pct enter <ctid>
```

This drops you into the container's root namespace directly — no password, no SSH. You get a
root shell immediately.

```bash
pct enter 201
# root@zebra:~#
```

**To exit:** type `exit` or press `Ctrl+D`

### Console access

For a proper console session (with login prompt, like a serial terminal):

```bash
pct console <ctid>
```

**To exit:** press `Ctrl+A` then `q`

The difference: `pct enter` gives you a namespaced shell directly (no login prompt), while
`pct console` attaches to the container's `/dev/console` and presents a getty login.

---

## Starting, Stopping, and Restarting

```bash
# Start a stopped container
pct start <ctid>

# Graceful shutdown (sends SIGPWR — init shuts down cleanly)
pct shutdown <ctid>

# Force stop immediately
pct stop <ctid>

# Reboot
pct reboot <ctid>

# Suspend to disk
pct suspend <ctid>

# Resume from suspend
pct resume <ctid>
```

| Command | Guest notified? | Use when |
|---------|----------------|----------|
| `pct shutdown` | Yes | Normal off |
| `pct stop` | No | Container is frozen or unresponsive |
| `pct reboot` | Yes | Normal restart |

---

## Checking Container Status

```bash
# Single container
pct status <ctid>

# All containers
pct list
```

```bash
pct status 201
# status: running
```

---

## Viewing and Modifying Config

```bash
pct config <ctid>
```

Sample output:

```
arch: amd64
cores: 2
hostname: zebra
memory: 2048
nameserver: 8.8.8.8
net0: name=eth0,bridge=vmbr0,firewall=1,hwaddr=BC:24:11:AA:BB:CC,ip=dhcp,type=veth
onboot: 1
ostype: ubuntu
rootfs: local-lvm:vm-201-disk-0,size=16G
swap: 512
```

### Modifying config with pct set

```bash
# Change memory
pct set <ctid> --memory 4096

# Change core count
pct set <ctid> --cores 4

# Set hostname
pct set <ctid> --hostname new-name

# Add a mount point
pct set <ctid> --mp0 /host/path,mp=/container/path

# Enable autostart
pct set <ctid> --onboot 1
```

---

## Creating a Container

```bash
pct create <ctid> <template> \
  --hostname <name> \
  --memory <mb> \
  --cores <n> \
  --rootfs <storage>:<size-in-gb> \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --ostype ubuntu \
  --unprivileged 1
```

Example — create a 2-core, 2GB Ubuntu container:

```bash
pct create 202 local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --hostname pangolin \
  --memory 2048 \
  --cores 2 \
  --rootfs local-lvm:16 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --ostype ubuntu \
  --unprivileged 1
pct start 202
```

> Unprivileged containers run as a non-root user on the host, which limits what they can do but
> significantly improves security isolation.

---

## Snapshots

```bash
# Create a snapshot
pct snapshot <ctid> <name> --description "before upgrade"

# List snapshots
pct listsnapshot <ctid>

# Roll back to a snapshot
pct rollback <ctid> <name>

# Delete a snapshot
pct delsnapshot <ctid> <name>
```

> **Storage:** Snapshots require ZFS, LVM-thin, or Ceph. Plain LVM and directories do not support them.
>
> **Warning — pct rollback:** Rolling back replaces the container's current state with the snapshot. Any data written or config changes made after the snapshot was taken are permanently lost. Take a new snapshot first if you want to preserve the current state.

---

## Cloning

```bash
# Full clone
pct clone <ctid> <new-ctid> --hostname <new-name> --full

# Linked clone (if storage supports it)
pct clone <ctid> <new-ctid> --hostname <new-name>
```

---

## Running Commands Inside a Container

Without entering an interactive shell:

```bash
pct exec <ctid> -- <command>

# Examples
pct exec 201 -- uptime
pct exec 201 -- bash -c "df -h /"
pct exec 201 -- systemctl status nginx
```

---

## File Transfer

Push a file from the host into a container:

```bash
pct push <ctid> <host-path> <container-path>

# Example
pct push 201 /tmp/config.yaml /etc/app/config.yaml
```

Pull a file from a container to the host:

```bash
pct pull <ctid> <container-path> <host-path>

# Example
pct pull 201 /var/log/app.log /tmp/app.log
```

---

## Mounting and Unmounting Container Filesystem

Access a stopped container's filesystem from the host without starting it:

```bash
# Mount (container must be stopped)
pct mount <ctid>
# Filesystem available at /var/lib/lxc/<ctid>/rootfs/

# Unmount when done
pct unmount <ctid>
```

Useful for recovering files from a container that won't start.

---

## Destroying a Container

> **Warning:** `pct destroy` permanently deletes the container and all its disks. There is no undo. Snapshot or back up first.

```bash
# Container must be stopped first
pct destroy <ctid>
```

---

## Available Templates

```bash
# List downloaded templates
pveam list local

# Search available templates
pveam available --section system

# Download a template
pveam download local ubuntu-24.04-standard_24.04-2_amd64.tar.zst
```

---

## Advanced Operations

Less common commands for disk management, container lifecycle, migration, diagnostics, and restore.

### Disk Operations

```bash
# Grow rootfs by 10GB
pct resize <ctid> rootfs +10G

# Set absolute size on an additional mount point
pct resize <ctid> mp0 20G

# Move volume to different storage
pct move-volume <ctid> rootfs ceph-pool

# Move volume to a different container
pct move-volume <ctid> mp0 local-lvm --target-vmid <new-ctid> --target-volume mp0

# Force storage rescan
pct rescan
```

### Container Lifecycle

```bash
# Restore from a vzdump backup archive
pct restore <ctid> /var/lib/vz/dump/lxc-<ctid>-*.tar.zst --storage local-lvm

# Convert to template (irreversible — clone first if unsure)
pct template <ctid>

# Destroy a container and all its disks (must be stopped)
# WARNING: permanent, no undo
pct destroy <ctid>

# Show config changes pending a restart
pct pending <ctid>

# Suspend / resume
pct suspend <ctid>
pct resume <ctid>
```

### Migration

```bash
# Migrate to another node in the same cluster
pct migrate <ctid> <target-node>
pct migrate <ctid> <target-node> --target-storage local-lvm

# Migrate to a different Proxmox cluster
pct remote-migrate <ctid> [<target-ctid>] <target-endpoint> \
  --target-bridge vmbr0 \
  --target-storage local-lvm
```

> Note: LXC containers typically stop during migration, unlike QEMU VMs which can live-migrate. Check if your shared storage setup supports online migration.

### Diagnostics & Recovery

```bash
# Show disk usage inside a container
pct df <ctid>

# Filesystem check (container must be stopped first)
pct fsck <ctid>

# Trim free space on thin-provisioned storage
pct fstrim <ctid>

# Show which host CPU cores containers are pinned to
pct cpusets

# Remove a stuck lock (only when no operation is actively running)
# WARNING: clearing a live lock can corrupt an in-progress operation
pct unlock <ctid>
```

### Full Command Reference

| Command | What it does | |
|---------|-------------|---|
| `pct list` | List all containers | |
| `pct status <ctid>` | Show status | |
| `pct start <ctid>` | Start container | |
| `pct shutdown <ctid>` | Graceful shutdown | |
| `pct stop <ctid>` | Force stop | |
| `pct reboot <ctid>` | Reboot | |
| `pct suspend <ctid>` | Suspend to disk | |
| `pct resume <ctid>` | Resume from suspend | |
| `pct enter <ctid>` | Root shell — exit: `Ctrl+D` | |
| `pct console <ctid>` | Console with getty — exit: `Ctrl+A` then `q` | |
| `pct exec <ctid> -- <cmd>` | Run command non-interactively | |
| `pct config <ctid>` | Show configuration | |
| `pct set <ctid> [OPTIONS]` | Modify configuration | |
| `pct pending <ctid>` | Show reboot-pending changes | |
| `pct snapshot <ctid> <name>` | Create snapshot | |
| `pct listsnapshot <ctid>` | List snapshots | |
| `pct rollback <ctid> <name>` | Roll back to snapshot | ⚠️ |
| `pct delsnapshot <ctid> <name>` | Delete snapshot | |
| `pct clone <ctid> <newid>` | Clone container | |
| `pct create <ctid> <template>` | Create new container | |
| `pct restore <ctid> <archive>` | Restore from backup | |
| `pct template <ctid>` | Convert to template (irreversible) | ⚠️ |
| `pct destroy <ctid>` | Delete container and all disks | ⚠️ |
| `pct migrate <ctid> <node>` | Migrate to another node | |
| `pct remote-migrate <ctid> ...` | Migrate to different cluster | |
| `pct resize <ctid> <disk> <size>` | Resize a disk | |
| `pct move-volume <ctid> <vol> ...` | Move volume to storage | |
| `pct rescan` | Force storage rescan | |
| `pct push <ctid> <file> <dest>` | Push file to container | |
| `pct pull <ctid> <path> <dest>` | Pull file from container | |
| `pct mount <ctid>` | Mount stopped container | |
| `pct unmount <ctid>` | Unmount container | |
| `pct df <ctid>` | Container disk usage | |
| `pct fsck <ctid>` | Filesystem check (stopped only) | ⚠️ |
| `pct fstrim <ctid>` | Trim free space | |
| `pct cpusets` | Show CPU set assignments | |
| `pct unlock <ctid>` | Remove stuck lock | ⚠️ |
| `pct help` | Show help | |

⚠️ = destructive or requires care — read the relevant section before running.

---

## Quick Reference

| Task | Command |
|------|---------|
| List all containers | `pct list` |
| Enter container (root shell) | `pct enter <ctid>` |
| Exit `pct enter` | `exit` or `Ctrl+D` |
| Open console (login prompt) | `pct console <ctid>` |
| Exit `pct console` | `Ctrl+A` then `q` |
| Start container | `pct start <ctid>` |
| Graceful shutdown | `pct shutdown <ctid>` |
| Force stop | `pct stop <ctid>` |
| Reboot | `pct reboot <ctid>` |
| View config | `pct config <ctid>` |
| Modify config | `pct set <ctid> --<option> <value>` |
| Create container | `pct create <ctid> <template> ...` |
| Create snapshot | `pct snapshot <ctid> <name>` |
| List snapshots | `pct listsnapshot <ctid>` |
| Roll back snapshot | `pct rollback <ctid> <name>` |
| Clone container | `pct clone <ctid> <new-ctid> --hostname <name> --full` |
| Run command in container | `pct exec <ctid> -- <command>` |
| Push file to container | `pct push <ctid> <host-path> <ctid-path>` |
| Pull file from container | `pct pull <ctid> <ctid-path> <host-path>` |
| Mount stopped container | `pct mount <ctid>` |
| Unmount container | `pct unmount <ctid>` |
| Destroy container | `pct destroy <ctid>` |
| List templates | `pveam list local` |

---

## Changelog

| Date | Note |
|------|------|
| 2026-03-20 | Initial document |
| 2026-03-20 | Added TL;DR table; added Advanced Operations section (disk, lifecycle, migration, diagnostics, full command reference); added destructive command warnings |
