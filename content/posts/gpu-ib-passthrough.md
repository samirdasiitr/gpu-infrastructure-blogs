---
title: "GPU and InfiniBand Passthrough for DGX/HGX H200"
date: 2026-04-21
draft: false
tags: ["gpu", "nccl", "infiniband", "vfio", "pcie"]
---

# GPU and InfiniBand Passthrough for DGX/HGX H200: A Deep Dive into PCIe Topology, IOMMU Groups, and NCCL Performance

## Introduction

Running GPU workloads inside virtual machines requires careful attention to PCIe topology, IOMMU group isolation, and network fabric configuration. This guide covers the complete stack for NVIDIA DGX H200 and HGX H200 platforms — from VFIO passthrough setup to achieving near-baremetal NCCL all-reduce performance.

We'll cover:
- IOMMU groups and ACS override for VFIO passthrough
- GPU + InfiniBand affinity under shared PCIe switches
- The difference between TI XIO3130 and Broadcom PEX890xx switch topologies
- NCCL topology files for PIX vs PHB-level GDR
- Achieving 400 Gb/s per-rail IB bandwidth inside VMs

---

## PCIe Topology: Why It Matters for GPU Virtualization

### The Physical Layout

A DGX H200 / HGX H200 system has 8 NVIDIA H200 GPUs, 8 ConnectX-7 InfiniBand NICs, and 4 NVSwitches, all interconnected through PCIe Gen 5 switches. The critical architectural detail is that **each GPU is paired with an affine IB NIC behind the same PCIe switch**.

```
CPU Socket 0                          CPU Socket 1
    │                                     │
    ├── PCIe Root Port                    ├── PCIe Root Port
    │   └── PCIe Switch                   │   └── PCIe Switch
    │       ├── GPU 0 (H200)              │       ├── GPU 4 (H200)
    │       └── CX-7 IB NIC 0            │       └── CX-7 IB NIC 4
    │                                     │
    ├── PCIe Root Port                    ├── PCIe Root Port
    │   └── PCIe Switch                   │   └── PCIe Switch
    │       ├── GPU 1 (H200)              │       ├── GPU 5 (H200)
    │       └── CX-7 IB NIC 1            │       └── CX-7 IB NIC 5
    │                                     │
    ├── PCIe Root Port                    ├── PCIe Root Port
    │   └── PCIe Switch                   │   └── PCIe Switch
    │       ├── GPU 2 (H200)              │       ├── GPU 6 (H200)
    │       └── CX-7 IB NIC 2            │       └── CX-7 IB NIC 6
    │                                     │
    ├── PCIe Root Port                    ├── PCIe Root Port
    │   └── PCIe Switch                   │   └── PCIe Switch
    │       ├── GPU 3 (H200)              │       ├── GPU 7 (H200)
    │       └── CX-7 IB NIC 3            │       └── CX-7 IB NIC 7
    │                                     │
    └── NVSwitch 0, 1                    └── NVSwitch 2, 3
```

This pairing exists because GPUDirect RDMA (GDR) requires the GPU and IB NIC to communicate with minimal PCIe hops. When both are behind the same switch, DMA transfers go directly through the switch fabric without traversing the CPU's root complex.

### PCIe Proximity Levels

NVIDIA defines proximity levels that NCCL uses to make routing decisions:

| Level | Name | Meaning | Typical Bandwidth |
|-------|------|---------|-------------------|
| PIX | Same Switch | GPU and NIC behind the same PCIe switch | Full PCIe Gen5 x16 (~64 GB/s) |
| PXB | Cross Bridge | Traverses multiple PCIe bridges/switches | Slightly reduced |
| PHB | Host Bridge | Goes through the CPU's PCIe root complex | ~32-48 GB/s |
| SYS | Cross Socket | Crosses the inter-socket link (UPI/QPI) | ~16-24 GB/s |
| NV# | NVLink | # NVLink connections (GPU-GPU only) | 18×26.5 = ~477 GB/s |

**For GPUDirect RDMA, PIX is optimal.** Data flows: GPU → PCIe switch → IB NIC, never touching CPU memory.

---

## Two Switch Architectures: TI XIO3130 vs Broadcom PEX890xx

