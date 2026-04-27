---
layout: post
title: "Service VM for NVSwitch Fabric Manager: NVLink Partitioning Without Bare Metal"
date: 2026-04-23 00:00:00 +0000
categories: gpu infra fabricmanager nvswitch
tags: nvswitch fabricmanager fmpm nvlink dgx h200 service-vm partitioning
---


## Introduction

On DGX H200 / HGX H200 systems with NVSwitch, the GPUs communicate via NVLink through 4 NVSwitch ASICs. These NVSwitches enable the full 900 GB/s all-to-all bandwidth between GPUs — but they need a management daemon to configure NVLink routing.

That daemon is **NVIDIA Fabric Manager** (FM). On bare metal, FM runs as a systemd service. But when GPUs are passed through to tenant VMs via VFIO, who manages the NVSwitches?

The answer is the **Service VM** — a lightweight VM that owns only the NVSwitches, runs Fabric Manager, and manages NVLink partitioning for all tenant VMs on the node.

---

## Why NVLink Partitioning Matters

### The Core Problem: NVLink Isolation

On an 8-GPU NVSwitch system, **all 8 GPUs can talk to all 8 GPUs** over NVLink by default. There are no boundaries. In a multi-tenant environment, this is a security and performance isolation failure:

- **Tenant A** owns GPUs 0-3 for a 4-GPU training job
- **Tenant B** owns GPUs 4-7 for inference
- Without NVLink partitioning, Tenant A's GPUs can still send NVLink traffic to Tenant B's GPUs — there is no hardware-level isolation

NVLink partitioning solves this by programming the NVSwitch routing tables to **restrict which GPUs can communicate with each other**. Once a partition is activated:

- GPUs within a partition have full NVLink bandwidth (18 links, 477 GB/s per GPU)
- GPUs in different partitions **cannot reach each other over NVLink at all** — the NVSwitch drops the packets
- This is hardware-enforced isolation, not software policy

```
Without partitioning:              With partitioning (4+4):

  GPU0 ←→ GPU1 ←→ ... ←→ GPU7      Partition 1          Partition 2
  (all-to-all, no isolation)        GPU0 ←→ GPU1          GPU4 ←→ GPU5
                                    GPU2 ←→ GPU3          GPU6 ←→ GPU7
                                    (isolated)            (isolated)
                                         ✗ no NVLink between partitions ✗
```

### Why This Needs Fabric Manager

The NVSwitch routing tables are not self-configuring. Fabric Manager is the only daemon that can:

1. **Discover the NVLink topology** — which GPU connects to which NVSwitch port
2. **Compute valid partitions** — not all GPU groupings are valid (NVLink wiring constrains which GPUs can be co-partitioned)
3. **Program the routing tables** — write the actual NVSwitch registers that enable/disable NVLink paths
4. **Enforce mutual exclusion** — ensure two overlapping partitions can't be active simultaneously

Without FM running, **NVLink doesn't work at all** on multi-GPU NVSwitch systems. The GPUs boot with NVLink disabled; FM activates it.

---

## Why a Service VM?

### The Operational Problem

