# Container Management CLI Guide

> `pct` is the Proxmox command-line tool for managing **LXC containers** — lightweight OS-level
> virtualization that shares the host kernel. Containers start faster, use less memory, and have
> near-native performance, but they're Linux-only and share the host kernel namespace.
>
> For full QEMU/KVM virtual machines, see the [VM Management CLI Guide](qm-vm-management.md).

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

> Snapshots require the storage backend to support them (ZFS, LVM-thin, Ceph). Plain LVM
> and directories do not support container snapshots.

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

```bash
pct destroy <ctid>
```

The container must be stopped first. This removes the container config and all its disks.

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
