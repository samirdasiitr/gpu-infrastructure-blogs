---
layout: post
title: "Running Baremetal NCCL All-Reduce Benchmarks on DGX/HGX H200"
date: 2026-04-22 00:00:00 +0000
categories: gpu infra nccl benchmark
tags: nccl cuda hpcx mpi infiniband dgx h200 allreduce
---

## Introduction

NCCL (NVIDIA Collective Communications Library) all-reduce is the definitive benchmark for multi-node GPU training performance. It measures how fast your GPUs can aggregate gradients across nodes — the operation that dominates distributed training wall-clock time.

This guide walks through setting up and running NCCL all-reduce benchmarks on DGX H200 / HGX H200 systems from scratch — no SLURM, no container runtime, just bare metal (or bare VM) with CUDA, HPC-X, and NCCL built from source.

**Target result:** 380+ GB/s bus bandwidth across 2 nodes × 8 GPUs with 400 Gb/s NDR InfiniBand.

---

## Prerequisites

A working multi-node GPU cluster with:
- NVIDIA H200 (or H100/A100) GPUs with NVLink/NVSwitch
- InfiniBand NDR (400 Gb/s) or HDR (200 Gb/s) fabric
- NVIDIA driver 580+ installed
- SSH passwordless access between nodes
- A shared filesystem (NFS, Lustre, or virtiofs) mounted at the same path on all nodes

Verify before proceeding:

```bash
# GPUs visible
nvidia-smi | head -5

# NVLink active
nvidia-smi nvlink --status -i 0 | head -5

# IB ports active
ibstat | grep -E 'State|Rate'

# SSH works between nodes
ssh peer-node hostname
```

---

## Step 1: Install CUDA Toolkit

We install CUDA locally (not system-wide) so multiple versions can coexist:

```bash
topdir=/mnt/benchmark/baremetal-nccl

mkdir -p $topdir && cd $topdir

# Download CUDA 13.0 (matches driver 580.x)
# For other versions: https://developer.nvidia.com/cuda-toolkit-archive
curl -O https://developer.download.nvidia.com/compute/cuda/13.0.0/local_installers/cuda_13.0.0_580.65.06_linux.run

# Install toolkit only (no driver, no OpenGL)
bash cuda_13.0.0_580.65.06_linux.run \
  --silent --toolkit --no-opengl-libs --no-drm \
  --installpath=${topdir}/cuda

# Create environment script
cat > ${topdir}/cuda/env.sh << EOF
#!/bin/bash
export CUDA_HOME=${topdir}/cuda
export PATH=\${CUDA_HOME}/bin:\$PATH
export LD_LIBRARY_PATH=\${CUDA_HOME}/lib64:\${LD_LIBRARY_PATH}
EOF
chmod +x ${topdir}/cuda/env.sh

# Verify
source ${topdir}/cuda/env.sh
nvcc --version
```

**Why local install?** System CUDA packages (`cuda-toolkit-12-x`) conflict with driver versions and require root. A local install lives in one directory, is trivially reproducible, and can be shared via NFS to all nodes.

---

## Step 2: Install HPC-X (MPI + UCX + RDMA)

HPC-X is NVIDIA's optimized MPI distribution that includes OpenMPI, UCX (Unified Communication X), HCOLL, and SHARP — everything needed for multi-node GPU communication.

```bash
cd $topdir

# Download HPC-X matching your CUDA version
# Browse: https://developer.nvidia.com/networking/hpc-x
wget https://content.mellanox.com/hpc/hpc-x/v2.26_cuda13/hpcx-v2.26-gcc-doca_ofed-ubuntu24.04-cuda13-x86_64.tbz

tar -xf hpcx-v2.26-gcc-doca_ofed-ubuntu24.04-cuda13-x86_64.tbz

# Load HPC-X environment (multi-threaded variant for GPU workloads)
source ${topdir}/hpcx-v2.26-gcc-doca_ofed-ubuntu24.04-cuda13-x86_64/hpcx-mt-init-ompi.sh
hpcx_load

# Verify
mpirun --version
ucx_info -v | head -3
```

