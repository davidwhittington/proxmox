# Proxmox TB4 eGPU Passthrough: Complete Implementation Spec

> Intel NUC (Meteor Lake) + Blackmagic eGPU Pro (Vega 56) + Proxmox VE 9.1.6
> Target: GPU passthrough to Ubuntu VM for Ollama LLM inference

---

## Hardware

| Component | Detail |
|-----------|--------|
| Host machine | Intel NUC, Meteor Lake (Core Ultra 7 165H) |
| Proxmox version | VE 9.1.6, kernel 6.17.13-2-pve |
| eGPU | Blackmagic eGPU **Pro** — AMD Radeon RX Vega 56, 8GB HBM2 |
| GPU PCIe ID (GPU) | `1002:687f` |
| GPU PCIe ID (HDMI audio) | `1002:aaf8` |
| TB controller | Intel JHL7440 Thunderbolt 3 Bridge (Titan Ridge) — TB3 device on TB4 host |
| Target VM | VM 100 "ollama", Ubuntu 24.04, 6 vCPU, 16GB RAM |

**Important**: The Blackmagic eGPU Pro uses Vega 56 (gfx900 / Vega 10 architecture). The non-Pro model uses an RX 580. These are very different GPUs with different PCIe IDs and different ROCm support status. Confirm which model you have before proceeding.

---

## Table of Contents

