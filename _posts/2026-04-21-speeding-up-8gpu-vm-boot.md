---
layout: post
title: "Speeding Up 8-GPU VM Boot: Kernel 6.19, Hugepage Preallocation, and PCIe BAR Mapping"
date: 2025-04-21 00:00:00 +0000
categories: gpu infra performance
tags: qemu kvm hugepages pcie vfio dgx h200 kernel
---

## Introduction

Booting a virtual machine with 8 NVIDIA H200 GPUs, 8 InfiniBand NICs, 1 TiB of hugepage-backed RAM, and 192 vCPUs is not a trivial operation. On a DGX H200 / HGX H200 system, an untuned 8-GPU VM can take **5+ minutes** to become ready — most of that time spent before the guest kernel even prints its first message.

This article breaks down where boot time is spent, what kernel 6.19 brings to the table, and practical techniques that reduced our 8-GPU VM boot from over 5 minutes to under 30 seconds.

---

## Where Does Boot Time Go?

We instrumented every phase of VM startup using QEMU wrap scripts, `dmesg` timestamps, and `perf` profiling. Here's the breakdown for an 8-GPU VM with 768 GiB of 1 GiB hugepages:

| Phase | Time (untuned) | Time (tuned) | What happens |
|-------|---------------|--------------|--------------|
| Hugepage preallocation | 60-90s | 5-8s | QEMU touches every page (`prealloc=on`) |
| VFIO DMA mapping | 30-120s | 10-15s | Host IOMMU maps guest memory for each device |
| SeaBIOS PCI BAR programming | 10-60s (or hang) | 2-3s | Firmware assigns MMIO windows to PCIe devices |
| OVMF/UEFI + GRUB firmware chain | 5-14s | ~0.1s | Direct kernel boot bypasses OVMF and GRUB entirely |
| Guest vIOMMU DMAR table build | 60-180s | 0s | Don't use guest vIOMMU for passthrough |
| Guest kernel boot | 3-5s | 1-2s | Kernel init, module loading |
| NVIDIA driver probe | 2-5s | 2-3s | `nvidia.ko` scans for GPUs |
| systemd + kata-agent | 2-3s | 1-2s | Userspace init |
| **Total** | **~180-480s** | **~25-35s** |  |

The top three bottlenecks — hugepage preallocation, VFIO DMA mapping, and PCI BAR programming — account for 90%+ of boot time. Let's attack each one.

---

## Bottleneck 1: Hugepage Preallocation

### The Problem

When QEMU starts with `memory-backend-file,prealloc=on,mem-path=/dev/hugepages`, it must **touch every page** to ensure the hugepages are actually allocated from the kernel's pool. For 768 GiB of 1 GiB hugepages, that's 768 `mmap` + `memset` operations, each touching 1 GiB of memory.

By default, QEMU does this **serially on a single thread**. At ~1 GiB/s per page fault + clear, that's 60-90 seconds just for memory setup.

### The Fix: `prealloc-threads`

QEMU 6.0+ supports `prealloc-threads=N` on the memory backend object:

```
-object memory-backend-file,id=dimm1,size=768G,mem-path=/dev/hugepages,share=on,prealloc=on,prealloc-threads=8
```

With 8 threads, each handles ~96 GiB in parallel. Boot allocation drops from 60-90s to **5-8s**.

**For libvirt**, the `<allocation>` element in `<memoryBacking>` controls this:

```xml
<memoryBacking>
  <hugepages/>
  <source type="file"/>
  <access mode="shared"/>
  <allocation mode="immediate" threads="32"/>
</memoryBacking>
```

### NUMA-Aware Allocation

On a 2-socket DGX with 880 hugepages per NUMA node, a single 768 GiB allocation on one NUMA node will fail (768 > 880, and QEMU may try to allocate from one node first). Split memory across NUMA:

