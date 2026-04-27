---
layout: post
title: "Automating GPU VM Lifecycle: Libvirt QEMU Hooks for NVLink Partitions and UFM Pkey Isolation"
date: 2025-04-24 00:00:00 +0000
categories: gpu infra orchestration
tags: qemu libvirt hooks fabricmanager nvlink ufm pkey infiniband isolation
---

## Introduction

When a tenant VM starts on a multi-GPU node, two things must happen **before** the VM's GPUs and IB NICs are usable:

1. **NVLink partition activation** — Fabric Manager must program the NVSwitch routing tables so the tenant's GPUs can talk to each other (and only each other)
2. **IB pkey assignment** — the InfiniBand Subnet Manager must assign partition keys (pkeys) to the tenant's IB ports so they can communicate on the fabric (and only with authorized peers)

Doing this manually is error-prone. Libvirt's **QEMU hook** mechanism lets us automate both steps as part of the VM lifecycle — partition activation on start, cleanup on stop.

---

## The Two Isolation Planes

Multi-tenant GPU infrastructure needs isolation on two independent planes:

```
┌─────────────────────────────────────────────────────────┐
│                    Physical Node                         │
│                                                          │
│  NVLink plane (intra-node, NVSwitch):                    │
│    GPU0 ←→ GPU1 ←→ GPU2 ←→ GPU3   [Partition 1]         │
│    GPU4 ←→ GPU5 ←→ GPU6 ←→ GPU7   [Partition 2]         │
│           ✗ no cross-partition NVLink ✗                   │
│                                                          │
│  IB plane (inter-node, fabric):                          │
│    IB0-IB3 → pkey 0x0005  [Tenant A fabric]              │
│    IB4-IB7 → pkey 0x0006  [Tenant B fabric]              │
│           ✗ different pkeys can't communicate ✗           │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**NVLink isolation** is local to one node — it controls which GPUs can use NVLink to talk to each other via NVSwitch. Managed by Fabric Manager (fmpm) in the Service VM.

**IB pkey isolation** is fabric-wide — it controls which IB ports can communicate across the entire InfiniBand network. Managed by UFM (Unified Fabric Manager) running on the IB subnet manager node.

Both must be configured per-tenant, per-VM, and both must be automated.

---

## Libvirt QEMU Hooks

### How Hooks Work

Libvirt calls `/etc/libvirt/hooks/qemu` at key points in a VM's lifecycle:

```
/etc/libvirt/hooks/qemu <domain> <action> <sub-action> -
```

The actions we care about:

| Action | Sub-action | When | Use |
|--------|-----------|------|-----|
| `prepare` | `begin` | Before VFIO devices are claimed | Activate NVLink partition, assign pkeys |
| `started` | `begin` | VM is running | Post-start validation |
| `stopped` | `end` | VM has stopped | Deactivate partition, remove pkeys |
| `release` | `end` | All resources released | Final cleanup |

The hook receives the **full domain XML on stdin** — this is how we determine which GPUs and IB devices are assigned to the VM.

### Hook Script: `/etc/libvirt/hooks/qemu`

```bash
#!/bin/bash
# /etc/libvirt/hooks/qemu
#
# Automates NVLink partition activation (via Service VM fmpm) and
# IB pkey assignment (via UFM REST API) on tenant VM lifecycle.

set -euo pipefail

DOMAIN="$1"
ACTION="$2"
SUB_ACTION="$3"

LOG="/var/log/libvirt/hooks/qemu-hook.log"
mkdir -p "$(dirname "$LOG")"

log() { echo "$(date -Iseconds) [${DOMAIN}] $*" >> "$LOG"; }

# ── Configuration ──────────────────────────────────────────
SERVICE_VM_IP="192.168.122.10"        # Service VM management IP
SERVICE_VM_SSH="ssh -o StrictHostKeyChecking=no root@${SERVICE_VM_IP}"
UFM_HOST="ufm-mgmt.cluster.local"     # UFM REST API endpoint
UFM_USER="admin"
UFM_PASS="admin"                       # Use vault/secrets in production
# ───────────────────────────────────────────────────────────

# Read domain XML from stdin (libvirt passes it)
DOMAIN_XML=$(cat)

case "${ACTION}/${SUB_ACTION}" in
    prepare/begin)
        log "prepare/begin: activating NVLink partition and IB pkeys"
        activate_nvlink_partition
        assign_ib_pkeys
        ;;
    stopped/end)
        log "stopped/end: deactivating NVLink partition and removing pkeys"
        deactivate_nvlink_partition
        remove_ib_pkeys
        ;;
    *)
        log "${ACTION}/${SUB_ACTION}: no-op"
        ;;
esac

