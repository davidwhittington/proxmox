# proxmox

Personal notes and documentation from running [Proxmox VE](https://www.proxmox.com/) on an ASUS NUC 14 PRO (Intel Meteor Lake, Core Ultra 7 165H) with a Blackmagic eGPU Pro (AMD Radeon RX Vega 56, 8GB HBM2).

This repo exists because the combination of Thunderbolt 4 passthrough, AMD Vega 10's missing FLR support, ROCm 7.x dropping gfx900, and the kernel 6.17 vendor-reset build failures produces a failure chain that is poorly documented anywhere in one place. These guides cover every mistake made along the way, with the exact commands and configurations that resolved each one.

The end goal was GPU-accelerated LLM inference via Ollama in a KVM VM. It works — 85 tokens/sec on llama3.2 via the Vulkan backend, versus ~6-8 tok/s CPU-only.

---

## Guides

### Deploying Ubuntu 24.04 via Cloud-Init

**[Read the guide →](https://davidwhittington.github.io/proxmox/ubuntu-cloudinit-vm.html)**

Fully automated Ubuntu 24.04 LTS VM provisioning using the noble cloud image — no interactive installer. Covers `qm create`, disk import and resize, SSH key injection via cloud-init, DHCP discovery, and SSH alias setup. Boots to a ready system in under 2 minutes.

Raw Markdown: [`docs/ubuntu-cloudinit-vm.md`](docs/ubuntu-cloudinit-vm.md)

---

### Local LLM + RAG Pipeline

**[Read the guide →](https://davidwhittington.github.io/ollama/)**

Deploying Ollama on the Ubuntu VM, exposing the API to the local network, standing up Open WebUI and Chroma via Docker, and building a private RAG ingestion pipeline that indexes local files. Fully local — nothing leaves the network.

Documented in: [`davidwhittington/ollama`](https://github.com/davidwhittington/ollama)

---

### TB4 eGPU Passthrough — Blackmagic eGPU Pro + Proxmox VE

**[Read the guide →](https://davidwhittington.github.io/proxmox/proxmox-egpu.html)**

Covers the full passthrough stack for a Thunderbolt 4 external GPU:

- IOMMU group verification
- TB4 cable requirements (this one wastes time if you skip it)
- Thunderbolt authorization model and why it hangs at boot
- vfio-pci configuration without softdeps
- Boot/reboot/hot-plug scenarios — all three fully handled via systemd services
- NIC renaming when the eGPU shifts PCI bus numbers
- AMD Vega FLR bug and vendor-reset DKMS (with kernel 6.17 patches)
- Secure Boot MOK enrollment for DKMS modules
- ROCm 7.x deprecation of gfx900 and the Vulkan/RADV fallback for Ollama

Raw Markdown: [`docs/proxmox-egpu-spec.md`](docs/proxmox-egpu-spec.md)

---

## Hardware

| Component | Detail |
|-----------|--------|
| Host | ASUS NUC 14 PRO, Intel Meteor Lake (Core Ultra 7 165H) |
| Proxmox | VE 9.1.6, kernel 6.17.13-2-pve |
| eGPU | Blackmagic eGPU Pro — AMD Radeon RX Vega 56, 8GB HBM2 |
| TB controller | Intel JHL7440 TB3 Bridge (Titan Ridge) — TB3 device on TB4 host |
| Target VM | VM 100 "ollama", Ubuntu 24.04, 6 vCPU, 16GB RAM |

## Status

Fully working as of 2026-03-18. Ollama reports 8GB VRAM and runs llama3.2 at ~85 tok/s via the Vulkan backend.

## Changelog

| Date | Entry |
|------|-------|
| 2026-03-18 | Initial commit: TB4 eGPU passthrough guide + HTML reference page |
| 2026-03-18 | Add Ubuntu 24.04 cloud-init VM guide and Ollama RAG pipeline link |
