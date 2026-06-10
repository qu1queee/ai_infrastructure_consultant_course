# 2.3 Multi-node / Multi-GPU & HPC Networking

## Learning Objectives

- Explain why network becomes the bottleneck in multi-node GPU training and quantify the impact
- Distinguish InfiniBand from RoCE and recommend the right one given a customer's constraints
- Read a cluster network topology diagram and identify bandwidth bottlenecks
- Explain RDMA to a non-specialist without hardware jargon
- Map NVLink/NVSwitch (intra-node) vs InfiniBand/RoCE (inter-node) to their correct scope

---

## Why Network Is the Bottleneck in Distributed Training

When training a large model across multiple nodes, GPUs must continuously synchronize gradients — every parameter update requires all nodes to agree on the same gradient values. The dominant pattern is **all-reduce**: each node contributes its local gradients, and the result (the average) is broadcast back to all nodes.

For 8 H100s training a 70B model in BF16:
- Model parameters: 70B × 2 bytes = **140 GB**
- Per-step all-reduce volume: ~140 GB (gradients, same size as model)
- At 400Gbps (NDR InfiniBand): ~2.8 seconds just for communication per step
- At 25Gbps (standard Ethernet): ~45 seconds per step

The network is not a detail — it determines whether distributed training is viable at all.

---

## RDMA — Why It Exists

Standard TCP/IP networking introduces two overheads that become unacceptable at HPC scale:

1. **CPU involvement:** every packet goes through the kernel, consuming CPU cycles and introducing scheduling latency
2. **Memory copies:** data is copied from application memory → kernel buffer → NIC, and back

**RDMA (Remote Direct Memory Access)** eliminates both. It lets one machine read from or write directly to another machine's memory, bypassing the CPU and OS kernel entirely. The NIC handles the transfer in hardware.

| Property | TCP/IP | RDMA |
|---|---|---|
| Latency | 50–100 μs | 1–3 μs |
| CPU utilization | High (data path) | Near zero |
| Memory copies | 2–4 per transfer | 0 (zero-copy) |
| Kernel involvement | Always | Bypassed |

At 1μs latency, a 128-node cluster can complete an all-reduce in milliseconds rather than seconds. This is the difference between GPU utilization of 30% (network-stalled) and 85%+ (compute-bound).

---

## InfiniBand

InfiniBand is a networking technology built specifically for RDMA. It is a separate fabric from Ethernet — its own cables, switches, and NICs (called HCAs — Host Channel Adapters).

### Speed Generations

| Generation | Bandwidth (per port) | Notes |
|---|---|---|
| HDR | 200 Gbps | Current standard in deployed clusters |
| NDR | 400 Gbps | Current new deployments (H100/H200 era) |
| XDR | 800 Gbps | Emerging, aligns with Blackwell |

Clusters use dual-port HCAs, so effective bandwidth is often 2× per host.

### Key Components

- **HCA (Host Channel Adapter):** the NIC equivalent. Plugs into PCIe slot. Mellanox/NVIDIA ConnectX series is the dominant vendor.
- **IB Switch:** purpose-built switch (Quantum series from NVIDIA). No Ethernet compatibility.
- **Subnet Manager:** software that manages the IB fabric, assigns addresses, configures routes. Usually runs on a dedicated management node or on the head node.

### SA-Level Knowledge

InfiniBand is the right answer when:
- Customer is building a large training cluster (16+ nodes)
- MPI-heavy workloads with tight latency requirements
- Performance is the overriding priority over cost

InfiniBand is the wrong answer when:
- Customer has existing Ethernet infrastructure and Ethernet-skilled ops team
- Budget is constrained
- Cluster is <8 nodes (network is rarely the bottleneck at this scale)

---

## RoCE — RDMA over Converged Ethernet

RoCE delivers RDMA semantics over standard Ethernet. The same ConnectX HCAs support both IB and RoCE — it is a configuration choice on the same hardware.

**RoCEv2** (the current version) encapsulates RDMA traffic in UDP/IP packets, making it routable across standard Ethernet switches.

### The Critical Requirement: Lossless Ethernet

Ethernet is lossy by design — packets are dropped and retransmitted. RDMA has no retransmit mechanism (that is the point — the CPU is bypassed). Dropped packets cause RDMA connections to hang.

RoCE requires **lossless Ethernet**, configured via:
- **PFC (Priority Flow Control):** pause frames that signal a switch port to stop sending for specific traffic classes. Effectively turns Ethernet into a lossless fabric for RDMA traffic.
- **ECN (Explicit Congestion Notification):** signals endpoints to slow down before buffers overflow, avoiding the need for PFC pause frames where possible.