### ConnectX-7 Internal Switch (TI XIO3130-based)

On DGX H200 systems, the ConnectX-7 NIC itself acts as a PCIe switch. The CX-7 contains an internal bridge (based on TI XIO3130 silicon) that provides downstream ports for both the IB function and the paired GPU.

```
lspci -tv (inside VM with device passthrough):

 +-[0000:64]-+-00.0  TI XIO3130 PCIe Switch (Upstream)
 |           +-00.0-[65-66]--+-00.0  TI XIO3130 (Downstream)
 |           |               └-01.0  TI XIO3130 (Downstream)
 |           +-66:00.0  NVIDIA H200 GPU
 |           └-67:00.0  Mellanox ConnectX-7 IB
```

**Key characteristic:** When the TI switch is passed through alongside GPU and IB, `nvidia-smi topo -m` shows **PIX** between GPU and NIC — both are behind the same virtual PCIe switch.

**PCIe Device IDs:**
- `104c:8232` — TI XIO3130 Upstream Port
- `104c:8233` — TI XIO3130 Downstream Port

### Broadcom PEX890xx Gen 5 Switch

HGX H200 systems (the baseboard variant without integrated CPU) use Broadcom PEX890xx PCIe Gen 5 switches. These are external switches connecting GPUs, NICs, NVMe SSDs, and the host CPU.

```
lspci -tv (host view):

 +-[0000:4c]-+-01.0-[4d-52]--00.0-[4e-52]--  Broadcom PEX890xx
 |                                           +-00.0-[4f]  CX-7 IB PF + VFs
 |                                           └-02.0-[50-52]  NVIDIA H200 GPU
```

**Key characteristic:** The PEX890xx is a larger switch fabric that also connects NVMe storage and management endpoints. It exposes management functions via the `switchtec` driver for bandwidth monitoring.

**PCIe Device IDs:**
- `1000:c030` — PEX890xx Switch Port (upstream/downstream)
- `1000:00b2` — PEX890xx Management Endpoint

### Topology Comparison

| Feature | TI XIO3130 (DGX) | Broadcom PEX890xx (HGX) |
|---------|-------------------|-------------------------|
| Location | Internal to CX-7 | External baseboard switch |
| Gen | PCIe Gen 5 | PCIe Gen 5 |
| Ports per switch | 2-3 (GPU + IB + optional NVMe) | 36-72 (shared fabric) |
| Management counters | No (no management endpoint) | Yes (via switchtec) |
| Pass-through needed | TI switch + GPU + IB | GPU + IB (switch too large to pass) |
| Guest PIX topology | Yes (if switch passed through) | No (devices on separate root ports) |

---

## IOMMU Groups: The Foundation of VFIO Passthrough

### What Are IOMMU Groups?

An IOMMU group is the smallest set of devices that the IOMMU can isolate. VFIO requires that **all devices in a group** be assigned to the same VM (or all stay on the host). You cannot split an IOMMU group across host and guest.

### Discovering IOMMU Groups

```bash
# List all devices with their IOMMU groups
for g in /sys/kernel/iommu_groups/*/devices/*; do
    GROUP=$(basename $(dirname $(dirname $g)))
    BDF=$(basename $g)
    DESC=$(lspci -nns $BDF 2>/dev/null | cut -d' ' -f2-)
    echo "group $GROUP: $BDF $DESC"
done | sort -t' ' -k2 -n

# Filter to just GPU and IB devices
for d in $(lspci -D | grep -iE '10de:|15b3:1021' | awk '{print $1}'); do
    GRP=$(basename $(readlink -f /sys/bus/pci/devices/$d/iommu_group))
    MEMBERS=$(ls /sys/kernel/iommu_groups/$GRP/devices | wc -l)
    echo "group $GRP ($MEMBERS devs): $d $(lspci -s $d | cut -d' ' -f2-)"
done
```

### The Giant IOMMU Group Problem

On many HGX systems, you'll find that **every device on the baseboard is in a single IOMMU group**. This happens because the PCIe switches lack Access Control Services (ACS) — without ACS, the IOMMU can't guarantee isolation between devices behind the same switch, so the kernel conservatively groups everything together.

