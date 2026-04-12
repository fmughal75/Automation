# 🚀 AI GPU Cluster Design Guide
> InfiniBand vs RoCEv2 (Arista) — End-to-End Architecture Reference

---

## Table of Contents

1. [Reference Architecture (2000 GPU Cluster)](#1-reference-architecture-2000-gpu-cluster)
2. [InfiniBand vs RoCEv2 — Decision Framework](#2-infiniband-vs-rocev2--decision-framework)
3. [InfiniBand Architecture](#3-infiniband-architecture)
4. [RoCEv2 Architecture (Arista)](#4-rocev2-architecture-arista)
5. [Network Configuration](#5-network-configuration)
6. [Congestion Control](#6-congestion-control)
7. [Monitoring & Observability](#7-monitoring--observability)
8. [Failure Modes & Remediation](#8-failure-modes--remediation)
9. [Performance Benchmarking](#9-performance-benchmarking)
10. [Deployment Checklist](#10-deployment-checklist)

---

## 1. Reference Architecture (2000 GPU Cluster)

### Topology: 3-Tier Clos (Fat-Tree)

```
                        [ SUPER SPINE ]
                      /    /    |    \    \
              [ SPINE LAYER — 400G Links ]
             /    /    /    \    \    \
         [ LEAF / ToR SWITCHES ]
            |    |    |    |    |    |
         [ GPU SERVERS — 8x GPU each ]
```

### Cluster Sizing

| Parameter | Value |
|---|---|
| Total GPU Servers | ~250 |
| GPUs per Server | 8 |
| Total GPUs | 2,000 |
| NIC Config | Dual-homed |
| Oversubscription | 1:1 (non-blocking) |
| Inter-node BW | 400 Gbps |
| GPU-to-NIC BW | HDR 200G / NDR 400G / 400GbE |

### Tier Breakdown

```
Tier 1 — Super Spine
  └─ Provides east-west bandwidth between spine pods
  └─ 400G ports, typically 8–16 switches

Tier 2 — Spine
  └─ Aggregates leaf layer
  └─ 64×400G radix switches
  └─ ECMP across all uplinks

Tier 3 — Leaf / ToR
  └─ 1 ToR per server rack (~6–8 servers per rack)
  └─ Dual-homed servers for redundancy
  └─ Breakout cables: 1×400G → 4×100G to servers
```

---

## 2. InfiniBand vs RoCEv2 — Decision Framework

### Feature Comparison

| Criteria | InfiniBand | RoCEv2 (Arista) |
|---|---|---|
| **Latency** | Ultra-low (~600ns) | Low (~1–2µs, tuned) |
| **Bandwidth** | HDR 200G / NDR 400G | 400GbE |
| **Congestion Control** | Native (credit-based) | Requires ECN + DCQCN |
| **Operational Complexity** | Medium | High (tuning-heavy) |
| **Flexibility** | Low (vendor lock-in) | High (open Ethernet) |
| **Ecosystem** | NVIDIA / Mellanox | Open / Multi-vendor |
| **RDMA Support** | Native | RoCEv2 (layer 3 RDMA) |
| **Multitenancy** | Limited | Strong |
| **Cost** | High (proprietary) | Lower (commodity HW) |
| **Best Use Case** | Pure AI Training | Hybrid DC + AI |

### Decision Tree

```
Is workload purely large-scale AI training?
  ├─ YES → Is budget unconstrained?
  │          ├─ YES → InfiniBand (HDR/NDR)
  │          └─ NO  → RoCEv2 on Arista
  └─ NO  → Do you need DC multi-tenancy?
             ├─ YES → RoCEv2 on Arista
             └─ NO  → Evaluate per workload
```

---

## 3. InfiniBand Architecture

### Components

| Component | Role |
|---|---|
| **HCA (Host Channel Adapter)** | GPU server NIC — ConnectX-7 or newer |
| **HDR/NDR Switches** | Quantum-2 (400G NDR), Quantum (200G HDR) |
| **Subnet Manager** | UFM (NVIDIA) or OpenSM |
| **Adaptive Routing** | Dynamic load balancing per-packet |
| **GPUDirect RDMA** | Zero-copy GPU↔NIC transfers |

### Logical Data Flow

```
Step 1:  GPU triggers RDMA Write via CUDA/NCCL
Step 2:  HCA initiates IB transport (RC QP)
Step 3:  Subnet Manager programs LFTs (Linear Forwarding Tables)
Step 4:  Switch fabric routes via adaptive routing
Step 5:  Remote HCA completes DMA write to GPU memory
Step 6:  Completion event delivered to GPU kernel
```

### Subnet Manager (UFM / OpenSM) Config

```bash
# OpenSM basic startup
opensm --sm_priority 1 \
       --routing_engine ftree \   # Fat-tree routing
       --lmc 2 \                  # LID Mask Control (multipath)
       --log_file /var/log/opensm.log

# UFM — Recommended for production
# Provides GUI, telemetry, and adaptive routing policies
ufm start
ufm configure --routing adaptive --ar-mode ENABLED
```

### Adaptive Routing Policy

```ini
# /etc/opensm/opensm.conf
routing_engine       = ftree
lmc                  = 2
sm_priority          = 1
ar_mode              = ENABLED          # Adaptive routing on
ar_linear_forwarding = TRUE
ar_group_cap         = 512
```

### IB Fabric Verification

```bash
# Show fabric topology
ibnetdiscover

# Check link state and speed
ibstatus

# Verify port counters (errors)
perfquery -x 0                  # Extended counters
ibcheckerrors -c 10             # Check all ports

# Run bandwidth test
ib_write_bw -d mlx5_0 -i 1 --report_gbits

# Run latency test
ib_write_lat -d mlx5_0 -i 1
```

---

## 4. RoCEv2 Architecture (Arista)

### Components

| Component | Role |
|---|---|
| **Arista 7800R3** | Spine/Super-Spine — 400G |
| **Arista 7060X6** | Leaf/ToR — 64×400G |
| **Mellanox ConnectX-7** | Server NIC (RoCEv2) |
| **EOS (Extensible OS)** | Arista switch OS |
| **ECN + DCQCN** | Congestion control |
| **PFC (Priority Flow Control)** | Lossless fabric for RDMA |

### Logical Data Flow

```
Step 1:  GPU triggers RDMA via NCCL
Step 2:  ConnectX-7 encapsulates in UDP/IP (RoCEv2)
Step 3:  PFC pauses applied if buffer threshold hit
Step 4:  ECN marks packets at congestion point
Step 5:  DCQCN reduces sender rate (via CNP)
Step 6:  Arista routes via ECMP across spine
Step 7:  Remote NIC decapsulates, delivers to GPU
```

### Arista EOS — PFC Configuration

```eos
! Enable lossless queuing for RDMA traffic (DSCP 26 = CS3)
traffic-policy RDMA_LOSSLESS
   match RDMA dscp 26
      actions
         set traffic-class 3
   !
!

interface Ethernet1/1
   traffic-policy input RDMA_LOSSLESS
   priority-flow-control mode on
   priority-flow-control priority 3 no-drop
!
```

### Arista EOS — ECN Configuration

```eos
! Configure ECN thresholds on egress queues
qos random-detect ecn
   traffic-class 3
      minimum-threshold 150 kbytes
      maximum-threshold 1500 kbytes
      drop-probability 100
!
```

### Arista EOS — ECMP & RDMA-Aware Hashing

```eos
! Enable ECMP with fine-grained hashing for RDMA flows
ip load-sharing trident fields ip
ip load-sharing source-ip destination-ip ip-protocol
ip load-sharing udp source-port destination-port

! Verify ECMP
show ip route summary
show hardware capacity utilization
```

---

## 5. Network Configuration

### MTU Settings

```bash
# Set jumbo frames on all GPU server interfaces (required for RoCEv2)
ip link set eth0 mtu 9000
ip link set eth1 mtu 9000

# Persistent — add to /etc/network/interfaces or nmcli
nmcli connection modify ens3f0 802-3-ethernet.mtu 9000
```

### DSCP / QoS Marking (Server-Side)

```bash
# Mark RDMA traffic with DSCP 26 using tc
tc qdisc add dev eth0 root handle 1: prio
tc filter add dev eth0 parent 1: protocol ip u32 \
   match ip dport 4791 0xffff \    # RoCEv2 UDP port
   action skbedit priority 3

# Or via mlnx_qos tool (Mellanox NICs)
mlnx_qos -i eth0 --pfc 0,0,0,1,0,0,0,0   # Enable PFC on TC3
mlnx_qos -i eth0 --trust dscp
```

### NCCL Environment Variables

```bash
# Tune NCCL for RoCEv2 / IB
export NCCL_DEBUG=INFO
export NCCL_IB_GID_INDEX=3              # Use RoCEv2 GID
export NCCL_IB_TC=26                    # DSCP marking
export NCCL_IB_SL=3                     # Service Level
export NCCL_IB_TIMEOUT=22
export NCCL_IB_RETRY_CNT=7
export NCCL_SOCKET_IFNAME=eth0          # Specify interface
export NCCL_NET_GDR_LEVEL=2            # GPUDirect RDMA
export NCCL_TOPO_FILE=/etc/nccl/topo.xml
```

### GPUDirect RDMA Verification

```bash
# Verify GPUDirect is active
nvidia-smi --query-gpu=gpu_name,pcie.link.gen.current --format=csv
ls /dev/nvidia-uvm*

# Check RDMA capable interfaces
rdma link show
ibv_devinfo | grep -E "port|state|active"
```

---

## 6. Congestion Control

### DCQCN Parameters (RoCEv2)

```bash
# Mellanox NIC DCQCN tuning via mlnx_qos / sysfs
# Target: keep queue depth low, react fast

# On the NIC
echo 1    > /sys/class/infiniband/mlx5_0/ports/1/congestion_control/enable
echo 5    > /sys/class/infiniband/mlx5_0/ports/1/congestion_control/Rp_min_dec_fac
echo 200  > /sys/class/infiniband/mlx5_0/ports/1/congestion_control/Rp_clamp_Tgt_Rate

# Key DCQCN knobs
# Kp (reaction point gain)      — higher = faster rate reduction
# Ki (injection point gain)     — controls ECN marking aggressiveness
# Alpha_resume_offset           — rate recovery speed after congestion
```

### InfiniBand Congestion Control

```bash
# IB CC via IBTA standard (switch-based)
# Configure via OpenSM or UFM

# opensm.conf
cc_enabled           = TRUE
cc_ca_type           = BOTH              # Apply to both CA and switches
cc_algo_type         = SWR              # Switch-to-receiver
cc_sw_pkey           = 0xFFFF
```

---

## 7. Monitoring & Observability

### Key Metrics to Track

| Metric | Threshold | Tool |
|---|---|---|
| IB Port Errors | 0 (any = alert) | `perfquery`, UFM |
| PFC Pause Frames | > 0 sustained = bad | `ethtool -S`, EOS |
| ECN Marked Packets | < 1% of traffic | Arista telemetry |
| NCCL AllReduce BW | > 90% line rate | NCCL bandwidth test |
| GPU-to-GPU Latency | < 5µs (IB), < 10µs (RoCE) | `ib_write_lat` |
| Switch Buffer Util | < 50% steady state | EOS `show queue` |

### Arista EOS Telemetry (Streaming)

```eos
! Enable gNMI streaming telemetry
management telemetry gnmi
   transport grpc default
      ssl profile TELEMETRY
   provider eos-native
!

! Subscribe to queue depths and PFC counters
! Use Prometheus + Grafana for dashboarding
```

### Prometheus / Node Exporter (Server-Side)

```bash
# Install RDMA exporter
pip install rdma_exporter

# Metrics exposed at :9099/metrics
# rdma_port_rcv_errors_total
# rdma_port_xmit_discards_total
# rdma_port_rcv_pkts_total
# rdma_port_xmit_data_bytes_total
```

---

## 8. Failure Modes & Remediation

### Common Issues

| Symptom | Root Cause | Fix |
|---|---|---|
| NCCL timeout / hang | PFC deadlock or link flap | Check PFC watchdog, verify DCQCN |
| Low AllReduce BW | ECMP imbalance or MTU mismatch | Verify hashing, check `mtu 9000` end-to-end |
| IB port errors | Bad cable or SFP | `perfquery`, replace transceiver |
| SM failover storm | Multiple OpenSM instances | Set priority, enable single master |
| GPU training stall | GPUDirect RDMA disabled | Reload `nvidia-peermem` module |

### PFC Deadlock Prevention

```bash
# Enable PFC Watchdog on Mellanox NICs
# Detects and breaks PFC deadlock automatically
echo 1 > /sys/kernel/debug/mlx5/0000:xx:00.0/pfcwd/enable
echo 200 > /sys/kernel/debug/mlx5/0000:xx:00.0/pfcwd/poll_time_ms
echo 600 > /sys/kernel/debug/mlx5/0000:xx:00.0/pfcwd/restore_time_ms
```

```eos
! Arista EOS — PFC Watchdog
pfc watchdog
   on
   polling-interval 200 milliseconds
   recovery-time 600 milliseconds
!
```

---

## 9. Performance Benchmarking

### NCCL AllReduce Benchmark

```bash
# Run across all 2000 GPUs (250 nodes × 8 GPU)
mpirun -np 250 \
  --hostfile /etc/nccl/hostfile \
  -x NCCL_IB_GID_INDEX=3 \
  -x NCCL_DEBUG=WARN \
  nccl-tests/build/all_reduce_perf \
    -b 8 -e 4G -f 2 -g 8

# Expected output (NDR IB, 2000 GPUs):
# Size: 4G  | BW: ~180 GB/s  | Time: ~22ms
```

### IB Bandwidth Test (Point-to-Point)

```bash
# Server A (receiver)
ib_write_bw -d mlx5_0 -i 1 --report_gbits -q 8

# Server B (sender)
ib_write_bw -d mlx5_0 -i 1 --report_gbits -q 8 <ServerA_IP>

# Expected: ~190 Gb/s for NDR 200G
```

### RoCEv2 Bandwidth Test

```bash
# Verify end-to-end RoCEv2 throughput
ib_write_bw --rdma_cm -d mlx5_0 --report_gbits \
            -x 3 \           # GID index for RoCEv2
            -q 8 \           # 8 QPs
            <remote_ip>
```

---

## 10. Deployment Checklist

### Pre-Deployment

- [ ] All switch ports set to `mtu 9216` (jumbo frames)
- [ ] PFC enabled on RDMA traffic class (TC3)
- [ ] ECN thresholds configured on all egress queues
- [ ] DSCP marking verified end-to-end (`tcpdump -i eth0 -v`)
- [ ] Subnet Manager running (IB) / BGP sessions up (RoCEv2)
- [ ] GPUDirect RDMA kernel module loaded (`nvidia-peermem`)
- [ ] NCCL environment variables set in job scheduler (Slurm/K8s)
- [ ] Topology file generated for NCCL (`nccl-topo-detect`)

### Post-Deployment Validation

- [ ] `ib_write_bw` achieves > 90% line rate
- [ ] NCCL `all_reduce_perf` passes across all nodes
- [ ] Zero IB port errors or PFC storm events after 1-hour soak
- [ ] Telemetry streaming to Grafana dashboard
- [ ] PFC Watchdog enabled on all NICs and switches
- [ ] Failover test: remove 1 spine switch, verify training continues

---

## Reference Links

- [NVIDIA UFM Documentation](https://docs.nvidia.com/networking/display/UFMSDNApplianceUserManual)
- [Arista EOS Configuration Guide](https://www.arista.com/en/support/toi)
- [NCCL GitHub & Tests](https://github.com/NVIDIA/nccl-tests)
- [RoCEv2 Deployment Best Practices — NVIDIA](https://docs.nvidia.com/networking/display/RDMAoRoCEv2)
- [IBTA InfiniBand Architecture Spec](https://www.infinibandta.org/ibta-specification/)

---

*Last Updated: April 2026 | Covers HDR/NDR InfiniBand and 400GbE RoCEv2*
