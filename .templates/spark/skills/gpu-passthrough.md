---
summary: User asks about GPU passthrough, VFIO, IOMMU, or passing GPUs to VMs
---

# GPU Passthrough

## When to use

The user asks about GPU passthrough, VFIO, IOMMU, or passing GPUs to VMs.

## Prerequisite

If `proxman --version` fails, run `npm install -g @proxman-io/cli@0.2.1`.

## Two-tier model

GPU passthrough touches two surfaces:

1. **Proxmox API** (covered by `proxman`) — VM config edits to attach a PCI device.
2. **Host-level kernel/boot/modules** (NOT in the API) — IOMMU enable, VFIO driver bind, host-driver blacklist. Requires SSH to the Proxmox host using your agent-managed SSH key (see `ssh-keys` skill).

## Step 1 — Host-level prereqs (SSH to the Proxmox host)

These are read-only diagnostics first, then config edits behind explicit user confirmation.

Diagnostics (safe):

```bash
ssh root@<node> 'dmesg | grep -i iommu | head'
ssh root@<node> 'cat /proc/cmdline | grep iommu'
ssh root@<node> 'find /sys/kernel/iommu_groups/ -type l | sort -V'
ssh root@<node> 'lspci -nn | grep -iE "nvidia|amd|radeon|vga"'
```

Configuration (only with explicit user confirmation, full impact statement, and a stated rollback path; reboot is required for changes to take effect):

- Kernel command line: edit `/etc/default/grub` → `GRUB_CMDLINE_LINUX_DEFAULT` adds `intel_iommu=on iommu=pt` (Intel) or `amd_iommu=on iommu=pt` (AMD) → `update-grub`.
- VFIO modules: append to `/etc/modules`:

  ```
  vfio
  vfio_iommu_type1
  vfio_pci
  ```

- VFIO bind: `/etc/modprobe.d/vfio.conf` → `options vfio-pci ids=<vendor:device>` (e.g. `10de:2684,10de:22ba` for an NVIDIA card).
- Blacklist host driver: `/etc/modprobe.d/blacklist.conf` → `blacklist nouveau` and `blacklist nvidia` (NVIDIA case).

## Step 2 — VM config (proxman)

```bash
proxman vm config-set --vmid <id> --hostpci0 "<pci-id>,pcie=1,x-vga=1" --json
```

(Confirm exact flag names via `proxman vm --help`.)

To inspect existing GPU assignments, look for `hostpci*` fields in `proxman vm config --vmid <id> --json`.

## Safety

- IOMMU groups are atomic — every device in a group passes through together. Verify before binding.
- GPU passthrough requires a host reboot. Schedule with the user.
- The host loses access to the passed-through GPU (no console video on it).
- Some consumer NVIDIA cards trigger Code 43 in Windows VMs — vendor-id hiding required.
- For multi-GPU systems, verify each GPU is in a separate IOMMU group.