When you pass GPUs to VMs via VFIO:
- Each tenant VM owns its GPUs directly (VFIO passthrough)
- But NVSwitches must be managed by a single entity — they serve **all** GPUs on the node
- Fabric Manager needs access to NVSwitch devices to program routing tables
- The host kernel may not have the NVIDIA driver loaded (it's in the VMs)
- Tenant VMs must not have NVSwitch access — that would let them reconfigure other tenants' NVLink paths

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Host (no NVIDIA driver)                                 │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Service VM   │  │ Tenant VM 1  │  │ Tenant VM 2  │  │
│  │              │  │              │  │              │  │
│  │ NVSwitch 0-3 │  │ GPU 0-3      │  │ GPU 4-7      │  │
│  │ Fabric Mgr   │  │ IB NIC 0-3   │  │ IB NIC 4-7   │  │
│  │ fmpm tool    │  │              │  │              │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                         │
│  Physical: 8× H200 GPU + 4× NVSwitch + 8× CX-7 IB     │
└─────────────────────────────────────────────────────────┘
```

The Service VM:
- **Tiny footprint**: 2 vCPUs, 4 GiB RAM
- **Owns only NVSwitches**: 4 NVSwitch ASICs via VFIO passthrough
- **Runs Fabric Manager**: Configures NVLink routing tables
- **Exposes fmpm API**: Partition manager for creating GPU partitions

Tenant VMs:
- Own GPUs + IB NICs
- Don't need NVSwitch access (NVLink routing is preconfigured by FM)
- See full NVLink bandwidth between their GPUs

---

## Service VM Configuration

### Libvirt XML

The Service VM is minimal — no GPUs, no IB, just NVSwitches:

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <name>service-vm</name>
  <memory unit='KiB'>4194304</memory>  <!-- 4 GiB -->
  <vcpu placement='static'>2</vcpu>

  <os firmware='efi'>
    <type arch='x86_64' machine='pc-q35-noble'>hvm</type>
    <firmware>
      <feature enabled='no' name='enrolled-keys'/>
      <feature enabled='no' name='secure-boot'/>
    </firmware>
    <loader readonly='yes' type='pflash'>/usr/share/OVMF/OVMF_CODE_4M.fd</loader>
    <nvram template='/usr/share/OVMF/OVMF_VARS_4M.fd'/>
    <boot dev='hd'/>
  </os>

  <cpu mode='host-passthrough' check='none' migratable='on'/>

  <devices>
    <!-- Boot disk -->
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/goldimages/service-vm.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk>

    <!-- Cloud-init seed -->
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/var/lib/libvirt/images/goldimages/seed-svm.iso'/>
      <target dev='sda' bus='sata'/>
      <readonly/>
    </disk>

    <!-- Network for management -->
    <interface type='network'>
      <source network='default'/>
      <model type='virtio'/>
    </interface>

    <!-- ============================================
         NVSwitch passthrough (4 NVSwitch ASICs)
         These are the only PCI devices passed through.
         The GPUs go to tenant VMs, not here.
         ============================================ -->
    <!-- NVSwitch 0 -->
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x07' slot='0x00' function='0x0'/>
      </source>
    </hostdev>
    <!-- NVSwitch 1 -->
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x08' slot='0x00' function='0x0'/>
      </source>
    </hostdev>
    <!-- NVSwitch 2 -->
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x09' slot='0x00' function='0x0'/>
      </source>
    </hostdev>
    <!-- NVSwitch 3 -->
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x0a' slot='0x00' function='0x0'/>
      </source>
    </hostdev>
  </devices>

  <!-- NVSwitch BARs need large pref64 MMIO window -->
  <qemu:commandline>
    <qemu:arg value='-global'/>
    <qemu:arg value='pcie-root-port.pref64-reserve=549755813888'/>
  </qemu:commandline>
</domain>
```

### Key Design Points

**1. pref64-reserve for NVSwitch BARs**

Each NVSwitch has a large BAR for NVLink routing table access. The QEMU global setting reserves 512 GiB of prefetchable MMIO:

```xml
<qemu:arg value='pcie-root-port.pref64-reserve=549755813888'/>
<!-- 549755813888 bytes = 512 GiB -->
```

Without this, OVMF can't assign the NVSwitch BARs and the devices appear broken.

**2. OVMF Firmware (not direct kernel boot)**

Unlike tenant VMs which use direct kernel boot for speed, the Service VM uses OVMF because:
- It starts once and stays running (boot time doesn't matter)
- OVMF handles the NVSwitch PCIe BAR assignment more reliably
- It supports the full UEFI boot chain for the small service OS

**3. No GPUs, No IB**

The Service VM only needs NVSwitches. It doesn't need:
- GPUs (those go to tenant VMs)
- IB NICs (no network traffic, just NVLink management)
- Large memory (FM uses ~500 MiB)
- Many vCPUs (FM is mostly idle)

---

## Fabric Manager Inside the Service VM

### Installation

```bash
# Inside the Service VM
apt-get install -y nvidia-fabricmanager

# Or if using the CUDA repo
apt-get install -y nvidia-fabricmanager-580
```

### Configuration

```bash
# /etc/nvidia-fabricmanager/fm.conf
# Key settings for Service VM operation:

# FM manages NVSwitches for partitioned GPU access
FABRIC_MODE=1

# Topology file (auto-generated on first run)
TOPOLOGY_FILE_PATH=/etc/nvidia-fabricmanager/fm_topology.cfg

# Partition management enabled
FM_STAY_RESIDENT_ON_FAILURES=1
```

### Starting Fabric Manager

```bash
systemctl enable nvidia-fabricmanager
systemctl start nvidia-fabricmanager

# Verify
systemctl status nvidia-fabricmanager
nvidia-fabricmanager --version
```

FM discovers the 4 NVSwitches via PCIe, reads the NVLink topology, and configures routing tables for all possible GPU partitions.

---

## NVLink Partitioning with fmpm

### What Is fmpm?

`fmpm` (Fabric Manager Partition Manager) is the CLI tool that communicates with the running Fabric Manager daemon to:
- List available partitions
- Activate/deactivate partitions
- Query GPU-to-NVLink mappings

### Partition Layout

On H200 with 8 GPUs and NVSwitch, FM pre-computes **15 possible partitions**:

| Partition ID | GPUs | NVLinks per GPU | Use Case |
|-------------|------|-----------------|----------|
| 0 | 1,2,3,4,5,6,7,8 (all 8) | 18 | Full node training |
| 1 | 1,2,3,4 | 18 | Half-node (NUMA 0) |
| 2 | 5,6,7,8 | 18 | Half-node (NUMA 1) |
| 3 | 1,3 | 18 | 2-GPU pair |
| 4 | 2,4 | 18 | 2-GPU pair |
| 5 | 5,7 | 18 | 2-GPU pair |
| 6 | 6,8 | 18 | 2-GPU pair |
| 7-14 | Single GPU each | 0 | Single-GPU instances |

**NVLink count:** Multi-GPU partitions get 18 NVLinks per GPU (full NVSwitch bandwidth). Single-GPU partitions get 0 NVLinks (no inter-GPU communication needed).

### The fmConnectParams_t Bug and Fix

`fmpm` uses `fmConnectParams_t` to establish a connection to the Fabric Manager daemon. The struct contains connection parameters including version, timeout, address type, and address info.

**The bug:** The upstream `fmpm.cpp` in the `Fabric-Manager-Client` repo does not set `connectParams.addressType`. On bare metal this works by accident — the default (zero-initialized) value happens to resolve correctly because FM and fmpm share the same host. But in the Service VM, where fmpm connects to FM over the network (or locally within the SVM), the uninitialized `addressType` causes `fmConnect()` to fail silently or return garbage data.

The symptom: `fmpm -q` either fails to connect, or connects but returns null PCI addresses and empty UUIDs for all GPUs:

```json
{
  "gpuInfo": [
    {
      "pciBusId": "00000000:00:00.0",   // ← null address
      "physicalId": 1,
      "uuid": ""                          // ← empty
    }
  ]
}
```

**The fix** (two lines in `fmpm.cpp`):

```diff
     fmConnectParams_t connectParams;
     connectParams.timeoutMs = 1000;
     connectParams.version = fmConnectParams_version;
+    connectParams.addressType = NV_FM_API_ADDR_TYPE_INET;

     ...

     fmReturn = fmConnect(&connectParams, &fmHandle);
     if (fmReturn != FM_ST_SUCCESS){
-        std::cout << "Failed to connect to Fabric Manager instance." << std::endl;
+        std::cout << "Failed to connect to Fabric Manager instance. " << fmReturn << std::endl;
         return fmReturn;
     }
```

1. **`connectParams.addressType = NV_FM_API_ADDR_TYPE_INET`** — Explicitly set the address type to INET (TCP/IP). Without this, the connection type is ambiguous and FM may not properly service the request.
2. **Print the error code** — The original error message gave no indication of *why* the connection failed. Adding `fmReturn` to the output makes debugging possible (`FM_ST_BADPARAM`, `FM_ST_GENERIC_ERROR`, etc.).

After the fix, `fmpm -q` returns valid partition data. The `pciBusId` and `uuid` fields remain `00000000:00:00.0` and empty respectively — this is expected because the Service VM doesn't own the GPUs, so FM can't resolve PCI BDFs or read GPU UUIDs. The authoritative GPU identifier is `physicalId`.

### Why pciBusId Is Always 00000000:00:00.0

Even with the fmpm fix, PCI bus IDs show null addresses. This is correct behavior:

1. The Service VM owns NVSwitches, not GPUs
2. FM discovers GPUs via NVLink topology (through the NVSwitches), not PCI enumeration
3. FM knows each GPU by its **physical ID** (Module ID from GPIO strap), which is stable hardware identity
4. Without direct PCI access to the GPUs, FM can't resolve their BDFs or read their UUIDs

**Solution:** Use `physicalId` as the authoritative GPU identifier. Map physical IDs to host PCI BDFs in your orchestrator:

```python
# Physical ID → Host PCI BDF mapping (DGX H200)
PHYSICAL_ID_TO_BDF = {
    1: "0000:1b:00.0",  # GPU 0
    2: "0000:43:00.0",  # GPU 1
    3: "0000:52:00.0",  # GPU 2
    4: "0000:61:00.0",  # GPU 3
    5: "0000:9d:00.0",  # GPU 4
    6: "0000:c3:00.0",  # GPU 5
    7: "0000:d1:00.0",  # GPU 6
    8: "0000:df:00.0",  # GPU 7
}
```

The physical ID is stable across host/guest boundaries because it comes from the NVSwitch's NVLink topology, not from PCI enumeration. This is the mapping your orchestrator needs to translate fmpm partitions into VFIO device assignments for tenant VMs.

### Using fmpm

```bash
# Inside the Service VM

# List all partitions
fmpm -q

# Activate partition 0 (all 8 GPUs)
fmpm -a 0

# Activate partition 1 (GPUs 1-4, half node)
fmpm -a 1

# Deactivate partition
fmpm -d 0

# Query active partitions
fmpm -q | jq '.partitionInfo[] | select(.isActive == 1)'
```

### fmpm Output Explained

```json
{
  "maxNumPartitions": 15,
  "numPartitions": 15,
  "partitionInfo": [
    {
      "partitionId": 0,
      "isActive": 1,           // Currently active
      "numGpus": 8,
      "gpuInfo": [
        {
          "physicalId": 1,      // GPIO strap ID (stable, use this!)
          "pciBusId": "00000000:00:00.0",  // Not resolved in SVM
          "uuid": "",           // Not available without GPU access
          "maxNumNvLinks": 18,
          "numNvLinksAvailable": 18,
          "nvlinkLineRateMBps": 26562     // 26.5 GB/s per link
        }
      ]
    }
  ]
}
```

**Key fields:**
- `physicalId` — The GPU's hardware Module ID (1-8). This is the reliable identifier.
- `maxNumNvLinks` — Maximum NVLinks the GPU can use (18 for multi-GPU, 0 for single)
- `nvlinkLineRateMBps` — Per-link bandwidth: 26,562 MB/s = 26.5 GB/s
- `isActive` — Whether this partition's NVLink routing is currently configured

### Total NVLink Bandwidth per Partition

```
Per GPU: 18 links × 26.5 GB/s = 477 GB/s bidirectional
8-GPU partition: 477 GB/s per GPU, full mesh via NVSwitch
4-GPU partition: 477 GB/s per GPU, full mesh via NVSwitch
2-GPU partition: 477 GB/s per GPU, point-to-point via NVSwitch
1-GPU partition: 0 GB/s (no NVLink, PCIe only)
```

---

## Operational Flow

### Starting the System

```
1. Host boots (no NVIDIA driver)
2. Start Service VM → FM discovers NVSwitches
3. fmpm -a 0 → Activate full 8-GPU partition (default)
4. Start Tenant VM → GPUs get full NVLink bandwidth
```

### Changing Partitions

```
1. Stop Tenant VMs using the current partition
2. fmpm -d <old_partition_id>
3. fmpm -a <new_partition_id>
4. Start new Tenant VMs with the appropriate GPU set
```

### Example: Split Node into Two 4-GPU VMs

```bash
# Service VM:
fmpm -d 0    # Deactivate full-node partition
fmpm -a 1    # Activate GPUs 1-4 partition
fmpm -a 2    # Activate GPUs 5-8 partition

# Host:
virsh start tenant-vm-numa0    # Gets GPUs 0-3
virsh start tenant-vm-numa1    # Gets GPUs 4-7
```

Both VMs get full NVLink bandwidth (18 links per GPU) within their partition. Critically, **GPU 0 in Tenant VM 1 cannot send NVLink traffic to GPU 4 in Tenant VM 2** — the NVSwitch routing tables block it. This is hardware-enforced tenant isolation.

---

## Service VM vs Bare Metal FM

| Feature | Bare Metal FM | Service VM FM |
|---------|--------------|---------------|
| FM runs on | Host OS | Dedicated lightweight VM |
| NVSwitch access | Host PCI | VFIO passthrough |
| GPU PCI BDF resolution | Yes (FM sees GPUs) | No (GPUs in tenant VMs) |
| UUID resolution | Yes | No (use physicalId) |
| Host NVIDIA driver | Required | Not required |
| Isolation | None (host has full access) | Strong (NVSwitches isolated) |
| Recovery | Restart FM service | Restart/rebuild SVM |
| Resource overhead | Minimal | 2 vCPU + 4 GiB RAM |

### When to Use a Service VM

- **Multi-tenant GPU nodes**: Each tenant gets their own GPU VM, SVM manages shared NVLink fabric
- **Host-minimal deployments**: Host runs only libvirt/QEMU, no NVIDIA stack
- **Security isolation**: NVSwitch management separated from tenant workloads

### When NOT to Use a Service VM

- **Single-tenant bare metal**: Just run FM on the host
- **Kata Containers / Kubernetes**: FM runs as a DaemonSet pod (no VM needed)
- **Systems without NVSwitch**: A100/H100 PCIe systems don't have NVSwitches

---

## B200 / B300: No PCIe NVSwitches — CX-7 QM3 Management Path

### The Architecture Difference

On **DGX H200** (Hopper), the 4 NVSwitch ASICs are standard PCIe devices visible to `lspci`:

```
07:00.0 3D controller: NVIDIA Corporation NVSwitch ...
08:00.0 3D controller: NVIDIA Corporation NVSwitch ...
09:00.0 3D controller: NVIDIA Corporation NVSwitch ...
0a:00.0 3D controller: NVIDIA Corporation NVSwitch ...
```

You can VFIO-passthrough them to a Service VM and run Fabric Manager inside. Simple.

On **DGX B200 / B300** (Blackwell), **there are no PCIe NVSwitch devices**. The NVSwitch ASICs communicate with GPUs exclusively over NVLink — they have no PCIe interface at all. They don't show up in `lspci`. You can't bind them to `vfio-pci`. You can't pass them to a VM.

So how does Fabric Manager manage the NVSwitches?

### CX-7 QM3: The In-Band Management Channel

On B200/B300, NVSwitch management goes through the **ConnectX-7 QM3 management PFs** — four special CX-7 ports at fixed BDFs:

```
0000:05:00.0  Mellanox Technologies ConnectX-7 [QM3 Mgmt]
0000:05:00.1  Mellanox Technologies ConnectX-7 [QM3 Mgmt]
0000:05:00.2  Mellanox Technologies ConnectX-7 [QM3 Mgmt]
0000:05:00.3  Mellanox Technologies ConnectX-7 [QM3 Mgmt]
```

These are identifiable by their VPD field `SMDL=SW_MNG` (Switch Management). They provide an in-band management channel from the PCIe domain to the NVLink domain, allowing Fabric Manager to:

- Read NVLink topology
- Configure NVLink routing tables
- Activate/deactivate NVLink partitions
- Monitor NVLink health

### Service VM on B200/B300

The Service VM concept still works, but the passthrough devices change:

```
┌─────────────────────────────────────────────────────────────────┐
│                      H200 Service VM                            │
│  Passed through: 4× NVSwitch PCIe devices (07-0a:00.0)         │
│  FM talks to NVSwitches directly via PCIe                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    B200/B300 Service VM                          │
│  Passed through: 4× CX-7 QM3 mgmt PFs (05:00.0-3)             │
│  FM talks to NVSwitches via CX-7 in-band management channel    │
└─────────────────────────────────────────────────────────────────┘
```

The libvirt XML for a B200 Service VM replaces the NVSwitch `<hostdev>` entries with the QM3 management PFs:

```xml
<!-- B200 Service VM: pass through CX-7 QM3 mgmt PFs instead of NVSwitches -->
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x05' slot='0x00' function='0x0'/>
  </source>
</hostdev>
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x05' slot='0x00' function='0x1'/>
  </source>
</hostdev>
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x05' slot='0x00' function='0x2'/>
  </source>
</hostdev>
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x05' slot='0x00' function='0x3'/>
  </source>
</hostdev>
```

### Other B200/B300 Differences

Beyond the NVSwitch management path, B200/B300 differs from H200 in IB networking:

| Feature | H200 | B200/B300 |
|---------|------|-----------|
| NVSwitch on PCIe | Yes (4 devices) | No (NVLink only) |
| FM management path | PCIe to NVSwitch | CX-7 QM3 in-band |
| Service VM passthrough | NVSwitch PCI devices | CX-7 QM3 mgmt PFs (05:00.0-3) |
| IB per GPU | SR-IOV VFs from shared PFs | Dedicated CX-7 IB PF per GPU |
| IB SR-IOV | Yes (16 VFs per PF) | No (PF passthrough directly) |
| GPU-IB topology | GPU + IB behind Broadcom PEX switch | GPU + IB behind CX-7 internal PCIe bridge |

On B200, each GPU has a **dedicated CX-7 IB PF** wired directly to it through the CX-7's internal PCIe bridge (MT2910). The PCIe topology looks like:

```
CX-7 Bridge (4e:00.0)
  ├── Port 00.0 → IB PF (4f:00.0)     ← dedicated to this GPU
  └── Port 02.0 → GPU (52:00.0)
```

GPU and IB share the **same CX-7 internal switch fabric** — GPUDirect RDMA never touches the CPU. This is an architectural guarantee, not a configuration choice.

Because IB PFs are dedicated (not shared), B200 doesn't need SR-IOV for IB at all — just pass the whole PF to the tenant VM alongside its paired GPU. The `kata-vfio-ibvfs.sh` script is a no-op on B200.

### Identifying CX-7 QM3 Ports

To find the QM3 management PFs on a B200/B300 system:

```bash
# Method 1: Known BDF prefix (stable across B200 systems)
lspci -s 05:00 -v

# Method 2: VPD field (most reliable)
for bdf in $(lspci -d 15b3: -D | awk '{print $1}'); do
    vpd=$(lspci -s "$bdf" -vvv 2>/dev/null | grep -i 'SMDL' || true)
    if echo "$vpd" | grep -q 'SW_MNG'; then
        echo "QM3 mgmt PF: $bdf"
    fi
done

# Method 3: These PFs should NOT have sriov_numvfs capability
# and should NOT be in /sys/class/infiniband/ (they're mgmt, not data)
```

These 4 PFs **must stay on the host** (or be passed to the Service VM) — they are NOT data-plane IB ports and should never go to tenant VMs. Our device-plugin code handles this:

```go
// Skip CX-7 Bridge mgmt PFs (B200 0000:05:00.[0-3])
if strings.HasPrefix(bdf, "0000:05:00.") {
    dlogf("bootstrap", "skip %s (%s): CX-7 Bridge mgmt PF", name, bdf)
    continue
}
```

---

## Troubleshooting

### FM Can't Find NVSwitches

```bash
# Inside SVM — verify NVSwitches are visible
lspci | grep -i nvswitch
nvidia-smi -L   # Should show NVSwitch devices
dmesg | grep -i nvswitch
```

### fmpm Returns All Zeros for pciBusId

This is expected behavior when FM runs in the SVM without GPU access. Use `physicalId` instead:

```bash
fmpm -q | jq '.partitionInfo[].gpuInfo[].physicalId'
```

### NVLink Not Working in Tenant VM

```bash
# Inside Tenant VM
nvidia-smi nvlink --status -i 0
# If all links show "inactive":
# → The partition isn't activated in the SVM
# → fmpm -a <partition_id> in the SVM
```

---

## Conclusion

The Service VM pattern decouples NVSwitch management from GPU workload execution:

1. **Service VM** owns NVSwitches (H200: PCIe passthrough) or CX-7 QM3 mgmt PFs (B200/B300: in-band management), runs Fabric Manager, manages partitions via fmpm
2. **Tenant VMs** own GPUs + IB NICs, get full NVLink bandwidth without FM access
3. **fmpm `physicalId`** is the stable GPU identifier — ignore `pciBusId` (it's the SVM's guest address). Fix: set `connectParams.addressType = NV_FM_API_ADDR_TYPE_INET`
4. **15 pre-computed partitions** cover 8g, 4g, 2g, and 1g GPU slicing with full NVLink bandwidth for multi-GPU partitions
5. **B200/B300** has no PCIe NVSwitches — FM reaches NVSwitches through CX-7 QM3 management PFs (`0000:05:00.0-3`, VPD `SMDL=SW_MNG`), and each GPU has a dedicated CX-7 IB PF (no SR-IOV needed)

The 4 GiB / 2 vCPU Service VM is a negligible resource cost for the operational flexibility it provides — clean host separation, partition management without host NVIDIA drivers, and a single control point for NVLink topology across both Hopper and Blackwell platforms.