**Why `hpcx-mt` (multi-threaded)?** GPU workloads use CUDA streams from multiple threads. The `-mt` build of UCX enables `MPI_THREAD_MULTIPLE` support, required for NCCL's internal threading.

### HPC-X Components

| Component | Purpose |
|-----------|---------|
| `ompi4/` | OpenMPI 4.x with CUDA-aware transport |
| `ucx/mt/` | UCX multi-threaded — RC/DC verbs for IB |
| `hcoll/` | Hierarchical collectives (optional, often disabled for NCCL) |
| `sharp/` | Scalable Hierarchical Aggregation (switch-level reduction) |
| `nccl_rdma_sharp_plugin/` | NCCL plugin for RDMA + SHARP |

---

## Step 3: Build NCCL from Source

Building NCCL from source ensures you get the latest optimizations and can target specific GPU architectures:

```bash
cd $topdir
source ${topdir}/cuda/env.sh

# Clone NCCL
git clone https://github.com/NVIDIA/nccl.git
cd nccl

# Build for H200 (sm_90) and B200 (sm_100)
make -j$(nproc) src.build \
  CUDA_HOME=$CUDA_HOME \
  NVCC_GENCODE="-gencode=arch=compute_90,code=sm_90 \
                -gencode=arch=compute_100,code=sm_100"
```

**Why build from source?** The packaged `libnccl2` from apt is often months behind. Source builds include the latest ring/tree algorithm fixes, GDR improvements, and NVSwitch optimizations.

### GPU Architecture Codes

| GPU | Compute Capability | NVCC Flag |
|-----|-------------------|-----------|
| A100 | sm_80 | `-gencode=arch=compute_80,code=sm_80` |
| H100/H200 | sm_90 | `-gencode=arch=compute_90,code=sm_90` |
| B200 | sm_100 | `-gencode=arch=compute_100,code=sm_100` |

---

## Step 4: Build NCCL Tests

```bash
cd $topdir/nccl

# Clone test suite
git clone https://github.com/NVIDIA/nccl-tests.git
cd nccl-tests

# Build with MPI support, pointing to our NCCL build
make -j$(nproc) \
  CUDA_HOME=$CUDA_HOME \
  NVCC_GENCODE="-gencode=arch=compute_90,code=sm_90 \
                -gencode=arch=compute_100,code=sm_100" \
  MPI=1 \
  MPI_HOME=${OMPI_HOME} \
  NCCL_HOME=$PWD/../build

# Organize binaries
cp -arv build/ ../bins
cd ../bins/
rm -fr *.o verifiable
cp -arv ../build/lib/libnccl*so* .

# Verify
ldd all_reduce_perf | grep -E 'nccl|cuda|mpi'
```

The `bins/` directory now contains everything needed:
```
bins/
├── all_reduce_perf      # main benchmark
├── all_gather_perf      # all-gather benchmark
├── broadcast_perf       # broadcast benchmark
├── reduce_scatter_perf  # reduce-scatter benchmark
├── libnccl.so.2         # NCCL library
└── libnccl.so.2.x.y     # NCCL library (versioned)
```

---

## Step 5: Environment Setup Script

Create a single script that loads everything:

```bash
cat > ${topdir}/env.sh << 'ENVSCRIPT'
#!/bin/bash
topdir=/mnt/benchmark/baremetal-nccl

# CUDA
source ${topdir}/cuda/env.sh

# HPC-X (MPI + UCX)
source ${topdir}/hpcx-v2.26-gcc-doca_ofed-ubuntu24.04-cuda13-x86_64/hpcx-mt-init-ompi.sh
hpcx_load

# NCCL
export NCCL_HOME=${topdir}/nccl
export PATH=${NCCL_HOME}/bins:$PATH
export LD_LIBRARY_PATH=${NCCL_HOME}/bins:$LD_LIBRARY_PATH
ENVSCRIPT
chmod +x ${topdir}/env.sh
```