```
# Symptom: one massive group containing ALL GPUs, NICs, NVMe, NVSwitches...
group 45: 0000:1b:00.0 3D controller: NVIDIA H200
group 45: 0000:43:00.0 3D controller: NVIDIA H200
group 45: 0000:18:00.0 Infiniband controller: ConnectX-7
group 45: 0000:40:00.0 Infiniband controller: ConnectX-7
... (hundreds of devices)
```

### ACS Override: Breaking the Giant Group

The `pcie_acs_override` kernel parameter tells the kernel to trust PCIe switch isolation even without hardware ACS. This is standard practice for GPU virtualization on DGX/HGX:

```bash
# Add to kernel boot parameters
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt pcie_acs_override=downstream,multifunction vfio_iommu_type1.allow_unsafe_interrupts=1"

update-grub
reboot
```

After reboot, each GPU and IB NIC should be in its own small IOMMU group (1-3 devices):

```
group 45: 0000:1b:00.0 3D controller: NVIDIA H200
group 46: 0000:18:00.0 Infiniband controller: ConnectX-7
group 47: 0000:43:00.0 3D controller: NVIDIA H200
group 48: 0000:40:00.0 Infiniband controller: ConnectX-7
```

**Security note:** `pcie_acs_override` trusts the PCIe fabric to provide isolation. On bare-metal DGX/HGX with a single tenant, this is the standard NVIDIA-recommended configuration.

### Required Kernel Parameters

| Parameter | Purpose |
|-----------|---------|
| `intel_iommu=on` | Enable Intel VT-d IOMMU |
| `iommu=pt` | Passthrough mode — host devices bypass IOMMU for performance |
| `pcie_acs_override=downstream,multifunction` | Split large IOMMU groups |
| `vfio_iommu_type1.allow_unsafe_interrupts=1` | Allow VFIO on platforms without interrupt remapping |

---

## VFIO Passthrough: GPU + IB to a VM

### PF vs VF Passthrough

For IB NICs, you have two options:

| Approach | Bandwidth | Complexity | Use Case |
|----------|-----------|------------|----------|
| **PF Passthrough** | Full 400 Gb/s | Simple | 1 VM per node |
| **SR-IOV VF** | Shared (depends on config) | Complex | Multiple VMs per node |

**For 1 VM per node (the DGX model), always use PF passthrough.** VFs add overhead and complexity with no benefit when there's only one tenant.

### Setting Up VFIO Passthrough

```bash
# 1. Load VFIO modules
modprobe vfio-pci

# 2. Unbind devices from host drivers and bind to vfio-pci
for bdf in 0000:1b:00.0 0000:18:00.0; do  # GPU + IB PF
    echo $bdf > /sys/bus/pci/devices/$bdf/driver/unbind 2>/dev/null
    echo "vfio-pci" > /sys/bus/pci/devices/$bdf/driver_override
    echo $bdf > /sys/bus/pci/drivers_probe
done

# 3. Verify
ls /dev/vfio/
```

### Libvirt XML for GPU + IB Passthrough

```xml
<!-- GPU 0 (NUMA 0) -->
<hostdev mode="subsystem" type="pci" managed="yes">
  <driver name="vfio"/>
  <source><address domain="0x0000" bus="0x1b" slot="0x00" function="0x0"/></source>
  <rom bar="off"/>
</hostdev>

<!-- IB PF for GPU 0 (same CX-7 switch) -->
<hostdev mode="subsystem" type="pci" managed="yes">
  <driver name="vfio"/>
  <source><address domain="0x0000" bus="0x18" slot="0x00" function="0x0"/></source>
  <rom bar="off"/>
</hostdev>
```

With `managed='yes'`, libvirt automatically handles driver unbind/rebind on VM start/stop.

### Memory Configuration for Large VMs

GPU passthrough VMs with hugepages require careful memory sizing:

```xml
<memory unit="KiB">1073741824</memory>  <!-- 1024 GiB total -->
<memoryBacking>
  <hugepages/>
  <source type="file"/>
  <access mode="shared"/>
  <allocation mode="immediate" threads="32"/>  <!-- parallel hugepage allocation -->
</memoryBacking>
```

The `threads="32"` parameter parallelizes hugepage preallocation — without it, touching 1 TiB of 1 GiB hugepages takes 60+ seconds serially.