exit 0
```

---

## Part 1: NVLink Partition Activation via QEMU Hook

### Mapping VM GPUs to Partition ID

The hook parses the domain XML to extract GPU host BDFs, maps them to physical IDs, then finds the matching fmpm partition:

```bash
# ── Physical ID mapping (DGX H200) ────────────────────────
# Host PCI BDF → Fabric Manager physical ID (GPIO strap Module ID)
declare -A BDF_TO_PHYSID=(
    ["0000:1b:00.0"]=1   # GPU 0
    ["0000:43:00.0"]=5   # GPU 1
    ["0000:52:00.0"]=7   # GPU 2
    ["0000:61:00.0"]=6   # GPU 3
    ["0000:9d:00.0"]=4   # GPU 4
    ["0000:c3:00.0"]=1   # GPU 5
    ["0000:d1:00.0"]=3   # GPU 6
    ["0000:df:00.0"]=2   # GPU 7
)

# fmpm partition ID → set of physical IDs
declare -A PARTITION_PHYSIDS=(
    [0]="1,2,3,4,5,6,7,8"   # 8g: all GPUs
    [1]="1,2,3,4"             # 4g: NUMA 0
    [2]="5,6,7,8"             # 4g: NUMA 1
    [3]="1,3"                 # 2g pair
    [4]="2,4"                 # 2g pair
    [5]="5,7"                 # 2g pair
    [6]="6,8"                 # 2g pair
    [7]="1"  [8]="2"  [9]="3"  [10]="4"   # 1g singles
    [11]="5" [12]="6" [13]="7" [14]="8"
)

extract_gpu_bdfs() {
    # Parse <hostdev> entries from domain XML, extract GPU BDFs
    # GPUs are NVIDIA class 0x0302 (3D controller)
    echo "${DOMAIN_XML}" | xmllint --xpath \
        '//hostdev[@type="pci"]/source/address/@bus | //hostdev[@type="pci"]/source/address/@slot | //hostdev[@type="pci"]/source/address/@function' \
        - 2>/dev/null | \
    # Reconstruct BDFs from attributes
    python3 -c "
import sys, re, xml.etree.ElementTree as ET

xml = '''<root>${DOMAIN_XML}</root>'''
tree = ET.fromstring(xml.encode())
gpu_bdfs = []
for hostdev in tree.findall('.//hostdev[@type=\"pci\"]'):
    src = hostdev.find('source/address')
    if src is not None:
        domain = src.get('domain', '0x0000')
        bus = src.get('bus', '0x00')
        slot = src.get('slot', '0x00')
        func = src.get('function', '0x0')
        bdf = f'{domain}:{bus}:{slot}.{func}'.replace('0x','').lower()
        bdf = f'0000:{bdf.split(\":\")[1]}:00.0'
        # Filter to known GPU BDFs
        if bdf in [$(printf '"%s",' "${!BDF_TO_PHYSID[@]}")]:
            gpu_bdfs.append(bdf)
for b in sorted(set(gpu_bdfs)):
    print(b)
"
}

find_partition_id() {
    local gpu_bdfs="$1"
    local physids=""

    # Convert BDFs to physical IDs
    for bdf in ${gpu_bdfs}; do
        pid="${BDF_TO_PHYSID[$bdf]:-}"
        [ -n "$pid" ] && physids="${physids:+${physids},}${pid}"
    done

    # Sort physical IDs for comparison
    physids=$(echo "$physids" | tr ',' '\n' | sort -n | paste -sd',')

    # Match against known partitions
    for part_id in "${!PARTITION_PHYSIDS[@]}"; do
        expected=$(echo "${PARTITION_PHYSIDS[$part_id]}" | tr ',' '\n' | sort -n | paste -sd',')
        if [ "$physids" = "$expected" ]; then
            echo "$part_id"
            return 0
        fi
    done

    log "ERROR: no matching partition for physids=${physids}"
    return 1
}
```

### Activation and Deactivation

```bash
activate_nvlink_partition() {
    local gpu_bdfs
    gpu_bdfs=$(extract_gpu_bdfs)

    if [ -z "$gpu_bdfs" ]; then
        log "no GPU hostdevs found in domain XML, skipping NVLink"
        return 0
    fi

    local num_gpus
    num_gpus=$(echo "$gpu_bdfs" | wc -l)

    if [ "$num_gpus" -eq 1 ]; then
        log "single-GPU VM — NVLink partition not needed (partition has 0 NVLinks)"
        # Still activate the single-GPU partition so FM tracks it
    fi

    local part_id
    part_id=$(find_partition_id "$gpu_bdfs")

    log "activating NVLink partition ${part_id} for ${num_gpus} GPUs (BDFs: $(echo $gpu_bdfs | tr '\n' ' '))"

    # Call fmpm in the Service VM
    ${SERVICE_VM_SSH} "fmpm -a ${part_id}" >> "$LOG" 2>&1
    local rc=$?

    if [ $rc -ne 0 ]; then
        log "ERROR: fmpm -a ${part_id} failed (rc=${rc})"
        # Non-fatal — VM can still start, but NVLink won't work between GPUs
        # Operator should investigate
        return 0
    fi

    log "NVLink partition ${part_id} activated successfully"

    # Store partition ID for cleanup on stop
    mkdir -p /run/libvirt/hooks
    echo "${part_id}" > "/run/libvirt/hooks/${DOMAIN}.nvlink-partition"
}