Source it on every node before running tests:

```bash
source /mnt/benchmark/baremetal-nccl/env.sh
```

---

## Step 6: Discover IB Devices

Before running NCCL, identify which IB devices are available. Device names (`mlx5_N`) vary between nodes:

```bash
# List all IB devices with status
ibdev2netdev

# Filter to active InfiniBand devices (exclude Ethernet)
for d in /sys/class/infiniband/mlx5_*; do
  NAME=$(basename $d)
  LAYER=$(cat $d/ports/1/link_layer 2>/dev/null)
  STATE=$(cat $d/ports/1/state 2>/dev/null)
  RATE=$(cat $d/ports/1/rate 2>/dev/null)
  [ "$LAYER" = "InfiniBand" ] && echo "$NAME: $STATE $RATE"
done

# Find the IB partition key (pkey)
for d in /sys/class/infiniband/mlx5_*; do
  cat $d/ports/1/pkeys/* 2>/dev/null | grep -vE '0x0000|0x7fff|0xffff' && echo $(basename $d)
done
```

### Auto-Detection Script

Since device names differ between nodes, auto-detect at runtime:

```bash
# Auto-detect IB devices (works on any node)
IB_DEVS=""
UCX_DEVS=""
for d in /sys/class/infiniband/mlx5_*; do
  [ "$(cat $d/ports/1/link_layer 2>/dev/null)" = "InfiniBand" ] || continue
  NAME=$(basename $d)
  IB_DEVS="${IB_DEVS:+$IB_DEVS,}$NAME"
  UCX_DEVS="${UCX_DEVS:+$UCX_DEVS,}${NAME}:1"
done
echo "NCCL_IB_HCA=$IB_DEVS"
echo "UCX_NET_DEVICES=$UCX_DEVS"
```

---

## Step 7: Single-Node Test

Start with a single-node test to verify GPU-to-GPU NVLink bandwidth:

```bash
source /mnt/benchmark/baremetal-nccl/env.sh

# Single node, 8 GPUs, all-reduce
all_reduce_perf -b 8 -e 16G -f 2 -g 8

# Expected output for H200 with NVSwitch:
#   16G messages: busbw ~450-480 GB/s (NVLink bandwidth)
```

**Key flags:**
- `-b 8` — start message size (8 bytes)
- `-e 16G` — end message size (16 GiB)
- `-f 2` — factor (double size each step)
- `-g 8` — 8 GPUs per process (single-node mode)

---

## Step 8: Multi-Node All-Reduce

### The Full Benchmark Script

```bash
#!/bin/bash
source /mnt/benchmark/baremetal-nccl/env.sh

# Define nodes
hosts=(dgx-vm1 dgx-vm2)
echo "HOSTS=$(echo ${hosts[*]} | tr ' ' ',')"

# Iterations
iter="--iters 20"

set -x

mpirun --allow-run-as-root \
  --mca pml ucx \
  --mca coll ^hcoll \
  --mca btl ^openib,smcuda \
  --map-by ppr:8:node \
  --bind-to none \
  --display-map --report-bindings \
  -x PATH=$PATH \
  -x LD_LIBRARY_PATH=$LD_LIBRARY_PATH \
  \
  -x OMPI_MCA_pml=ucx \
  -x OMPI_MCA_btl=^openib,smcuda \
  -x OMPI_MCA_btl_openib_warn_default_gid_prefix=0 \
  -x OMPI_MCA_coll_hcoll_enable=0 \
  -x OMPI_MCA_btl_tcp_if_include=enp2s0 \
  \
  -x UCX_TLS=rc_x,cuda \
  -x UCX_NET_DEVICES="mlx5_0:1,mlx5_1:1,mlx5_2:1,mlx5_3:1,mlx5_4:1,mlx5_5:1,mlx5_6:1,mlx5_7:1" \
  \
  -x NCCL_SOCKET_IFNAME=enp2s0 \
  -x NCCL_IB_HCA="mlx5_0,mlx5_1,mlx5_2,mlx5_3,mlx5_4,mlx5_5,mlx5_6,mlx5_7" \
  -x NCCL_DEBUG=info \
  -x NCCL_NET_GDR_LEVEL=5 \
  -x NCCL_NET_GDR_READ=1 \
  -x NCCL_CROSS_NIC=1 \
  -x NCCL_MIN_NCHANNELS=16 \
  -x NCCL_IB_QPS_PER_CONNECTION=8 \
  -x NCCL_IB_SPLIT_DATA_ON_QPS=0 \
  -x NCCL_PXN_DISABLE=0 \
  -x NCCL_COLLNET_ENABLE=0 \
  -x CUDA_MODULE_LOADING=EAGER \
  \
  -H $(for i in ${hosts[*]}; do echo ${i}:8; done | paste -s -d',') \
  -np $((8 * ${#hosts[*]})) \
  all_reduce_perf -b 8 -f 2 -g 1 -e 16G ${iter}
```

