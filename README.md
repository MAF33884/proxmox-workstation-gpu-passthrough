# Single‑Box Workstation in Proxmox with GPU Passthrough

> **Turn any Proxmox host into a Windows daily‑driver desktop with near‑native GPU performance.**

---

## Overview
This guide walks through converting a headless Proxmox VE server into a full Windows 10/11 workstation VM using PCIe GPU passthrough.  
You’ll get:  

* Native graphics via an isolated GPU (e.g. Quadro M4000, Tesla P4)  
* USB input passthrough for keyboard / mouse / audio  
* Auto‑start, no‑sleep power profile – perfect for “Play On Home” or everyday use

---

## Hardware Checklist
| Part | Requirement |
|------|-------------|
| **CPU** | Intel VT‑d / AMD‑Vi enabled |
| **Motherboard** | UEFI + IOMMU toggle |
| **GPUs** | • Host GPU (iGPU or spare card)<br>• Passthrough GPU |
| **Storage** | SSD/LVM thin‑pool for VM disk |
| **USB** | Optional external hub/switch for inputs |

---

## 1 · Enable IOMMU
```bash
# GRUB ( Intel )
sudo nano /etc/default/grub
# add inside GRUB_CMDLINE_LINUX_DEFAULT
intel_iommu=on iommu=pt

sudo update-grub && sudo reboot
```
*(replace with `amd_iommu=on` on AMD)*

---

## 2 · Bind the GPU to vfio‑pci
```bash
# get device IDs
lspci -nn | grep -i nvidia
# example IDs 10de:13f1 10de:0fbb
echo "options vfio-pci ids=10de:13f1,10de:0fbb" |   sudo tee /etc/modprobe.d/vfio.conf
sudo update-initramfs -u -k all && sudo reboot
```

---

## 3 · Create VM (Example ID 103)
```ini
# /etc/pve/qemu-server/103.conf
bios: ovmf
machine: pc-q35-9.2+pve1
cpu: host,flags=+aes;+ssbd;+hv-tlbflush;+hv-evmcs
args: -cpu host,kvm=on
memory: 8192
sockets: 1
cores: 4
vga: std            # temporary VNC
hostpci0: 0000:04:00.0,pcie=1,x-vga=1,rombar=0
hostpci1: 0000:04:00.1
boot: order=virtio0;ide2
ide2: local:iso/Win10.iso,media=cdrom
ide0: local:iso/virtio-win.iso,media=cdrom
virtio0: local-lvm:vm-103-disk-0,size=96G,iothread=1
net0: virtio=DE:AD:BE:EF:00:01,bridge=vmbr0
agent: 1
startup: order=1
onboot: 1
```
> Remove `vga: std` after NVIDIA drivers are installed.

---

## 4 · Install Windows + VirtIO drivers
1. Boot installer → load **`viostor\w10md64`** for disk  
2. After first login, mount VirtIO ISO again  
3. Install **NetKVM** (`NetKVM\w10md64`) for network

---

## 5 · Disable Secure Boot
Press `ESC` in OVMF → *Device Manager → Secure Boot* → **Disable**

---

## 6 · Install NVIDIA Driver
Use official Quadro/GeForce package or extract `Display.Driver` and update through Device Manager if “hardware not found”.

---

## 7 · USB Passthrough (whole controller)
```ini
# add to VM conf
hostpci2: 0000:00:14.0,pcie=1
```
*(replace with your USB controller PCI ID)*

---

## 8 · Power Profile (no‑sleep)
```powershell
powercfg -change -standby-timeout-ac 0
powercfg -change -monitor-timeout-ac 0
powercfg -hibernate off
```

---

## Troubleshooting
| Symptom | Fix |
|---------|-----|
| Black screen after driver | add `rombar=0` or inject vBIOS ROM |
| “No NVIDIA hardware” | ensure `x‑vga=1`, Secure Boot off, use Device Mgr `.inf` |
| 169.254.x.x IP | install VirtIO Net driver, verify `vmbr0` bridge |

---

## Snapshot
After first clean boot with drivers + USB working:
```bash
qm snapshot 103 "golden_gpu_vm"
```

---

## License
MIT – do whatever you like, just share improvements!