deactivate_nvlink_partition() {
    local state_file="/run/libvirt/hooks/${DOMAIN}.nvlink-partition"

    if [ ! -f "$state_file" ]; then
        log "no NVLink partition state for ${DOMAIN}, skipping deactivation"
        return 0
    fi

    local part_id
    part_id=$(cat "$state_file")

    log "deactivating NVLink partition ${part_id}"
    ${SERVICE_VM_SSH} "fmpm -d ${part_id}" >> "$LOG" 2>&1 || true

    rm -f "$state_file"
    log "NVLink partition ${part_id} deactivated"
}
```

### The Lifecycle

```
virsh start tenant-vm-4g
  │
  ├── libvirt: prepare/begin
  │     ├── Hook parses XML → finds GPUs 0-3 (BDFs 1b,43,52,61)
  │     ├── Maps to physids 1,5,7,6 → sorted: 1,5,6,7 → partition 1
  │     ├── SSH to Service VM: fmpm -a 1
  │     ├── NVSwitch routing tables programmed
  │     └── Saves partition ID to /run/libvirt/hooks/tenant-vm-4g.nvlink-partition
  │
  ├── libvirt: claims VFIO devices, starts QEMU
  │
  ├── VM boots → nvidia-smi shows 4 GPUs with NVLink active
  │
  ... (workload runs) ...
  │
  ├── virsh stop tenant-vm-4g (or VM shuts down)
  │
  └── libvirt: stopped/end
        ├── Hook reads saved partition ID (1)
        ├── SSH to Service VM: fmpm -d 1
        ├── NVSwitch routing cleared
        └── Cleans up state file
```

---

## Part 2: IB Pkey Assignment via UFM

### What Are Pkeys?

InfiniBand **partition keys** (pkeys) are the IB equivalent of VLANs. Every IB port has a **pkey table** — an array of 16-bit keys that define which IB partitions (logical networks) the port belongs to:

```
Pkey table for mlx5_0 port 1:
  Index 0: 0xFFFF   (default full-member, can reach everything)
  Index 1: 0x8005   (full-member of partition 0x0005)
  Index 2: 0x0000   (empty)
  ...
```

- **0xFFFF** = default partition (full-member), present on all ports
- **0x8xxx** = full-member of partition `xxx` (can send and receive)
- **0x0xxx** = limited-member of partition `xxx` (can only respond, not initiate)
- **0x0000** = empty slot

Two ports can communicate **only if they share a pkey** (ignoring the membership bit). Pkey 0xFFFF is the global default — to isolate tenants, assign unique pkeys and remove 0xFFFF from tenant ports.

### How UFM Manages Pkeys

**UFM** (Unified Fabric Manager) is NVIDIA's IB fabric management platform. It runs the Subnet Manager (SM) and provides a REST API for programmatic fabric configuration.

The flow:
1. Each IB port has a **GUID** (Globally Unique Identifier) — a 64-bit hardware address
2. UFM maps GUIDs to pkeys via its partition management API
3. When the SM sweeps the fabric, it programs the pkey tables on each switch and HCA

### Discovering IB Port GUIDs

Before assigning pkeys, we need the GUIDs of the IB ports assigned to the VM. For **VFs** (H200 SR-IOV), the host sets GUIDs before binding to vfio-pci:

```bash
# Host-side: set GUID on VF before VM start
# PF_IFACE = host netdev name for the PF
# VF_INDEX = VF number (0-15)
# GUID = unique 8-byte identifier

ip link set ${PF_IFACE} vf ${VF_INDEX} node_guid ${NODE_GUID}
ip link set ${PF_IFACE} vf ${VF_INDEX} port_guid ${PORT_GUID}
```

For **PFs** (B200 dedicated CX-7), the GUID is burned into the hardware and readable via:

```bash
cat /sys/class/infiniband/mlx5_X/node_guid
cat /sys/class/infiniband/mlx5_X/ports/1/gids/0
```

### GUID Management in the Hook

```bash
# ── GUID scheme ────────────────────────────────────────────
# We derive deterministic GUIDs from the VM name + VF index.
# Format: aa:bb:cc:DD:DD:DD:VV:PP
#   DD:DD:DD = 24-bit hash of domain name
#   VV = VF index (00-0F)
#   PP = port (01 for single-port HCA)