### With IB Partition Key (Pkey)

If your fabric uses IB partition keys for tenant isolation:

```bash
# Find your pkey
for d in /sys/class/infiniband/mlx5_*; do
  cat $d/ports/1/pkeys/* 2>/dev/null | grep -vE '0x0000|0x7fff|0xffff'
done
# Example output: 0x8003 (full membership) or 0x0003 (limited)

# Add to mpirun:
  -x UCX_IB_PKEY=0x0003 \
  -x NCCL_IB_PKEY=1 \       # pkey TABLE INDEX, not the key value
```

### With Auto-Detection (Different Nodes)

When nodes have different `mlx5_N` numbering:

```bash
mpirun --allow-run-as-root \
  ... \
  -H node1:8,node2:8 \
  -np 16 \
  bash -c '
    # Auto-detect IB devices per node
    IB="" UCX_D=""
    for d in /sys/class/infiniband/mlx5_*; do
      [ "$(cat $d/ports/1/link_layer 2>/dev/null)" = "InfiniBand" ] || continue
      N=$(basename $d)
      IB="${IB:+$IB,}$N"
      UCX_D="${UCX_D:+$UCX_D,}${N}:1"
    done
    export NCCL_IB_HCA="$IB"
    export UCX_NET_DEVICES="$UCX_D"
    echo "[$(hostname)] NCCL_IB_HCA=$NCCL_IB_HCA"
    exec all_reduce_perf -b 8 -f 2 -g 1 -e 16G --iters 20
  '
```

---

## Understanding the Parameters

### MPI Parameters

| Parameter | Purpose |
|-----------|---------|
| `--mca pml ucx` | Use UCX for MPI point-to-point (IB verbs) |
| `--mca coll ^hcoll` | Disable HCOLL collectives (NCCL handles collectives) |
| `--mca btl ^openib,smcuda` | Disable OpenIB and SM+CUDA byte transfer layers |
| `--map-by ppr:8:node` | Place 8 processes per node (1 per GPU) |
| `--bind-to none` | Don't bind processes to CPUs (NCCL manages affinity) |
| `-x VAR=val` | Export environment variable to all ranks |

### UCX Parameters

| Parameter | Purpose |
|-----------|---------|
| `UCX_TLS=rc_x,cuda` | Use accelerated RC transport + CUDA memory |
| `UCX_NET_DEVICES=mlx5_*:1` | Which IB devices and ports to use |
| `UCX_IB_PKEY=0xNNNN` | IB partition key (if fabric uses pkeys) |