---

## Inside the VM: PCIe Topology and NCCL

### The PIX vs PHB Challenge

When you pass GPU and IB as separate `<hostdev>` entries, libvirt places each on its own virtual pcie-root-port:

```
Guest PCI topology:
  pcie.0 (root bus)
    ├── pci.6 (root port) → GPU 0
    ├── pci.7 (root port) → GPU 1
    ...
    ├── pci.14 (root port) → IB 0
    ├── pci.15 (root port) → IB 1
```

Since GPU and IB are on **separate root ports**, `nvidia-smi topo -m` shows **PHB** (not PIX):

```
        GPU0    NIC0    NIC1
GPU0     X      PHB     PHB      ← No PIX!
NIC0    PHB      X      PHB
NIC1    PHB     PHB      X
```

### Restoring PIX Topology

To get PIX, the GPU and IB must be behind the **same virtual PCIe switch** in the guest. Two approaches:

**Approach 1: Pass the TI switch through (DGX systems)**

If the TI XIO3130 switch is in the same IOMMU group as the GPU/IB, it gets passed through automatically. The guest then sees the physical switch topology and reports PIX.

**Approach 2: NCCL topology file override (works everywhere)**

Even without guest-level PIX, NCCL can be told the correct affinity via `NCCL_TOPO_FILE`. This is the recommended approach for production:

```xml
<!-- /etc/nccl-topo.xml -->
<system version="1">
  <cpu numaid="0" affinity="ffffffff,ffffffff" arch="x86_64" vendor="GenuineIntel">
    <!-- GPU 0 and its affine IB NIC under the same CPU node -->
    <pci busid="0000:06:00.0" class="0x030200" link_speed="32.0 GT/s PCIe" link_width="16">
      <gpu dev="0" sm="90" mem="141312" gdr="1"/>
    </pci>
    <pci busid="0000:0e:00.0" class="0x020700" link_speed="32.0 GT/s PCIe" link_width="16">
      <nic name="mlx5_0" dev="0" speed="400" port="1" gdr="1"/>
    </pci>
    <!-- GPU 1 and its affine IB NIC -->
    <pci busid="0000:07:00.0" class="0x030200" link_speed="32.0 GT/s PCIe" link_width="16">
      <gpu dev="1" sm="90" mem="141312" gdr="1"/>
    </pci>
    <pci busid="0000:0f:00.0" class="0x020700" link_speed="32.0 GT/s PCIe" link_width="16">
      <nic name="mlx5_1" dev="1" speed="400" port="1" gdr="1"/>
    </pci>
    <!-- ... repeat for all 8 GPU+NIC pairs ... -->
  </cpu>
  <nvswitches>
    <nvswitch link_bw="900"/>
  </nvswitches>
</system>
```

### Does PHB vs PIX Affect Performance?

**Short answer: No, for data throughput.**

The guest's virtual PCIe topology is just metadata — the actual DMA path is determined by the host hardware. When the GPU does a RDMA write to the IB NIC:

1. GPU issues a PCIe TLP (Transaction Layer Packet)
2. The TLP travels through the **host's physical PCIe fabric**
3. The host's TI/Broadcom switch routes it directly to the IB NIC
4. The CPU is never involved (true P2P)

The guest seeing PHB instead of PIX only affects:
- **NCCL's automatic topology detection** — NCCL may not pair GPU with the optimal NIC
- **GDR enable/disable decision** — NCCL may refuse GDR at certain levels

Both are solved by `NCCL_TOPO_FILE` and `NCCL_NET_GDR_LEVEL=SYS`.

### Proving P2P (Not PHB) at the Hardware Level

Use GPU PCIe counters to prove direct DMA:

```bash
# Terminal 1: Run RDMA traffic with CUDA memory
ib_write_bw -d mlx5_0 --use_cuda=0 -s 4194304 -n 50000 &
sleep 2
ib_write_bw -d mlx5_0 --use_cuda=0 -s 4194304 -n 50000 localhost

# Terminal 2: Watch GPU PCIe counters
nvidia-smi dmon -s t -i 0 -d 1
# Columns: pcie_tx (MB/s)  pcie_rx (MB/s)
```