generate_guid() {
    local domain="$1"
    local vf_index="$2"
    local hash
    hash=$(echo -n "$domain" | md5sum | cut -c1-6)
    local d1="${hash:0:2}" d2="${hash:2:2}" d3="${hash:4:2}"
    printf "aa:bb:cc:%s:%s:%s:%02x:01" "$d1" "$d2" "$d3" "$vf_index"
}

extract_ib_bdfs() {
    # Extract IB hostdev BDFs from domain XML
    # IB devices are class 0x0207 (InfiniBand controller) or known BDF ranges
    echo "${DOMAIN_XML}" | python3 -c "
import sys, xml.etree.ElementTree as ET

# Known IB PF BDF prefixes (H200)
IB_PFXS = {'18:', '40:', '4f:', '5e:', '9a:', 'c0:', 'ce:', 'dc:'}

xml = sys.stdin.read()
tree = ET.fromstring(xml.encode())
for hostdev in tree.findall('.//hostdev[@type=\"pci\"]'):
    src = hostdev.find('source/address')
    if src is None:
        continue
    bus = src.get('bus', '0x00').replace('0x','').lower()
    if any(bus.startswith(p.replace(':','')) for p in IB_PFXS):
        domain = src.get('domain','0x0000').replace('0x','')
        slot = src.get('slot','0x00').replace('0x','')
        func = src.get('function','0x0').replace('0x','')
        print(f'{domain}:{bus}:{slot}.{func}')
"
}

setup_vf_guids() {
    # For H200 SR-IOV: set GUIDs on VFs before VFIO claim
    local ib_bdfs="$1"
    local idx=0

    for vf_bdf in ${ib_bdfs}; do
        local guid
        guid=$(generate_guid "${DOMAIN}" ${idx})

        # Find the PF for this VF
        local pf_bdf
        pf_bdf=$(readlink -f "/sys/bus/pci/devices/${vf_bdf}/physfn" 2>/dev/null | xargs basename 2>/dev/null)

        if [ -z "$pf_bdf" ]; then
            log "  ${vf_bdf}: PF passthrough (GUID already set in hardware)"
            idx=$((idx + 1))
            continue
        fi

        # Find PF's IB device name
        local ib_dev
        ib_dev=$(ls "/sys/bus/pci/devices/${pf_bdf}/infiniband/" 2>/dev/null | head -1)

        if [ -z "$ib_dev" ]; then
            log "  WARN: PF ${pf_bdf} has no IB interface (vfio-bound?)"
            idx=$((idx + 1))
            continue
        fi

        # Find VF index on this PF
        local vf_index
        for vfdir in /sys/bus/pci/devices/${pf_bdf}/virtfn*; do
            local vf_real
            vf_real=$(readlink -f "$vfdir" | xargs basename)
            if [ "$vf_real" = "$vf_bdf" ]; then
                vf_index=$(basename "$vfdir" | sed 's/virtfn//')
                break
            fi
        done

        if [ -z "$vf_index" ]; then
            log "  WARN: can't find VF index for ${vf_bdf} on PF ${pf_bdf}"
            idx=$((idx + 1))
            continue
        fi

        # Find PF's netdev
        local pf_net
        pf_net=$(ls "/sys/bus/pci/devices/${pf_bdf}/net/" 2>/dev/null | head -1)

        if [ -n "$pf_net" ]; then
            log "  VF ${vf_bdf} (PF ${pf_bdf}/${ib_dev}, vf${vf_index}): GUID=${guid}"
            ip link set "${pf_net}" vf "${vf_index}" node_guid "${guid}" 2>/dev/null || true
            ip link set "${pf_net}" vf "${vf_index}" port_guid "${guid}" 2>/dev/null || true
        else
            # PF has no netdev — use IB tools
            log "  VF ${vf_bdf}: PF has no netdev, using sysfs GUID (already set)"
        fi

        # Store GUID for pkey assignment
        echo "${guid}" >> "/run/libvirt/hooks/${DOMAIN}.ib-guids"

        idx=$((idx + 1))
    done
}
```

### UFM REST API for Pkey Assignment

UFM exposes a REST API for partition (pkey) management. The key endpoints:

```
POST /ufmRest/resources/pkeys              # Create a pkey partition
PUT  /ufmRest/resources/pkeys/<pkey>       # Add GUIDs to a partition
GET  /ufmRest/resources/pkeys              # List all partitions
DELETE /ufmRest/resources/pkeys/<pkey>      # Remove a partition
```

### The Pkey Workflow

```bash
# ── Pkey allocation scheme ─────────────────────────────────
# Each tenant VM gets a unique pkey derived from its domain name.
# Range: 0x0002 - 0x7FFE (0x0001 is reserved, 0x7FFF is default)