Misconfigured PFC/ECN is the #1 cause of RoCE performance problems in production. This is the operational burden of choosing RoCE over IB.

### IB vs RoCE — The Decision Framework

| Factor | InfiniBand | RoCE |
|---|---|---|
| Latency | ~1 μs | ~2–3 μs |
| Bandwidth | Up to 800 Gbps (XDR) | Up to 400 Gbps (200GbE) |
| Cost | Higher (separate fabric) | Lower (leverage existing switches) |
| Ops complexity | IB-specific tooling | Familiar Ethernet + PFC tuning |
| Performance | Best possible | ~85–90% of IB |
| When to use | Max performance, greenfield, research/HPC | Cost-sensitive, Ethernet-first orgs, enterprise |

For most enterprise customers adopting AI infrastructure: **RoCE is the practical choice**. For hyperscalers and research institutions building DGX SuperPODs: **InfiniBand**.

---

## NVLink and NVSwitch — Intra-node Fabric

NVLink and NVSwitch are NVIDIA's proprietary GPU interconnect — they operate *within* a node, not between nodes. Do not confuse with InfiniBand.

| | NVLink (H100 SXM) | PCIe 5.0 |
|---|---|---|
| GPU-to-GPU BW | 900 GB/s (bidirectional) | ~128 GB/s |
| Scope | Within node (8 GPUs via NVSwitch) | Within node |
| Topology | All-to-all via NVSwitch | Tree via PCIe switch |

**NVSwitch** is the chip that connects all 8 GPUs in an SXM node (DGX H100) in a full all-to-all topology. Any GPU can communicate with any other GPU at full NVLink bandwidth simultaneously.

**The two-level hierarchy:**
```
Within node:  GPU ←→ NVSwitch ←→ GPU    (NVLink, 900 GB/s)
Between nodes: Node ←→ IB/RoCE Switch ←→ Node  (InfiniBand/RoCE, 400 Gbps)
```

This is why multi-node training at scale is NVLink + InfiniBand — each level handles its own scope.

---

## Network Topologies

### Fat-Tree

The standard topology for HPC clusters. Every switch tier has full bisection bandwidth — any node can communicate with any other node at line rate, without oversubscription.

```
         [Core Switches]
        /       |       \
  [Aggr]    [Aggr]    [Aggr]
  / | \     / | \     / | \
[L] [L] [L][L] [L] [L][L] [L] [L]   ← Leaf switches
 |   |   |  |   |   |  |   |   |
[N] [N] [N][N] [N] [N][N] [N] [N]   ← Nodes (each with HCA)
```

Fat-tree is expensive at large scale (many switches) but gives predictable, high performance.

### Rail-Optimized Topology

Common in GPU clusters. Each node has multiple HCAs (e.g., 8 × NDR for an 8-GPU node), and each HCA connects to a different leaf switch (a "rail"). This maximizes parallel all-reduce bandwidth across the cluster.

### Key SA Talking Point

When a customer says "our 16-node cluster has poor distributed training performance," the first things to check are:
1. Network topology — is there oversubscription at any level?
2. Bandwidth per GPU — are all 8 HCAs on a DGX node actually connected?
3. PFC/ECN config (if RoCE) — is the fabric actually lossless?
4. NCCL configuration — is NCCL seeing the right topology?

---

## NCCL — The Software Bridge

NCCL (NVIDIA Collective Communications Library) is the library that implements all-reduce, all-gather, and other collective operations. It auto-detects the network topology (NVLink, IB, RoCE) and chooses the optimal algorithm.

Key SA-level facts:
- NCCL uses InfiniBand via the `libibverbs` / `librdmacm` stack — same path as any RDMA application
- `NCCL_DEBUG=INFO` produces detailed topology detection logs — useful for diagnosing why NCCL isn't using the fast path
- NCCL performs worse on Ethernet than IB/RoCE even at the same bandwidth, because TCP adds latency that disrupts collective synchronization

---

## Key Takeaways

- Network is the bottleneck in multi-node training. Quantify it: a 25Gbps Ethernet cluster is effectively unusable for large distributed training.
- RDMA eliminates CPU overhead and memory copies — ~1μs latency vs ~50μs TCP.
- InfiniBand: dedicated fabric, best performance, higher cost and ops complexity. Choose for max-performance clusters.
- RoCE: RDMA over Ethernet, 85–90% of IB performance, requires lossless Ethernet (PFC + ECN). Choose for cost-sensitive or Ethernet-first customers.
- NVLink/NVSwitch is intra-node only. InfiniBand/RoCE is inter-node. They are complementary layers, not alternatives.
- Fat-tree gives non-blocking bisection bandwidth. Rail-optimized maximizes all-reduce parallelism per GPU.