1. [IOMMU Verification](#phase-1-iommu-verification)
2. [The Cable Problem](#phase-2-the-cable-problem)
3. [Thunderbolt Authorization](#phase-3-thunderbolt-authorization)
4. [vfio-pci Configuration](#phase-4-vfio-pci-configuration)
5. [Boot-With-eGPU-Connected Problem](#phase-5-the-boot-with-egpu-connected-problem)
6. [NIC Renaming Problem](#phase-6-the-nic-renaming-problem)
7. [AMD Vega Reset Bug and vendor-reset](#phase-7-amd-vega-reset-bug-and-vendor-reset)
8. [VM Configuration](#phase-8-vm-configuration)
9. [ROCm Deprecation and the Vulkan Solution](#phase-9-rocm-deprecation-and-the-vulkan-solution)
10. [Verification Commands](#verification-commands)
11. [Troubleshooting](#troubleshooting)
12. [Known Issues / Deprecated](#known-issues--deprecated)
13. [Changelog](#changelog)

---

## Phase 1: IOMMU Verification

Before touching anything else, verify IOMMU groups. You need the GPU and its audio function in isolated groups -- groups they don't share with other devices you'd want to keep on the host.

```bash
find /sys/kernel/iommu_groups/ -type l | sort -V
```

### Without eGPU connected

Relevant TB4-related groups on this NUC at baseline:

| Group | BDF | Device | Notes |
|-------|-----|--------|-------|
| 4 | `00:07.0` | TB4 PCIe Root Port #0 [8086:7ec4] | Isolated -- good |
| 5 | `00:07.2` | TB4 PCIe Root Port #2 [8086:7ec6] | Isolated -- good |
| 9 | `00:0d.0` + `00:0d.2` + `00:0d.3` | TB4 USB ctrl + NHI #0 + NHI #1 | Grouped together -- fine, you don't pass these through |

### After hot-plugging the eGPU

| Group | BDF | Device |
|-------|-----|--------|
| 19 | `2c:00.0` | Intel JHL7440 TB3 Bridge |
| 20-22 | `2d:01-04` | TB3 downstream bridges |
| 23 | `2e:00.0` | AMD Vega 10 PCIe Bridge |
| 24 | `2f:00.0` | AMD Vega 10 PCIe Bridge |
| **25** | **`30:00.0`** | **Radeon RX Vega 56 -- isolated** |
| **26** | **`30:00.1`** | **Vega 56 HDMI Audio -- isolated** |
| 27 | `31:00.0` | Intel JHL7540 TB3 USB Controller |

The GPU (30:00.0) and HDMI audio (30:00.1) each land in their own isolated groups. This is the ideal case for passthrough. You do not need the ACS override patch or any other workaround to split groups here.

---

## Phase 2: The Cable Problem

**Failure mode**: The eGPU appeared as a USB device in dmesg but never enumerated as a PCIe device. `lspci` showed nothing GPU-related regardless of how long you waited.

```
usb 3-3: Product: eGPU Pro
usb 3-3: Manufacturer: Blackmagic Design
usb 3-3: USB disconnect, device number 3
```

The device would connect, immediately disconnect, reconnect in a loop. No `thunderbolt` lines in dmesg at all.

**Why**: USB-C is a connector standard, not a protocol. A USB-C cable can carry USB 3.x, DisplayPort, Power Delivery, or Thunderbolt -- but only if it's a certified Thunderbolt cable. Thunderbolt uses a PCIe tunnel negotiated over the TB protocol layer. A plain USB-C cable has no Thunderbolt signaling capability, so the eGPU enumerates purely as a USB device (its management controller), and the PCIe tunnel that would expose the GPU never opens.

**Fix**: Use a certified Thunderbolt 3 or Thunderbolt 4 cable. Genuine TB cables have a lightning bolt icon on the plug. TB3 and TB4 cables are interoperable with TB3/TB4 ports.

This failure is especially confusing because the eGPU does partially show up -- the management USB device connects. So `lsusb` shows "Blackmagic Design eGPU Pro" and you might think you're on the right track. You're not. The GPU will never appear in `lspci` without Thunderbolt signaling.

---

## Phase 3: Thunderbolt Authorization

With a proper TB cable installed, the device shows up correctly in dmesg:

```
thunderbolt 1-1: new device found, vendor=0x4 device=0xa153
thunderbolt 1-1: Blackmagic Design eGPU Pro
```

But `lspci` still shows no GPU. The TB device is visible at the thunderbolt sysfs layer but the PCIe tunnel hasn't opened.

**Why**: Linux's Thunderbolt security model requires explicit OS-level authorization before establishing a PCIe tunnel to a connected device. The kernel sees the TB device but deliberately holds the tunnel closed until something authorizes it. This is the "SL1" (Secure Connect Level 1) security mode -- the default on most systems.

Check the authorization state:

```bash
cat /sys/bus/thunderbolt/devices/1-1/authorized
# 0 = unauthorized
```

Authorize manually:

```bash
echo 1 > /sys/bus/thunderbolt/devices/1-1/authorized
```

Wait about 3 seconds. The PCIe tunnel opens, the GPU's PCIe bridges enumerate, and the Radeon RX Vega 56 appears in `lspci`. This manual step works for initial testing but isn't suitable for automated operation -- that's handled by the udev rule and boot service in Phase 5.

---

## Phase 4: vfio-pci Configuration

`vfio-pci` is the Linux kernel driver that claims PCIe devices for userspace passthrough. When a device is bound to `vfio-pci` instead of its native driver (`amdgpu`, `radeon`, etc.), QEMU/KVM can present it directly to a VM guest.

### `/etc/modprobe.d/vfio.conf`

```
options vfio-pci ids=1002:687f,1002:aaf8
```

This tells `vfio-pci` to automatically claim any device with those PCIe IDs when they enumerate. Both the GPU and the HDMI audio function need to be claimed together -- you cannot pass through just the GPU and leave the audio on the host; they share the same PCIe slot.

**Do not add softdeps here.** A common configuration you'll see online:

```
softdep amdgpu pre: vfio-pci
```

Do not use this. On a Thunderbolt setup, the GPU doesn't exist at module load time -- it only appears after TB authorization, which happens in userspace. Softdeps configure ordering for modules loading at boot against devices that are present at boot. For a hot-plugged TB device, adding `softdep amdgpu pre: vfio-pci` causes the initramfs to stall waiting for a PCIe device that won't exist until userspace runs. This causes boot hangs.

### `/etc/modules-load.d/vfio.conf`

```
vendor-reset
vfio
vfio_iommu_type1
vfio_pci
```

Order matters: `vendor-reset` must be listed before `vfio_pci` so it's in place to intercept reset calls when `vfio-pci` claims the device. See Phase 7 for why this matters.

After editing either file:

```bash
update-initramfs -u -k all
```

---

## Phase 5: The Boot-With-eGPU-Connected Problem

**Failure mode**: If the Blackmagic eGPU is physically connected to the NUC when Proxmox boots, the machine hangs completely. No console output, no network, no recovery. Hard power cycle required.

**Why**: Thunderbolt PCIe tunneling requires userspace authorization. At boot, the TB4 controller enumerates the connected TB3 device (the eGPU), but the PCIe tunnel doesn't open until something authorizes it. The kernel sees a TB device at a specific bus address, reserves bus numbers for the expected PCIe hierarchy behind it, and waits. Various early boot components can stall on this: QEMU's device scanning, kernel PCI enumeration timeouts, or systemd waiting for expected devices. The exact hang point varies by kernel version, but the root cause is always the same: PCIe device expected at boot, authorization required but not yet available.

The solution is three-component: a udev rule for hot-plug events, a boot service for devices already connected at startup, and a pre-shutdown service to de-authorize before reboot so the next boot is clean. All three connection scenarios (hot-plug, boot with GPU connected, reboot with GPU connected) are fully handled.

### Component 1: udev Rule for Hot-Plug

`/etc/udev/rules.d/99-egpu-vfio.rules`:

```
ACTION=="add", SUBSYSTEM=="thunderbolt", ATTR{vendor_name}=="Blackmagic Design", RUN+="/usr/local/bin/egpu-attach.sh %k"
```

This fires whenever any Blackmagic TB device is connected during normal operation. `%k` passes the kernel device name (e.g., `1-1`) as an argument to the script.

### Component 2: Attach Script

`/usr/local/bin/egpu-attach.sh`:

```bash
#!/bin/bash
LOGFILE=/var/log/egpu-attach.log
echo "[$(date)] eGPU attach triggered: $1" >> $LOGFILE

AUTH_PATH="/sys/bus/thunderbolt/devices/${1}/authorized"
if [ -f "$AUTH_PATH" ]; then
    echo 1 > "$AUTH_PATH"
    echo "[$(date)] Authorized $1" >> $LOGFILE
fi

sleep 3

for vendor_dev in "1002:687f" "1002:aaf8"; do
    vendor=$(echo $vendor_dev | cut -d: -f1)
    device=$(echo $vendor_dev | cut -d: -f2)
    for syspath in /sys/bus/pci/devices/*/; do
        v=$(cat ${syspath}vendor 2>/dev/null | sed 's/0x//')
        d=$(cat ${syspath}device 2>/dev/null | sed 's/0x//')
        if [ "$v" = "$vendor" ] && [ "$d" = "$device" ]; then
            bdf=$(basename $syspath)
            drv=$(readlink ${syspath}driver 2>/dev/null | xargs basename 2>/dev/null)
            if [ "$drv" != "vfio-pci" ]; then
                echo "$bdf" > /sys/bus/pci/devices/${bdf}/driver/unbind 2>/dev/null
                echo "vfio-pci" > /sys/bus/pci/devices/${bdf}/driver_override
                echo "$bdf" > /sys/bus/pci/drivers/vfio-pci/bind 2>/dev/null
                echo "[$(date)] Bound $bdf to vfio-pci" >> $LOGFILE
            fi
        fi
    done
done
echo "[$(date)] eGPU attach complete" >> $LOGFILE
```

The explicit bind loop in this script is mostly redundant -- `options vfio-pci ids=` in `vfio.conf` causes `vfio-pci` to automatically claim devices with those IDs as they enumerate. The script's primary job is TB authorization. The bind loop handles edge cases where `vfio-pci` didn't claim the device (e.g., another driver loaded first), and provides logging.

```bash
chmod +x /usr/local/bin/egpu-attach.sh
```

### Component 3: Boot Authorization Service

Handles the case where the eGPU is already physically connected when Proxmox boots.

`/etc/systemd/system/egpu-boot-auth.service`:

```ini
[Unit]
Description=Blackmagic eGPU Pro - authorize devices connected at boot
After=multi-user.target
Wants=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/egpu-boot-auth.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

`/usr/local/bin/egpu-boot-auth.sh`:

```bash
#!/bin/bash
LOGFILE=/var/log/egpu-attach.log
echo "[$(date)] eGPU boot-auth: scanning for connected devices" >> $LOGFILE

for dev in /sys/bus/thunderbolt/devices/*/; do
    name=$(cat ${dev}device_name 2>/dev/null)
    auth=$(cat ${dev}authorized 2>/dev/null)
    if [ "$name" = "eGPU Pro" ] && [ "$auth" = "0" ]; then
        echo 1 > ${dev}authorized 2>/dev/null
        echo "[$(date)] Boot-authorized $(basename $dev)" >> $LOGFILE
        sleep 5
        echo "[$(date)] Boot-auth complete for $(basename $dev)" >> $LOGFILE
    fi
done
```

This runs after `multi-user.target` -- by then, systemd is far enough along that the PCIe enumeration triggered by TB authorization won't cause hangs.

```bash
chmod +x /usr/local/bin/egpu-boot-auth.sh
```

### Component 4: Pre-Shutdown De-authorization Service

This is the piece most guides omit. Without it, the shutdown-then-reboot cycle will hang on the next boot because the eGPU is still authorized (and therefore PCIe-active) during reboot. The boot sequence hits the PCIe device before userspace is ready to handle it.

`/etc/systemd/system/egpu-shutdown.service`:

```ini
[Unit]
Description=Blackmagic eGPU Pro - de-authorize before shutdown/reboot
DefaultDependencies=no
Before=shutdown.target reboot.target halt.target
After=sysinit.target

[Service]
Type=oneshot
ExecStart=/bin/true
ExecStop=/usr/local/bin/egpu-detach.sh
RemainAfterExit=yes
TimeoutStopSec=15

[Install]
WantedBy=multi-user.target
```

**Critical design note on the service pattern**: The shutdown service must use `ExecStart=/bin/true` with `RemainAfterExit=yes`, placing the real work in `ExecStop`. This is not obvious. If you wire the service directly to shutdown targets via `WantedBy=shutdown.target`, systemd generates a destructive transaction conflict when you try to start a VM. The error looks like:

```
Transaction for 100.scope/start is destructive (shutdown.target has 'start' job queued)
```

The correct pattern: start with a no-op (`/bin/true`), stay "active" during normal operation, and run the cleanup work in `ExecStop` when systemd naturally tears down services during shutdown. The service becomes a normal active service during runtime and participates in the standard shutdown ordering rather than fighting against it.

`/usr/local/bin/egpu-detach.sh`:

```bash
#!/bin/bash
LOGFILE=/var/log/egpu-attach.log
echo "[$(date)] eGPU detach triggered (pre-shutdown)" >> $LOGFILE

for vendor_dev in "1002:687f" "1002:aaf8"; do
    vendor=$(echo $vendor_dev | cut -d: -f1)
    device=$(echo $vendor_dev | cut -d: -f2)
    for syspath in /sys/bus/pci/devices/*/; do
        v=$(cat ${syspath}vendor 2>/dev/null | sed 's/0x//')
        d=$(cat ${syspath}device 2>/dev/null | sed 's/0x//')
        if [ "$v" = "$vendor" ] && [ "$d" = "$device" ]; then
            bdf=$(basename $syspath)
            echo "$bdf" > /sys/bus/pci/devices/${bdf}/driver/unbind 2>/dev/null
            echo "[$(date)] Unbound $bdf from vfio-pci" >> $LOGFILE
        fi
    done
done

for dev in /sys/bus/thunderbolt/devices/*/; do
    name=$(cat ${dev}device_name 2>/dev/null)
    auth=$(cat ${dev}authorized 2>/dev/null)
    if [ "$name" = "eGPU Pro" ] && [ "$auth" = "1" ]; then
        echo 0 > ${dev}authorized 2>/dev/null
        echo "[$(date)] De-authorized $(basename $dev)" >> $LOGFILE
    fi
done

echo "[$(date)] eGPU detach complete" >> $LOGFILE
```

```bash
chmod +x /usr/local/bin/egpu-detach.sh
```

### Enable Everything

```bash
systemctl daemon-reload
systemctl enable egpu-boot-auth.service egpu-shutdown.service
systemctl start egpu-shutdown.service
udevadm control --reload-rules
```

---

## Phase 6: The NIC Renaming Problem

**Failure mode**: After connecting the eGPU and rebooting, the Proxmox host loses network connectivity. The bridge `vmbr0` becomes orphaned because its member interface (`enp86s0`) no longer exists.

**Why**: Linux predictable network interface naming derives the name partly from the device's PCI path. The Intel I226-LM NIC sits on a PCIe slot with a fixed physical location, but its bus number shifts when the Thunderbolt eGPU connection introduces additional PCIe bridges into the topology. Before eGPU: bus 86. After eGPU: bus 88. The naming algorithm sees a different path and assigns a different name. The interface itself is identical -- same hardware, same MAC address -- but systemd-udev renames it on every boot based on the current PCI topology.

**Fix**: Pin the interface to a stable name using its MAC address.

```bash
# Identify the MAC address (run while the eGPU is connected)
ip link show enp88s0
# example: link/ether 88:ae:dd:72:13:25 brd ff:ff:ff:ff:ff:ff
```

`/etc/udev/rules.d/10-stable-nic.rules`:

```
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="88:ae:dd:72:13:25", NAME="lan0"
```

Using `lan0` as the stable name makes it obvious this is a pinned interface rather than an auto-named one.

Update `/etc/network/interfaces` to use the new name:

```
iface lan0 inet manual

auto vmbr0
iface vmbr0 inet static
  address 192.168.6.69/22
  gateway 192.168.4.1
  bridge-ports lan0
  bridge-stp off
  bridge-fd 0
```

After this, reload udev and test with a reboot:

```bash
udevadm control --reload-rules
udevadm trigger
```

The NIC will now enumerate as `lan0` regardless of whether the eGPU is connected or not.

---

## Phase 7: AMD Vega Reset Bug and vendor-reset

**Failure mode**: Starting VM 100 with the Vega 56 passed through crashed the Proxmox host. The host became completely unresponsive and required a hard reboot. Before the crash, dmesg showed:

```
error writing '1' to '/sys/bus/pci/devices/0000:30:00.0/reset': Inappropriate ioctl for device
failed to reset PCI device '0000:30:00.0', but trying to continue
org.freedesktop.DBus.Error.NoReply: Did not receive a reply.
```

**Why**: QEMU requires that a PCIe device support Function Level Reset (FLR) before it can be passed through to a guest. FLR allows QEMU to bring the device to a known state before the guest starts and to safely reset it when the guest stops. AMD Vega 10 (the architecture behind Vega 56) does not implement FLR in hardware. The kernel attempts the reset via `ioctl`, gets `EOPNOTSUPP`, and QEMU either crashes or hangs the host when it tries to continue with an unresettable device.

**Fix**: `vendor-reset` is a DKMS kernel module written by Geoffrey McRae (gnif) that implements custom GPU reset sequences for AMD GPUs that lack FLR support. For Vega 10, it uses AMD BACO (Bus Active, Chip Off) power sequencing to properly reset the GPU state. It hooks into the kernel's PCIe reset path so QEMU's reset call gets intercepted and handled correctly.

### Building vendor-reset on Kernel 6.17

The upstream `vendor-reset` repo has not been updated for Linux kernel 6.x. Two patches are required before it will compile.

**Patch 1 -- Missing function prototypes** (kernel 6.x promotes `-Wmissing-prototypes` to an error):

```bash
echo 'ccflags-y += -Wno-missing-prototypes' >> /usr/src/vendor-reset/src/Makefile
```

**Patch 2 -- Moved header** (`asm/unaligned.h` moved to `linux/unaligned.h` in kernel 6.1):

```bash
sed -i 's|asm/unaligned.h|linux/unaligned.h|g' \
  $(grep -rl 'asm/unaligned.h' /usr/src/vendor-reset/)
```

Full build sequence:

```bash
apt-get install -y pve-headers-$(uname -r) dkms git build-essential
cd /usr/src
git clone https://github.com/gnif/vendor-reset.git
cd vendor-reset
echo 'ccflags-y += -Wno-missing-prototypes' >> src/Makefile
sed -i 's|asm/unaligned.h|linux/unaligned.h|g' $(grep -rl 'asm/unaligned.h' .)
dkms install .
```

If `dkms install` fails, check `/var/lib/dkms/vendor-reset/<version>/<kernel>/build/make.log`.

### Secure Boot

If Secure Boot is enabled (common on NUC hardware), the unsigned DKMS module will be blocked from loading. Enroll a Machine Owner Key (MOK):

```bash
mokutil --import /var/lib/dkms/mok.pub
# Enter a one-time password when prompted
```

Reboot. The UEFI shows a blue "Shim UEFI key management" screen before Proxmox boots. Select: **Enroll MOK** > **Continue** > **Yes** > enter the password > **Reboot**.

After enrollment, DKMS-built modules signed with that key load without issue on all future boots.

### Load Order

`vendor-reset` must be listed before `vfio_pci` in the module load file. When `vfio-pci` claims a device, `vendor-reset` needs to already be active in the kernel to intercept subsequent reset calls.

```
# /etc/modules-load.d/vfio.conf
vendor-reset
vfio
vfio_iommu_type1
vfio_pci
```

Verify it's loaded:

```bash
lsmod | grep vendor_reset
# vendor_reset           32768  0
```

---

## Phase 8: VM Configuration

VM 100 is already configured as an Ubuntu 24.04 server. These are the additional parameters needed for GPU passthrough.

```bash
qm set 100 -hostpci0 0000:30:00.0,pcie=1,rombar=1
qm set 100 -hostpci1 0000:30:00.1,pcie=1,rombar=0
```

### Requirements

**Machine type must be q35** -- the i440fx machine type (Proxmox default for older VMs) does not support PCIe passthrough. q35 emulates an Intel ICH9 chipset with PCIe slots.

```bash
qm set 100 -machine q35
```

**CPU must be `host`** -- exposes the host CPU's actual feature flags to the guest, which GPU drivers often require.

```bash
qm set 100 -cpu host
```

**`pcie=1`** on both devices -- use PCIe semantics rather than PCI for the host-mapped device slots.

**`rombar=1`** on GPU, **`rombar=0`** on HDMI audio -- the audio function (`30:00.1`) has no ROM bar in its PCIe config space. Setting `rombar=1` on a device that doesn't have one causes guest boot issues.

### The Non-Fatal Audio Warning

When starting the VM, you will see:

```
kvm: vfio: Cannot reset device 0000:30:00.1, no available reset mechanism.
```

This is expected and harmless. `vendor-reset` provides a reset sequence for the GPU function (`30:00.0`) but not for the HDMI audio function. The audio function has no FLR and no vendor-specific reset path. QEMU logs the warning and continues. The VM starts and operates correctly.

---

## Phase 9: ROCm Deprecation and the Vulkan Solution

This phase involves the most subtle failure in the project: everything works at the hardware and kernel level, but the software stack targeting the GPU is broken for a completely different reason.

### Guest Setup (Ubuntu 24.04)

The Ubuntu 24.04 base kernel doesn't include `amdgpu`. It's in the extras package:

```bash
apt-get install -y linux-modules-extra-$(uname -r)
apt-get install -y linux-firmware
modprobe amdgpu
```

Verify the GPU is recognized:

```bash
ls /dev/dri/
# card0  renderD128
dmesg | grep -i amdgpu | head -5
```

Persist across reboots:

```bash
echo 'amdgpu' > /etc/modules-load.d/amdgpu.conf
```

### The ROCm Problem

Ollama detects the Vega 56 via HSA (AMD's Heterogeneous System Architecture runtime) and attempts to use its bundled ROCm compute runner. It crashes immediately:

```
failure during GPU discovery error="runner crashed"
```

Confirming the failure directly:

```bash
LD_LIBRARY_PATH=/usr/local/lib/ollama/rocm /usr/local/lib/ollama/rocm/libggml-hip.so
# Segmentation fault
```

**Why ROCm fails on Vega 56**:

AMD Vega 10 (`gfx900`) was officially deprecated in **ROCm 6.0** (released early 2024) and removed from compiled kernel library support. Ollama 0.18.x bundles **ROCm 7.2**. The bundled `libhipblas`, `librocblas`, and `libamdhip64` libraries no longer contain pre-compiled GPU kernels for the `gfx900` target. When the ROCm runner initializes and attempts to load kernels for the detected GPU architecture, it segfaults because the compiled kernel code for gfx900 is absent.

**ROCm support history for Vega 10 (gfx900)**:

| Version | Status |
|---------|--------|
| ROCm 5.x | Full gfx900 support |
| ROCm 6.0 | gfx900 deprecated, removed from compiled libraries |
| ROCm 7.x | gfx900 not present, segfault on init |
| Ollama 0.18+ | Bundles ROCm 7.x -- affected |

### Why `HSA_OVERRIDE_GFX_VERSION` Does Not Fix This

The common workaround `HSA_OVERRIDE_GFX_VERSION=9.0.0` does not help. That variable instructs the HSA runtime to report a specific GFX version string. It's designed to spoof a newer GPU as an older compatible one -- for example, making an RDNA3 card identify as RDNA2 to use RDNA2-compiled kernels. For Vega 10, `gfx900` is already the GPU's actual version. The problem is not a version string mismatch. The problem is that compiled kernel libraries for gfx900 don't exist in ROCm 7.x. Overriding the version string changes nothing about what kernel code is available.

### The Vulkan Solution

Ollama includes a Vulkan GPU compute backend alongside ROCm. Vulkan uses Mesa's RADV driver, which is maintained by the open source community and supports all GCN and RDNA generations including Vega 10, independent of AMD's ROCm lifecycle decisions. Vulkan compute doesn't rely on pre-compiled GPU kernels -- it compiles shaders at runtime via SPIR-V intermediate representation.

Install Vulkan support in the guest:

```bash
apt-get install -y vulkan-tools libvulkan1 mesa-vulkan-drivers
```

Verify the GPU is visible:

```bash
vulkaninfo 2>&1 | grep deviceName
# deviceName = AMD Radeon RX Vega (RADV VEGA10)
```

Enable the Vulkan backend in Ollama via a systemd override:

```bash
mkdir -p /etc/systemd/system/ollama.service.d
cat > /etc/systemd/system/ollama.service.d/rocm.conf << 'EOF'
[Service]
Environment=OLLAMA_VULKAN=1
EOF
systemctl daemon-reload && systemctl restart ollama
```

Also configure ldconfig so Ollama's bundled shared libraries resolve correctly:

```bash
echo '/usr/local/lib/ollama/rocm' > /etc/ld.so.conf.d/ollama-rocm.conf
echo '/usr/local/lib/ollama' > /etc/ld.so.conf.d/ollama.conf
ldconfig
```

### Verification

Check Ollama's startup logs:

```bash
journalctl -u ollama | grep -E "Vulkan|inference compute|vram"
```

Expected output:

```
inference compute  library=Vulkan  name=Vulkan0
description="AMD Radeon RX Vega (RADV VEGA10)"
type=discrete  total="8.0 GiB"  available="8.0 GiB"
```

### Result

After switching to the Vulkan backend: **85 tokens/second on llama3.2**. CPU-only inference on this VM runs at approximately 6-8 tokens/second. The Vulkan backend delivers roughly 10-15x the throughput.

### Why Vulkan Works When ROCm Does Not

- RADV (Mesa's Vulkan AMD driver) is community-maintained and covers all GCN/RDNA generations. Its support for Vega 10 is not tied to AMD's ROCm decisions.
- Vulkan compute uses SPIR-V shaders compiled at runtime. There's no "pre-compiled kernel library for gfx900" to go missing.
- The tradeoff: Vulkan compute is generally slightly less optimized than ROCm for heavy fp16/bf16 workloads, because ROCm tuned libraries (rocBLAS, hipBLAS) use hand-optimized assembly for specific architectures. For LLM inference the practical difference is modest.

---

## Verification Commands

### Host-side

```bash
# GPU bound to vfio-pci
lspci -nnk | grep -A2 "1002:687f\|1002:aaf8"
# expected: Kernel driver in use: vfio-pci

# TB authorization state
cat /sys/bus/thunderbolt/devices/1-1/authorized
# expected: 1

# Watch attach/detach log
tail -f /var/log/egpu-attach.log

# vendor-reset loaded
lsmod | grep vendor_reset

# IOMMU groups after hot-plug
find /sys/kernel/iommu_groups/ -type l | sort -V | grep "2[cdef]:\|3[01]:"
```

### Guest-side

```bash
# GPU visible in guest
lspci | grep AMD

# DRM devices created
ls /dev/dri/

# Vulkan driver sees GPU
vulkaninfo 2>&1 | grep deviceName

# Ollama using GPU
journalctl -u ollama | grep -E "vram|Vulkan|inference compute"
```

---

## Troubleshooting

### eGPU not appearing in `lspci` after plug-in

1. Check the log: `tail /var/log/egpu-attach.log`
2. Verify TB device: `ls /sys/bus/thunderbolt/devices/`
3. Check authorization: `cat /sys/bus/thunderbolt/devices/1-1/authorized`
4. If `0`, authorize manually: `echo 1 > /sys/bus/thunderbolt/devices/1-1/authorized`
5. If no TB device appears at all, the cable is USB-C only -- replace with a certified TB3/TB4 cable

### Boot hang with eGPU connected

This should not occur in normal operation -- `egpu-boot-auth.service` handles boot-with-GPU-connected and `egpu-shutdown.service` ensures a clean state before reboot. If it happens anyway:

1. Check if the shutdown service ran: `journalctl -u egpu-shutdown --since yesterday`
2. If it didn't de-authorize, PCIe was active during the previous reboot
3. Verify both services are active: `systemctl is-active egpu-shutdown.service egpu-boot-auth.service`
4. Check the egpu-shutdown unit file for the correct `ExecStart=/bin/true` + `ExecStop` pattern
5. Emergency recovery if services are broken: boot without eGPU, hot-plug after boot, then debug the service

### Network down after reboot with eGPU

NIC renamed due to PCI bus shift. Temporary recovery:

```bash
ip link set enp88s0 master vmbr0
```

Permanent fix: apply the udev stable naming rule in Phase 6.

### VM start fails with destructive transaction error

The `egpu-shutdown.service` unit file has wrong structure. Verify:
- `ExecStart=/bin/true` (not the detach script)
- `ExecStop=/usr/local/bin/egpu-detach.sh`
- `RemainAfterExit=yes`
- `WantedBy=multi-user.target` (not `shutdown.target`)

### ROCm segfaults or Ollama shows `total_vram=0`

Expected with Vega 10 on Ollama 0.18+. ROCm 7.x dropped gfx900. Switch to Vulkan:

1. Add `Environment=OLLAMA_VULKAN=1` to the systemd override
2. Install `mesa-vulkan-drivers` in the guest
3. Confirm with `vulkaninfo 2>&1 | grep deviceName`

Do not try `HSA_OVERRIDE_GFX_VERSION`. It will not fix this.

### vendor-reset fails to build

On kernel 6.x, both patches are required:
1. `ccflags-y += -Wno-missing-prototypes` in `src/Makefile`
2. Replace `asm/unaligned.h` with `linux/unaligned.h` throughout source

If Secure Boot is enabled, the unsigned module will silently fail to load. Check `dmesg | grep vendor` and enroll the MOK key.

### `amdgpu` module not found in guest

```bash
apt-get install -y linux-modules-extra-$(uname -r)
apt-get install -y linux-firmware
modprobe amdgpu
```

---

## Known Issues / Deprecated

### ROCm 7.x: Vega 10 (gfx900) support removed

AMD dropped Vega 10 from ROCm in version 6.0. Ollama 0.18+ bundles ROCm 7.x. Any workflow relying on ROCm for Vega 56 inference is broken on current Ollama. The fix is the Vulkan backend (`OLLAMA_VULKAN=1`), not a downgrade of Ollama.

### vendor-reset: unmaintained against kernel 6.x

The `gnif/vendor-reset` repository has not been updated for Linux kernel 6.x API changes. The two patches in Phase 7 are required. If those patches stop working against a future kernel version, the reset mechanism will fail and VM start will crash the host again.

### Thunderbolt PCIe tunneling and boot ordering

All three connection scenarios are fully supported by the Phase 5 service stack:

- **Hot-plug**: udev fires `egpu-attach.sh`, which authorizes the TB tunnel and lets `vfio-pci` claim the device
- **Boot with GPU connected**: `egpu-boot-auth.service` runs after `multi-user.target` and authorizes any connected device
- **Reboot with GPU connected**: `egpu-shutdown.service` de-authorizes the TB device before the machine goes down; `egpu-boot-auth.service` re-authorizes it on the next boot

Thunderbolt PCIe tunneling is fundamentally dynamic -- authorization must happen in userspace after boot. The services above close this gap. A future kernel with automated TB authorization policy could simplify the implementation, but the current stack is complete.

### Audio function reset warning

The HDMI audio function (`30:00.1`) cannot be reset between VM starts. If the audio function accumulates bad state, the only recovery is a full de-authorization/re-authorization cycle of the TB device. In practice this has not caused problems, but it's a known limitation.

### Secure Boot and DKMS

DKMS modules require MOK enrollment on Secure Boot systems. If Proxmox is updated to a new kernel, DKMS rebuilds `vendor-reset` automatically and signs the new build with the same enrolled key. Re-enrollment is not required unless the key is rotated.

---

## Changelog

| Date | Entry |
|------|-------|
| 2026-03-18 | Initial implementation: TB4 authorization, vfio-pci, boot/shutdown services, vendor-reset (patched for kernel 6.17), NIC stable naming, VM 100 GPU passthrough, Vulkan backend for Ollama (ROCm 7.x drops gfx900). Achieved 85 tok/s on llama3.2. |