allocate_pkey() {
    local domain="$1"
    local hash
    hash=$(echo -n "$domain" | md5sum | cut -c1-4)
    # Convert to 15-bit pkey (0x0002 - 0x7FFE)
    local pkey_val=$((16#${hash} % 0x7FFD + 2))
    printf "0x%04x" "$pkey_val"
}

assign_ib_pkeys() {
    local ib_bdfs
    ib_bdfs=$(extract_ib_bdfs)

    if [ -z "$ib_bdfs" ]; then
        log "no IB hostdevs found, skipping pkey assignment"
        return 0
    fi

    # Set GUIDs on VFs
    setup_vf_guids "$ib_bdfs"

    # Read the GUIDs we set
    local guid_file="/run/libvirt/hooks/${DOMAIN}.ib-guids"
    if [ ! -f "$guid_file" ]; then
        log "no GUIDs recorded, skipping UFM pkey"
        return 0
    fi

    local pkey
    pkey=$(allocate_pkey "${DOMAIN}")
    log "assigning pkey ${pkey} to $(wc -l < "$guid_file") IB ports"

    # Store pkey for cleanup
    echo "${pkey}" > "/run/libvirt/hooks/${DOMAIN}.ib-pkey"

    # Create pkey partition in UFM (idempotent)
    local guids_json
    guids_json=$(cat "$guid_file" | while read -r guid; do
        # Convert GUID format: aa:bb:cc:dd:ee:ff:00:01 → 0xaabbccddeeff0001
        local hex_guid="0x$(echo "$guid" | tr -d ':')"
        echo "\"${hex_guid}\""
    done | paste -sd',')

    # UFM REST API: create/update pkey partition
    curl -s -k -u "${UFM_USER}:${UFM_PASS}" \
        -X POST "https://${UFM_HOST}/ufmRest/resources/pkeys" \
        -H "Content-Type: application/json" \
        -d "{
            \"pkey\": \"${pkey}\",
            \"ip_over_ib\": false,
            \"membership\": \"full\",
            \"guids\": [${guids_json}],
            \"index0\": false
        }" >> "$LOG" 2>&1

    log "pkey ${pkey} assigned via UFM"
}

remove_ib_pkeys() {
    local pkey_file="/run/libvirt/hooks/${DOMAIN}.ib-pkey"
    local guid_file="/run/libvirt/hooks/${DOMAIN}.ib-guids"

    if [ ! -f "$pkey_file" ]; then
        log "no pkey state for ${DOMAIN}, skipping"
        return 0
    fi

    local pkey
    pkey=$(cat "$pkey_file")

    log "removing pkey ${pkey} from UFM"

    # Remove GUIDs from this pkey partition
    if [ -f "$guid_file" ]; then
        local guids_json
        guids_json=$(cat "$guid_file" | while read -r guid; do
            local hex_guid="0x$(echo "$guid" | tr -d ':')"
            echo "\"${hex_guid}\""
        done | paste -sd',')

        curl -s -k -u "${UFM_USER}:${UFM_PASS}" \
            -X DELETE "https://${UFM_HOST}/ufmRest/resources/pkeys/${pkey}" \
            -H "Content-Type: application/json" \
            -d "{\"guids\": [${guids_json}]}" >> "$LOG" 2>&1
    fi

    rm -f "$pkey_file" "$guid_file"
    log "pkey ${pkey} removed"
}
```

---

## UFM CLI: ufmcli for Pkey Management

### ufmcli vs REST API

For interactive debugging, `ufmcli` is the command-line interface to UFM. For automation (hooks), the REST API is better. Both do the same thing.

### Listing Pkeys and GUID Membership

```bash
# List all pkey partitions
ufmcli pkey show

# Show GUIDs in a specific pkey
ufmcli pkey show --pkey 0x0005

# Output:
# Pkey: 0x0005
# GUID                    Membership   Port
# 0xaabbccddeeff0001     full         mlx5_0/1
# 0xaabbccddeeff0101     full         mlx5_1/1
# ...
```

### Mapping GUID to Pkey

```bash
# Add a GUID to a pkey partition (full membership)
ufmcli pkey add --pkey 0x0005 --guids 0xaabbccddeeff0001 --membership full

# Add multiple GUIDs at once
ufmcli pkey add --pkey 0x0005 \
    --guids 0xaabbccddeeff0001,0xaabbccddeeff0101,0xaabbccddeeff0201,0xaabbccddeeff0301 \
    --membership full

# Remove GUIDs from a pkey
ufmcli pkey remove --pkey 0x0005 --guids 0xaabbccddeeff0001
```

### Finding Port GUIDs

```bash
# On the host — find GUID for each IB VF
for dev in /sys/class/infiniband/mlx5_*; do
    name=$(basename "$dev")
    guid=$(cat "$dev/node_guid" 2>/dev/null)
    port_guid=$(cat "$dev/ports/1/gids/0" 2>/dev/null | sed 's/://g' | tail -c 17)
    bdf=$(readlink -f "$dev/device" | xargs basename)
    echo "${name} BDF=${bdf} NODE_GUID=${guid} PORT_GUID=${port_guid}"
done

# Inside VM — find what pkeys are assigned
for dev in /sys/class/infiniband/mlx5_*; do
    name=$(basename "$dev")
    echo "=== ${name} ==="
    for p in "$dev"/ports/1/pkeys/*; do
        idx=$(basename "$p")
        val=$(cat "$p")
        [ "$val" != "0x0000" ] && echo "  pkey[${idx}] = ${val}"
    done
done
```

### Verifying Pkey Isolation

```bash
# Inside Tenant VM A (pkey 0x0005):
cat /sys/class/infiniband/mlx5_0/ports/1/pkeys/0   # 0xffff (default)
cat /sys/class/infiniband/mlx5_0/ports/1/pkeys/1   # 0x8005 (tenant A)

# Inside Tenant VM B (pkey 0x0006):
cat /sys/class/infiniband/mlx5_0/ports/1/pkeys/0   # 0xffff (default)
cat /sys/class/infiniband/mlx5_0/ports/1/pkeys/1   # 0x8006 (tenant B)

# VM A and VM B share pkey 0xffff (default) — they CAN communicate
# To fully isolate: remove 0xffff from tenant ports via UFM
```

### Full Isolation: Removing Default Pkey

The default pkey `0xFFFF` allows all ports to communicate. For strict tenant isolation:

```bash
# Remove default pkey from tenant ports
ufmcli pkey remove --pkey 0x7fff --guids <tenant_guids>

# Now ONLY the tenant's pkey is in their pkey table
# They can only communicate with ports that share the same tenant pkey
```

After removal:

```bash
# Inside Tenant VM A:
cat /sys/class/infiniband/mlx5_0/ports/1/pkeys/0   # 0x8005 (tenant A only)
cat /sys/class/infiniband/mlx5_0/ports/1/pkeys/1   # 0x0000 (empty)

# VM A can ONLY reach other ports with pkey 0x0005
```

### NCCL Pkey Configuration

When running NCCL across nodes with pkey isolation, tell NCCL and UCX which pkey to use:

```bash
# The pkey INDEX in the table, not the value
# If pkey 0x8005 is at index 1:
export NCCL_IB_PKEY=1

# UCX wants the actual pkey VALUE (without the membership bit)
export UCX_IB_PKEY=0x0005

# Verify inside the VM
cat /sys/class/infiniband/mlx5_0/ports/1/pkeys/1
# Should show 0x8005 (0x8000 = full member bit | 0x0005 = pkey value)
```

---

## Complete Hook Script

Putting it all together, here's the full `/etc/libvirt/hooks/qemu`:

```bash
#!/bin/bash
# /etc/libvirt/hooks/qemu
#
# VM lifecycle hook for NVLink partition + IB pkey management.
# Called by libvirt on: prepare/begin, started/begin, stopped/end, release/end
#
# Dependencies:
#   - SSH access to Service VM (for fmpm)
#   - curl + UFM REST API access (for pkey management)
#   - xmllint or python3 (for XML parsing)
#   - ip (iproute2) for VF GUID setting

set -euo pipefail

DOMAIN="$1"
ACTION="$2"
SUB_ACTION="${3:-}"

LOG="/var/log/libvirt/hooks/qemu-hook.log"
STATE_DIR="/run/libvirt/hooks"
mkdir -p "$(dirname "$LOG")" "$STATE_DIR"

# ── Service VM + UFM config ───────────────────────────────
SERVICE_VM_IP="192.168.122.10"
SERVICE_VM_SSH="ssh -o ConnectTimeout=5 -o StrictHostKeyChecking=no root@${SERVICE_VM_IP}"
UFM_HOST="ufm-mgmt.cluster.local"
UFM_USER="admin"
UFM_PASS="admin"
# ──────────────────────────────────────────────────────────

log() { echo "$(date -Iseconds) [${DOMAIN}] $*" >> "$LOG"; }

# Read domain XML from stdin
DOMAIN_XML=$(cat)

# ── GPU BDF → Physical ID mapping ─────────────────────────
declare -A BDF_TO_PHYSID=(
    ["0000:1b:00.0"]=1  ["0000:43:00.0"]=5
    ["0000:52:00.0"]=7  ["0000:61:00.0"]=6
    ["0000:9d:00.0"]=4  ["0000:c3:00.0"]=1
    ["0000:d1:00.0"]=3  ["0000:df:00.0"]=2
)

# ── Partition ID → Physical ID set ─────────────────────────
declare -A PARTITION_PHYSIDS=(
    [0]="1,2,3,4,5,6,7,8"  [1]="1,2,3,4"  [2]="5,6,7,8"
    [3]="1,3" [4]="2,4" [5]="5,7" [6]="6,8"
    [7]="1" [8]="2" [9]="3" [10]="4"
    [11]="5" [12]="6" [13]="7" [14]="8"
)

get_gpu_physids() {
    echo "${DOMAIN_XML}" | python3 -c "
import sys, xml.etree.ElementTree as ET
BDF_MAP = {
    '1b': 1, '43': 5, '52': 7, '61': 6,
    '9d': 4, 'c3': 1, 'd1': 3, 'df': 2,
}
tree = ET.fromstring(sys.stdin.read())
pids = set()
for hd in tree.findall('.//{http://www.w3.org/1999/xhtml}hostdev') or tree.findall('.//hostdev'):
    src = hd.find('.//source/address') or hd.find('source/address')
    if src is None: continue
    bus = src.get('bus','').replace('0x','').lower()
    if bus in BDF_MAP:
        pids.add(BDF_MAP[bus])
print(','.join(str(p) for p in sorted(pids)))
" 2>/dev/null || true
}

match_partition() {
    local physids="$1"
    for pid in "${!PARTITION_PHYSIDS[@]}"; do
        local expect
        expect=$(echo "${PARTITION_PHYSIDS[$pid]}" | tr ',' '\n' | sort -n | paste -sd',')
        if [ "$physids" = "$expect" ]; then
            echo "$pid"
            return 0
        fi
    done
    return 1
}

do_prepare() {
    # ── NVLink partition ──
    local physids
    physids=$(get_gpu_physids)
    if [ -n "$physids" ]; then
        local part_id
        if part_id=$(match_partition "$physids"); then
            log "activating NVLink partition ${part_id} (physids: ${physids})"
            if ${SERVICE_VM_SSH} "fmpm -a ${part_id}" >> "$LOG" 2>&1; then
                echo "${part_id}" > "${STATE_DIR}/${DOMAIN}.nvlink"
                log "NVLink partition ${part_id} activated"
            else
                log "WARN: fmpm -a ${part_id} failed"
            fi
        else
            log "WARN: no matching partition for physids=${physids}"
        fi
    fi

    # ── IB pkey ──
    # Derive pkey from domain name
    local pkey_val
    pkey_val=$(printf "%d" "0x$(echo -n "${DOMAIN}" | md5sum | cut -c1-4)")
    pkey_val=$(( (pkey_val % 0x7FFD) + 2 ))
    local pkey
    pkey=$(printf "0x%04x" "$pkey_val")

    # Collect GUIDs from IB hostdevs (read from sysfs before VFIO claim)
    local guids=""
    for bdf in $(echo "${DOMAIN_XML}" | grep -oP 'bus="0x\K[0-9a-f]+' | sort -u); do
        local full_bdf="0000:${bdf}:00.0"
        local ib_dir="/sys/bus/pci/devices/${full_bdf}/infiniband"
        [ -d "$ib_dir" ] || continue
        for ibdev in "$ib_dir"/*/; do
            local guid
            guid=$(cat "${ibdev}/node_guid" 2>/dev/null) || continue
            [ -n "$guid" ] && guids="${guids:+${guids},}${guid}"
        done
    done

    if [ -n "$guids" ]; then
        log "assigning pkey ${pkey} to GUIDs: ${guids}"
        echo "${pkey}" > "${STATE_DIR}/${DOMAIN}.pkey"
        echo "${guids}" > "${STATE_DIR}/${DOMAIN}.guids"

        # UFM REST call
        local guids_json
        guids_json=$(echo "$guids" | tr ',' '\n' | while read -r g; do
            echo "\"0x$(echo "$g" | tr -d ':')\"";
        done | paste -sd',')

        curl -s -k -u "${UFM_USER}:${UFM_PASS}" \
            -X POST "https://${UFM_HOST}/ufmRest/resources/pkeys" \
            -H "Content-Type: application/json" \
            -d "{\"pkey\":\"${pkey}\",\"guids\":[${guids_json}],\"membership\":\"full\",\"ip_over_ib\":false}" \
            >> "$LOG" 2>&1 || log "WARN: UFM pkey assignment failed"

        log "pkey ${pkey} assigned"
    fi
}