### NCCL Parameters

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `NCCL_IB_HCA` | `mlx5_0,...` | Which IB HCAs to use |
| `NCCL_SOCKET_IFNAME` | `enp2s0` | TCP interface for bootstrap/control |
| `NCCL_DEBUG` | `info` | Log level (info/warn/trace) |
| `NCCL_NET_GDR_LEVEL` | `5` (SYS) | Enable GDR at any topology distance |
| `NCCL_NET_GDR_READ` | `1` | Enable GDR for read operations |
| `NCCL_CROSS_NIC` | `1` | Allow traffic across non-affine NICs |
| `NCCL_MIN_NCHANNELS` | `16` | Minimum parallel communication channels |
| `NCCL_IB_QPS_PER_CONNECTION` | `8` | Queue pairs per IB connection |
| `NCCL_IB_SPLIT_DATA_ON_QPS` | `0` | Don't split data across QPs |
| `NCCL_PXN_DISABLE` | `0` | Enable PXN (Proxy through NVLink) |
| `NCCL_COLLNET_ENABLE` | `0` | Disable CollNet/SHARP (use direct IB) |
| `NCCL_IB_PKEY` | `1` | Pkey table index (not the key value!) |
| `CUDA_MODULE_LOADING` | `EAGER` | Load all CUDA modules at init (faster first kernel) |

### NCCL Test Parameters

| Flag | Purpose |
|------|---------|
| `-b 8` | Start message size (bytes) |
| `-e 16G` | End message size |
| `-f 2` | Multiply size by 2 each step |
| `-g 1` | 1 GPU per MPI process |
| `--iters 20` | Iterations per message size |

---

## Step 9: Reading the Output

```
#       size    count   type   redop    root    time  algbw  busbw  #wrong
         8         2  float     sum      -1    37.2   0.00   0.00       0
        16         4  float     sum      -1    36.8   0.00   0.00       0
       ...
  8589934592  2147483648  float  sum    -1  23145  371.22  696.04      0
 17179869184  4294967296  float  sum    -1  45892  374.37  702.07      0
```

**Key columns:**
- **algbw** (GB/s) — Algorithm bandwidth: `data_size / time`
- **busbw** (GB/s) — Bus bandwidth: normalized for the collective operation

**Bus bandwidth formula for all-reduce:**

```
busbw = algbw × 2 × (n-1) / n
```

Where `n` = total number of GPUs. For 16 GPUs: `busbw = algbw × 1.875`

### Expected Results

| Setup | Large Message busbw | What limits it |
|-------|-------------------|----------------|
| 1 node, 8 GPU (NVLink only) | ~450-480 GB/s | NVSwitch bandwidth |
| 2 nodes, 16 GPU (8×NDR400) | ~370-400 GB/s | IB network bandwidth |
| 2 nodes, 16 GPU (8×HDR200) | ~180-200 GB/s | IB network bandwidth |
| 2 nodes, 16 GPU (no GDR) | ~150-190 GB/s | CPU staging overhead |

---

## Step 10: Enable GPUDirect RDMA (GDR)

GDR allows GPUs to DMA directly to/from InfiniBand NICs without CPU involvement. This is critical for achieving full IB bandwidth.

```bash
# Load the peermem kernel module
modprobe nvidia_peermem    # newer systems
# or
modprobe nv_peer_mem       # older systems

# Verify
lsmod | grep -iE 'peer|gdr'
cat /sys/module/nvidia_peermem/version 2>/dev/null || \
cat /sys/module/nv_peer_mem/version 2>/dev/null

# NCCL should now show GDR 1:
# Look for: "Connected all rings, use ring PXN 0 GDR 1"
```

**If GDR shows 0 despite peermem loaded:**

```bash
# Check NCCL_NET_GDR_LEVEL — must be high enough for your topology
export NCCL_NET_GDR_LEVEL=5     # 5 = SYS = allow GDR everywhere

# Check for topology file issues
export NCCL_TOPO_FILE=/path/to/topo.xml  # or unset to let NCCL auto-detect

# Verify with raw RDMA test (bypasses NCCL)
ib_write_bw -d mlx5_0 --use_cuda=0 -s 4194304 -n 5000 &
sleep 2
ib_write_bw -d mlx5_0 --use_cuda=0 -s 4194304 -n 5000 localhost
# If this shows ~20-48 GB/s, GDR works at hardware level
```

---

## Step 11: NCCL Topology File

For VMs where `nvidia-smi topo -m` doesn't show the correct GPU-NIC affinity, create a topology file:

```xml
<!-- /etc/nccl-topo.xml -->
<system version="1">
  <cpu numaid="0" affinity="ffffffff,ffffffff" arch="x86_64" vendor="GenuineIntel">
    <pci busid="0000:06:00.0" class="0x030200" link_speed="32.0 GT/s PCIe" link_width="16">
      <gpu dev="0" sm="90" mem="141312" gdr="1"/>
    </pci>
    <pci busid="0000:0e:00.0" class="0x020700" link_speed="32.0 GT/s PCIe" link_width="16">
      <nic name="mlx5_0" dev="0" speed="400" port="1" gdr="1"/>
    </pci>
    <!-- ... repeat for all GPU+NIC pairs ... -->
  </cpu>
  <nvswitches>
    <nvswitch link_bw="900"/>
  </nvswitches>
</system>
```

```bash
export NCCL_TOPO_FILE=/etc/nccl-topo.xml
```

See our [GPU and IB Passthrough topology guide](/gpu-blogs/gpu/infra/nccl/2025/04/21/gpu-ib-passthrough.html) for the full topology file format.

---

## Troubleshooting

### "PML ucx cannot be selected"

UCX can't find IB devices. The `mlx5_N` names don't match:

```bash
# Check actual device names
ibv_devinfo | grep hca_id
# Update UCX_NET_DEVICES to match
```

### "not enough slots available"

MPI slots don't match `--map-by`:

```bash
# If using --map-by ppr:8:node, hosts need :8 slots
-H node1:8,node2:8 -np 16

# Or use --bind-to none --map-by ppr:8:node
```

### Low Bandwidth (< 200 GB/s for 2-node NDR400)

1. Check GDR: look for `GDR 1` in NCCL output
2. Check all 8 NICs are used: `NCCL_DEBUG=INFO ... | grep NET/`
3. Check IB rates: `ibstat | grep Rate` (should be 400 Gb/s)
4. Check NVLink: `nvidia-smi nvlink --status -i 0`

### "UCX WARN transport 'rc_x' is not available"

The IB VFs don't support accelerated RC. Use verbs RC:

```bash
export UCX_TLS=rc_v,cuda_copy,cuda_ipc
# instead of
export UCX_TLS=rc_x,cuda
```

---

## Complete Directory Structure

After setup, your benchmark directory looks like:

```
/mnt/benchmark/baremetal-nccl/
├── env.sh                          # Source this first
├── cuda/                           # CUDA Toolkit
│   ├── bin/nvcc
│   ├── lib64/
│   └── env.sh
├── hpcx-v2.26-.../                 # HPC-X (MPI + UCX)
│   ├── ompi4/
│   ├── ucx/mt/
│   └── hpcx-mt-init-ompi.sh
├── nccl/                           # NCCL source + build
│   ├── build/lib/libnccl.so.2
│   ├── bins/                       # Test binaries + libs
│   │   ├── all_reduce_perf
│   │   ├── all_gather_perf
│   │   └── libnccl.so.2
│   └── nccl-tests/
└── run-nccl.sh                     # Your benchmark script
```

---

## Conclusion

The baremetal NCCL benchmark stack is straightforward:

1. **CUDA Toolkit** — local install, no system packages
2. **HPC-X** — provides optimized MPI + UCX with IB support
3. **NCCL from source** — latest algorithms, target your GPU arch
4. **nccl-tests** — the standard `all_reduce_perf` benchmark

The critical runtime variables are:
- `NCCL_IB_HCA` — which NICs to use (auto-detect if nodes differ)
- `NCCL_NET_GDR_LEVEL=5` — enable GPUDirect RDMA
- `nvidia_peermem` module — kernel support for GDR
- `UCX_TLS=rc_x,cuda` — use accelerated RC transport

With 8× NDR400 InfiniBand and GDR enabled, expect **370-400 GB/s bus bandwidth** on the 16 GiB all-reduce. If you're seeing less, check GDR status, NIC count, and IB link rates in that order.