If the GPU shows PCIe TX/RX matching the IB transfer rate, it's doing DMA directly (P2P). If the GPU shows zero but the transfer works, the CPU is staging the data (PHB).

---

## NCCL Configuration for Maximum Performance

### Environment Variables

```bash
# IB device selection (auto-detect per node)
export NCCL_IB_HCA=$(for d in /sys/class/infiniband/mlx5_*; do
    [ "$(cat $d/ports/1/link_layer)" = "InfiniBand" ] && basename $d
done | paste -sd,)

# Topology and GDR
export NCCL_TOPO_FILE=/etc/nccl-topo.xml
export NCCL_NET_GDR_LEVEL=SYS     # Allow GDR at any topology distance
export NCCL_NET_GDR_READ=1         # Enable GDR for reads

# Performance tuning
export NCCL_IB_QPS_PER_CONNECTION=8
export NCCL_MIN_NCHANNELS=32
export NCCL_SOCKET_IFNAME=enp2s0   # Management network for bootstrap

# GPUDirect RDMA kernel module
modprobe nvidia_peermem  # or nv_peer_mem
```

### Verifying the Setup

```bash
# NVLink between GPUs
nvidia-smi topo -m
# Should show NV18 between all GPU pairs

# IB port status
for dev in $(ls /sys/class/infiniband/ | grep mlx5); do
    RATE=$(cat /sys/class/infiniband/$dev/ports/1/rate)
    STATE=$(cat /sys/class/infiniband/$dev/ports/1/state)
    echo "$dev: $RATE $STATE"
done
# Should show: 400 Gb/sec (4X NDR), 4: ACTIVE

# NVLink bandwidth (18 links × 26.5 GB/s = ~477 GB/s)
nvidia-smi nvlink --status -i 0
```

### Expected NCCL All-Reduce Performance

For a 2-node × 8 GPU all-reduce with 16 GiB message:

| Configuration | Bus BW | Notes |
|--------------|--------|-------|
| Baremetal (reference) | ~380-400 GB/s | Full 8× NDR400 IB |
| VM with PF passthrough + GDR | ~360-380 GB/s | Near-baremetal |
| VM with VF + GDR | ~280-340 GB/s | VF overhead |
| VM with VF, no GDR | ~180-200 GB/s | CPU staging |

---

## Troubleshooting

### GDR Shows 0 in NCCL

```bash
# Check nvidia_peermem is loaded
lsmod | grep -iE 'peer|gdr'
# If not: modprobe nvidia_peermem

# Check NCCL plugin supports GDR
NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=NET your_test 2>&1 | grep -i gdr

# Force GDR
export NCCL_NET_GDR_LEVEL=SYS
export NCCL_NET_GDR_READ=1
```

### Different mlx5 Numbering Across Nodes

The `mlx5_N` naming depends on PCI enumeration order, which varies between hosts. Auto-detect instead of hardcoding:

```bash
export NCCL_IB_HCA=$(for d in /sys/class/infiniband/mlx5_*; do
    [ "$(cat $d/ports/1/link_layer 2>/dev/null)" = "InfiniBand" ] && basename $d
done | paste -sd,)
```

### QEMU Hangs with Many PCIe Root Ports

Each pcie-root-port with `pref64-reserve=256G` consumes MMIO address space. More than ~10 ports with 256G each (>2.5 TiB total) can hang SeaBIOS during BAR programming. Solutions:
- Reduce pref64-reserve on non-GPU ports (IB NICs only need 16M)
- Use fewer root ports
- Group GPU + IB behind a single port

---

## Conclusion

GPU and IB passthrough on DGX/HGX H200 is production-ready when configured correctly. The key principles:

1. **ACS override** enables per-device IOMMU groups
2. **PF passthrough** gives full 400 Gb/s per NIC (no VF overhead)
3. **NCCL topology files** compensate for the guest's flat PCIe topology
4. **nvidia_peermem** enables GPUDirect RDMA for zero-copy GPU↔NIC transfers
5. The **physical P2P path** through the CX-7/TI/Broadcom switch works regardless of the guest's virtual topology

With this setup, multi-node NCCL all-reduce achieves 90-95% of baremetal bandwidth — the remaining gap is purely VFIO DMA mapping overhead, not a fundamental limitation.