do_stopped() {
    # ── NVLink cleanup ──
    if [ -f "${STATE_DIR}/${DOMAIN}.nvlink" ]; then
        local part_id
        part_id=$(cat "${STATE_DIR}/${DOMAIN}.nvlink")
        log "deactivating NVLink partition ${part_id}"
        ${SERVICE_VM_SSH} "fmpm -d ${part_id}" >> "$LOG" 2>&1 || true
        rm -f "${STATE_DIR}/${DOMAIN}.nvlink"
    fi

    # ── Pkey cleanup ──
    if [ -f "${STATE_DIR}/${DOMAIN}.pkey" ]; then
        local pkey guids_json
        pkey=$(cat "${STATE_DIR}/${DOMAIN}.pkey")
        if [ -f "${STATE_DIR}/${DOMAIN}.guids" ]; then
            guids_json=$(cat "${STATE_DIR}/${DOMAIN}.guids" | tr ',' '\n' | while read -r g; do
                echo "\"0x$(echo "$g" | tr -d ':')\"";
            done | paste -sd',')

            curl -s -k -u "${UFM_USER}:${UFM_PASS}" \
                -X DELETE "https://${UFM_HOST}/ufmRest/resources/pkeys/${pkey}" \
                -H "Content-Type: application/json" \
                -d "{\"guids\":[${guids_json}]}" \
                >> "$LOG" 2>&1 || true
        fi
        rm -f "${STATE_DIR}/${DOMAIN}.pkey" "${STATE_DIR}/${DOMAIN}.guids"
        log "pkey ${pkey} removed"
    fi
}