```xml
<cpu>
  <numa>
    <cell id="0" cpus="0-95" memory="384G" unit="GiB"/>
    <cell id="1" cpus="96-191" memory="384G" unit="GiB"/>
  </numa>
</cpu>
<numatune>
  <memnode cellid="0" mode="strict" nodeset="0"/>
  <memnode cellid="1" mode="strict" nodeset="1"/>
</numatune>
```

This ensures each NUMA node allocates only 384 GiB from its local pool.

### Memory Alignment

**Critical:** With 1 GiB hugepages, the VM memory size must be a **multiple of 1024 MiB**. A value like 1400000 MiB (1367.19 GiB) is NOT aligned and causes allocation failures or rounding overhead. Always use exact GiB values:

```
768 GiB = 786432 MiB  ✓
512 GiB = 524288 MiB  ✓
1400000 MiB           ✗ (not 1024-aligned)
```

---

## Bottleneck 2: VFIO IOMMU DMA Mapping

### The Problem

When QEMU opens a VFIO device and maps guest memory, the kernel's VFIO type1 IOMMU backend must:

1. **Pin every guest page** in host physical memory (prevent swapping)
2. **Create IOMMU page table entries** for each page, for each device

With 768 GiB of memory and 16 VFIO devices (8 GPUs + 8 IB NICs), the kernel creates:

```
768 GiB ÷ 1 GiB per page × 16 devices = 12,288 IOMMU mappings
```