case "${ACTION}/${SUB_ACTION}" in
    prepare/begin) do_prepare ;;
    stopped/end)   do_stopped ;;
    *)             : ;;  # no-op for other phases
esac

exit 0
```

### Installing the Hook

```bash
# Copy to libvirt hooks directory
cp qemu-hook.sh /etc/libvirt/hooks/qemu
chmod 755 /etc/libvirt/hooks/qemu

# Restart libvirtd to pick up the hook
systemctl restart libvirtd

# Test (dry-run — check the log)
virsh start tenant-vm 2>&1
tail -f /var/log/libvirt/hooks/qemu-hook.log
```

---

## Troubleshooting

### Hook Not Firing

```bash
# Verify hook exists and is executable
ls -la /etc/libvirt/hooks/qemu
# Must be: -rwxr-xr-x root root /etc/libvirt/hooks/qemu

# Check libvirtd logs
journalctl -u libvirtd --since '5 min ago' | grep -i hook
```

### fmpm Connection Fails from Hook

```bash
# Test SSH from host to Service VM
ssh -o ConnectTimeout=5 root@192.168.122.10 "fmpm -q" | head -5

# Common issue: SSH key not available to libvirtd
# Fix: copy root's SSH key and ensure libvirtd runs as root
# Or use password-less sudo + expect wrapper
```

### UFM API Errors

```bash
# Test UFM connectivity
curl -s -k -u admin:admin https://ufm-mgmt.cluster.local/ufmRest/resources/pkeys | python3 -m json.tool

# Common errors:
# 401 = bad credentials
# 404 = UFM not running or wrong endpoint
# 409 = pkey already exists (idempotent — not an error)
```

### Pkey Not Visible in VM

```bash
# Inside VM — check pkey table
for dev in /sys/class/infiniband/mlx5_*; do
    echo "=== $(basename $dev) ==="
    for p in "$dev"/ports/1/pkeys/*; do
        val=$(cat "$p")
        [ "$val" != "0x0000" ] && echo "  [$(basename $p)] = $val"
    done
done

# If empty: SM hasn't swept yet, or GUID doesn't match what UFM expects
# Force SM sweep:
# On SM node: opensm -f /tmp/osm.log (or via UFM UI)
```

### Wrong Pkey Index for NCCL

```bash
# NCCL_IB_PKEY is the TABLE INDEX, not the pkey value
# Find the right index:
cat /sys/class/infiniband/mlx5_0/ports/1/pkeys/* | grep -n 'your_pkey_value'
# The line number minus 1 = the index
# Or scan:
for i in /sys/class/infiniband/mlx5_0/ports/1/pkeys/*; do
    val=$(cat "$i")
    echo "  [$(basename $i)] = $val"
done | grep -v 0x0000
```

---

## Summary

| Layer | Tool | What It Does | When |
|-------|------|-------------|------|
| NVLink | fmpm (Service VM) | Programs NVSwitch routing tables | VM prepare/begin |
| IB VF GUID | `ip link set ... vf ... guid` | Sets port identity before VFIO claim | VM prepare/begin |
| IB pkey | UFM REST API / ufmcli | Assigns fabric partition key to GUIDs | VM prepare/begin |
| NVLink cleanup | fmpm -d | Deactivates partition | VM stopped/end |
| IB pkey cleanup | UFM REST DELETE | Removes GUIDs from pkey | VM stopped/end |

The libvirt QEMU hook ties these together into a single automated lifecycle:

1. `virsh start tenant-vm` triggers `prepare/begin`
2. Hook parses XML, activates NVLink partition via Service VM, assigns IB pkeys via UFM
3. VM boots with NVLink and IB isolation already configured
4. `virsh stop tenant-vm` triggers `stopped/end`
5. Hook deactivates partition and removes pkeys

No manual steps. No operator intervention. GPU and IB isolation is enforced from the moment the VM starts until the moment it stops.