But with 4 KiB IOMMU page tables (the IOMMU doesn't use 1 GiB pages internally on all platforms), it's actually:

```
768 GiB ÷ 4 KiB × 16 devices = ~3.2 billion page table entries
```

This can take 30-120 seconds and **locks kernel data structures**, making the entire host unresponsive during VM boot.

### The Fixes

#### 1. Reduce VM Memory

GPU VRAM (141 GiB × 8 = 1128 GiB) lives on the GPU cards, NOT in host RAM. Most GPU workloads need 64-256 GiB of host RAM for CPU-side tensors and driver buffers:

```xml
<!-- Before: 960 GiB (overkill) -->
<memory unit="GiB">960</memory>

<!-- After: 256 GiB (sufficient for most training jobs) -->
<memory unit="GiB">256</memory>
```

This alone reduces IOMMU mapping time by 4x.

#### 2. Kernel 6.19: `iommufd` Backend

The traditional VFIO type1 IOMMU backend pins all memory upfront. Kernel 6.19 introduces `iommufd` — a modern IOMMU backend that supports:

- **Lazy/on-demand mapping**: Pages are mapped only when the device actually accesses them
- **Batch operations**: Map thousands of pages in a single ioctl
- **Huge page awareness**: Uses 1 GiB IOMMU pages when hardware supports it

To use iommufd with QEMU 8.0+:

```
-object iommufd,id=iommufd0
-device vfio-pci,host=0000:1b:00.0,iommufd=iommufd0
```

Check if your kernel supports it:

```bash
ls /dev/iommu        # iommufd device
modprobe iommufd     # load module
```

#### 3. Raise VFIO DMA Entry Limit

The default `dma_entry_limit` (65535) may be too low for large VMs:

```bash
echo 2000000 > /sys/module/vfio_iommu_type1/parameters/dma_entry_limit
```

#### 4. Fewer VFIO Devices = Fewer Mappings

Each VFIO device requires its own set of IOMMU mappings. Reducing from 16 devices to 8 (PF passthrough instead of separate PF + VF) halves the mapping overhead.

---

## Bottleneck 3: SeaBIOS PCI BAR Programming

### The Problem

When QEMU starts, SeaBIOS (the default firmware for q35 machines) must program PCI Base Address Registers (BARs) for all devices. Each H200 GPU has a 256 GiB prefetchable BAR (BAR2) for GPU memory access.

With 16 PCIe root ports, each reserving 256 GiB of prefetchable MMIO:

```
16 ports × 256 GiB = 4 TiB of prefetchable MMIO space
```

SeaBIOS must find room in the guest's physical address space (limited by `host-phys-bits-limit=52` → 4 PiB) to map all these BARs. With 4 TiB of MMIO, SeaBIOS enters a slow iterative BAR assignment algorithm that can **hang for minutes** or fail entirely.

### Guest PCIe Topology

Inside the VM, `lspci -tv` reveals the topology that SeaBIOS (or OVMF) must program BARs for. Each VFIO-passthrough device sits behind a TI XIO3130 PCIe switch — this is QEMU's implementation of `pcie-root-port`:

```
lspci -tv (inside VM with device passthrough):

 +-[0000:64]-+-00.0  TI XIO3130 PCIe Switch (Upstream)
 |           +-00.0-[65-66]--+-00.0  TI XIO3130 (Downstream)
 |           |               └-01.0  TI XIO3130 (Downstream)
 |           +-66:00.0  NVIDIA H200 GPU
 |           └-67:00.0  Mellanox ConnectX-7 IB
```

Each pcie-root-port has its own **prefetchable 64-bit MMIO window** (pref64). The firmware must size these windows before devices can initialize:

- **GPU port** needs a 256 GiB pref64 window — the H200's BAR2 maps the full 141 GiB HBM plus control regions
- **IB port** needs only ~16 MiB — ConnectX-7 VFs have small BARs

With 16 root ports (8 GPU + 8 IB) and a naive "same size for all" approach, the firmware tries to allocate `16 × 256 GiB = 4 TiB` of MMIO space. This is what causes SeaBIOS to hang or spend minutes in its BAR assignment loop. The fix — split reserves per port type — gives the firmware a tractable address layout:

```
Ports 0-7  (GPU):  256G × 8 = 2048 GiB pref64
Ports 8-15 (IB):    16M × 8 =  128 MiB pref64
Total: ~2048 GiB (comfortably within phys-bits=52 limit)
```

### The Fix: Split Pref64 Reserves

Only GPU ports need 256 GiB pref64 (to map the H200's BAR2). IB NIC ports only need ~16 MiB. Assign reserves proportionally:

```
Ports 0-7 (GPUs):     pref64-reserve=256G  → 2048 GiB total
Ports 8-15 (IB NICs): pref64-reserve=16M   → 128 MiB total
Total: ~2048 GiB (within SeaBIOS capability)
```

For Kata Containers, this is patched in `genericAppendPCIeRootPort()`:

```go
pref64 := "256G"
memRes := "64M"
if i >= number/2 {
    pref64 = "16M"  // IB VF ports don't need large MMIO
    memRes = "1M"
}
```

For libvirt, QEMU auto-assigns based on the actual devices plugged in — this is less of an issue since libvirt doesn't pre-reserve.

### PCI Bridge Slot Conflicts

The q35 machine type places a `pci-bridge` at PCI address `addr=2`. If pcie-root-ports auto-assign to the same slot, QEMU silently fails or hangs. Ensure the bridge and root ports don't collide:

```xml
<!-- Bridge at high slot -->
<controller type="pci" index="1" model="pci-bridge">
  <address type="pci" domain="0x0000" bus="0x00" slot="0x1e" function="0x0"/>
</controller>
```

---

## Bottleneck 4: Direct Kernel Boot (Bypass OVMF/UEFI)

### The Problem

By default, libvirt boots VMs through a firmware chain:

```
OVMF/UEFI (2-5s) → GRUB (1-2s) → kernel load from disk (1-2s) → kernel init
```

OVMF is a full UEFI implementation that:
- Initializes virtual hardware
- Enumerates PCI devices and programs BARs
- Loads GRUB from the EFI system partition
- GRUB reads `grub.cfg`, loads the kernel + initrd from the virtual disk
- Only then does the Linux kernel start

For an 8-GPU VM with 16+ PCIe devices, OVMF's PCI enumeration alone takes 3-10 seconds. With large prefetchable BAR windows (256 GiB per GPU), OVMF's BAR assignment can take even longer than SeaBIOS.

### The Fix: `-kernel` and `-initrd` Direct Boot

QEMU can load the kernel and initrd **directly into guest memory**, bypassing OVMF/GRUB entirely:

```xml
<os>
  <type arch="x86_64" machine="pc-q35-10.2">hvm</type>
  <kernel>/boot/vmlinuz-6.8.0-107-generic</kernel>
  <initrd>/boot/initrd.img-6.8.0-107-generic</initrd>
  <cmdline>root=/dev/vda1 console=ttyS0 earlyprintk=ttyS0 mitigations=off audit=0 nowatchdog pci=noio selinux=0</cmdline>
  <boot dev="hd"/>
</os>
```

**No `<loader>` or `<nvram>` elements** — their absence tells QEMU to use SeaBIOS (lightweight) instead of OVMF (heavy).

### How It Works

With direct kernel boot:

```
QEMU loads kernel+initrd into guest RAM (instant, ~100ms)
  → SeaBIOS minimal PCI init (1-2s)
    → Linux kernel starts directly
      → mounts rootfs from /dev/vda1
        → systemd → ready
```

Without direct boot (OVMF path):

```
QEMU starts OVMF firmware (500ms)
  → OVMF PCI enumeration + BAR programming (3-10s for 16 devices)
    → OVMF loads GRUB from EFI partition (1-2s)
      → GRUB reads config, loads kernel from disk (1-2s)
        → Linux kernel starts
          → mounts rootfs
            → systemd → ready
```

### Time Savings

| Phase | OVMF Path | Direct Kernel Boot |
|-------|-----------|-------------------|
| Firmware init | 3-10s | 0s (no OVMF) |
| GRUB | 1-2s | 0s (no GRUB) |
| Kernel load | 1-2s from disk | ~100ms from RAM |
| **Total firmware overhead** | **5-14s** | **~0.1s** |

### The Trade-offs

| Feature | OVMF | Direct Kernel Boot |
|---------|------|-------------------|
| Secure Boot | Yes | No |
| EFI variables | Yes | No |
| Boot menu | Yes | No |
| Kernel selection | GRUB menu | Hardcoded in XML |
| Kernel upgrades | `apt upgrade` inside VM | Update XML `<kernel>` path on host |
| PCI BAR assignment | OVMF firmware | SeaBIOS (lighter) or kernel `pci=noio` |

### Kernel and Initrd Placement

The `<kernel>` and `<initrd>` paths are on the **host filesystem**, not the guest. The host must have the guest's kernel available:

```bash
# Option 1: Copy from guest image
virt-copy-out -d dgx-vm /boot/vmlinuz-6.8.0-107-generic /tmp/
virt-copy-out -d dgx-vm /boot/initrd.img-6.8.0-107-generic /tmp/

# Option 2: Mount guest image and copy
guestmount -a /var/lib/libvirt/images/dgx-vm.qcow2 -m /dev/sda1 /mnt
cp /mnt/boot/vmlinuz-6.8.0-107-generic /boot/
cp /mnt/boot/initrd.img-6.8.0-107-generic /boot/
umount /mnt

# Option 3: Share kernel via virtiofs (update path in XML)
# Keep kernels in /srv/tenant/boot/ and reference from XML
```

### Combining with `pci=noio`

The `pci=noio` kernel parameter tells Linux to skip PCI resource rebalancing — trusting the firmware's (SeaBIOS) BAR assignments. This avoids the kernel re-doing work that SeaBIOS already completed:

```xml
<cmdline>root=/dev/vda1 rw console=ttyS0 pci=noio mitigations=off audit=0 nowatchdog selinux=0</cmdline>
```

### Complete Example: Fastest Possible Boot Config

```xml
<os>
  <type arch="x86_64" machine="pc-q35-10.2">hvm</type>
  <!-- Direct kernel boot — no OVMF, no GRUB -->
  <kernel>/boot/vmlinuz-6.8.0-107-generic</kernel>
  <initrd>/boot/initrd.img-6.8.0-107-generic</initrd>
  <cmdline>root=/dev/vda1 rw console=ttyS0 earlyprintk=ttyS0 pci=noio mitigations=off audit=0 nowatchdog transparent_hugepage=always default_hugepagesz=1G hugepagesz=1G selinux=0 modprobe.blacklist=nouveau</cmdline>
  <!-- No <loader> or <nvram> = no OVMF -->
  <boot dev="hd"/>
</os>
```

---

## Bottleneck 5: Guest vIOMMU (Don't Use It)

### The Problem

Adding `-device intel-iommu,intremap=on,caching-mode=on` creates a virtual IOMMU inside the guest. With 768 GiB of guest RAM, the guest kernel spends **minutes** building DMAR (DMA Remapping) tables at boot.

### The Fix: Remove Guest vIOMMU

For VFIO passthrough, the **host** IOMMU provides DMA isolation. The guest doesn't need its own:

```xml
<!-- Remove or set to false -->
<iommu model='intel'>
  <!-- DELETE THIS -->
</iommu>
```

Or in TOML (Kata):

```toml
enable_iommu = false
```

Remove `intel_iommu=on` from guest kernel parameters too:

```
# Before
cmdline: ... intel_iommu=on iommu=pt ...

# After
cmdline: ... pci=noio mitigations=off ...
```

---

## Bottleneck 6: QEMU Debug Logging

### The Trap

QEMU's `-d int` flag (trace interrupts) is useful for debugging single-vCPU VMs but **completely deadlocks** a 192-vCPU VM. The synchronous logging can't keep up with the interrupt rate from 192 LAPICs + IOAPIC initialization.

We lost hours to this — QEMU appeared "stuck" with no console output, all vCPUs in `kvm_vcpu_block`. The real problem was the logging thread blocking the main event loop.

### Safe Debug Flags

```
# Safe for multi-vCPU VMs
-D /tmp/qemu-debug.log -d guest_errors,unimp,cpu_reset

# NEVER use on production VMs
-d int          # deadlocks with many vCPUs
-d mmio         # floods log, fills disk
```

---

## Kernel 6.19 Specific Improvements

### Why 6.19?

The DGX H200 ships with a custom 6.19 kernel (`6.19.0-dgx-custom`). Key features for GPU VM boot:

#### 1. iommufd Support

As described above, `iommufd` replaces the legacy VFIO type1 backend with on-demand IOMMU mapping. This is the single biggest boot time improvement for large-memory VMs with many VFIO devices.

#### 2. Improved Hugepage Allocation

Kernel 6.19 includes patches for faster `memfd_create` + `fallocate` for hugepage backends. The compound page handling is more efficient, reducing per-page fault overhead.

#### 3. PCIe Subsystem Improvements

- Faster ACS capability scanning
- Improved IOMMU group assignment
- Better handling of PCIe Gen 5 switch topologies (Broadcom PEX890xx)

#### 4. VFIO Improvements

- Batch DMA map/unmap operations
- Reduced lock contention during parallel device attachment
- Better error reporting for IOMMU mapping failures

### Building a Custom 6.19 Kernel

For the guest VM kernel, critical config options:

```
# GPU support
CONFIG_VFIO=m
CONFIG_VFIO_PCI=m
CONFIG_VFIO_IOMMU_TYPE1=m

# Hugepage support
CONFIG_HUGETLBFS=y
CONFIG_HUGETLB_PAGE=y
CONFIG_TRANSPARENT_HUGEPAGE=y

# NVLink/NVSwitch (needs NVIDIA out-of-tree modules)
# Just ensure PCI/PCIe support is comprehensive
CONFIG_PCI=y
CONFIG_PCIEPORTBUS=y
CONFIG_PCIE_DPC=y

# InfiniBand
CONFIG_INFINIBAND=m
CONFIG_INFINIBAND_IPOIB=m
CONFIG_MLX5_CORE=m
CONFIG_MLX5_INFINIBAND=m

# Performance
CONFIG_NO_HZ_FULL=y
CONFIG_HIGH_RES_TIMERS=y
CONFIG_PREEMPT_VOLUNTARY=y

# Disable unnecessary features (faster boot)
CONFIG_SECURITY_SELINUX=n
CONFIG_AUDIT=n
CONFIG_DEBUG_INFO_BTF=n
```

---

## The Optimized Boot Sequence

After applying all optimizations, here's the timeline for an 8-GPU VM:

```
t=0.0s   QEMU starts
t=0.1s   Memory backend created (prealloc-threads=8 starts touching pages)
t=5.0s   768 GiB hugepages allocated (8 threads, ~96 GiB each)
t=6.0s   KVM vCPUs created (192 vCPUs)
t=7.0s   VFIO devices opened, DMA mapping begins
t=12.0s  All 16 VFIO devices mapped (reduced memory = fewer pages)
t=13.0s  SeaBIOS PCI BAR programming (8 GPU ports × 256G + 8 IB ports × 16M)
t=15.0s  Guest kernel starts (direct -kernel boot, no BIOS chain)
t=16.0s  Kernel: PCI enumeration, module loading
t=18.0s  systemd starts
t=20.0s  nvidia.ko probes 8 GPUs (no fabricmanager needed for passthrough)
t=22.0s  mlx5_core probes 8 IB NICs
t=25.0s  kata-agent / cloud-init ready
t=25.0s  VM ready for workloads
```

**Total: ~25 seconds** from `virsh start` to SSH-able VM.

---

## Quick Reference: All Optimizations

```bash
# Host kernel parameters
intel_iommu=on iommu=pt pcie_acs_override=downstream,multifunction \
vfio_iommu_type1.allow_unsafe_interrupts=1 \
default_hugepagesz=1G hugepagesz=1G hugepages=1760

# QEMU memory backend
-object memory-backend-file,id=dimm1,size=768G,\
  mem-path=/dev/hugepages,share=on,prealloc=on,prealloc-threads=8

# No guest vIOMMU
# Do NOT add: -device intel-iommu

# Split pref64 reserves
# GPU ports: pref64-reserve=256G
# IB ports:  pref64-reserve=16M

# Guest kernel cmdline (lean)
root=/dev/vda1 rw console=ttyS0 mitigations=off audit=0 \
nowatchdog pci=noio selinux=0 modprobe.blacklist=nouveau

# VFIO DMA entry limit
echo 2000000 > /sys/module/vfio_iommu_type1/parameters/dma_entry_limit

# Libvirt: parallel hugepage allocation
<allocation mode="immediate" threads="32"/>

# Libvirt: NUMA-aware memory
<numatune>
  <memnode cellid="0" mode="strict" nodeset="0"/>
  <memnode cellid="1" mode="strict" nodeset="1"/>
</numatune>
```

---

## Conclusion

The majority of 8-GPU VM boot time is spent in three places:

1. **Hugepage preallocation** (60-90s → 5-8s with `prealloc-threads`)
2. **VFIO IOMMU mapping** (30-120s → 10-15s with less memory + iommufd)
3. **PCI BAR firmware programming** (10-60s → 2-3s with split pref64)

Kernel 6.19's `iommufd` support is the most impactful single change — it transforms VFIO from an all-or-nothing upfront mapping to an on-demand system that scales gracefully with VM size.

The difference between a 5-minute boot and a 25-second boot is entirely in infrastructure configuration — no application changes required. For DGX/HGX operators running 1 VM per node, these optimizations bring VM boot time within the range where it's no longer a operational bottleneck.
