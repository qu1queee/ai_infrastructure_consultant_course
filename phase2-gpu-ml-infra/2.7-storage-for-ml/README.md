# 2.7 Storage for ML

## Learning Objectives

- Explain why storage is a first-class concern in GPU cluster design, not an afterthought
- Distinguish parallel filesystems from object storage and know when each is appropriate
- Compare Lustre, WEKA, and VAST at the level needed for a customer recommendation conversation
- Describe checkpointing strategies at scale and the failure modes of each
- Position Nebius managed storage tiers relative to self-managed alternatives

---

## Why Storage Matters for GPU Infrastructure

In a multi-node training job, GPUs are the most expensive resource. The job of everything else — compute, network, storage — is to keep them busy. Storage that can't deliver data fast enough stalls the training loop and wastes GPU time.

For a 64-node H100 cluster running a 70B model:
- **Data loading bottleneck:** if the dataset loader can't keep up with the training forward pass, GPUs wait. Each 1-second stall on 64 × 8 = 512 H100s at ~$3/hour each = $0.43 wasted per stall.
- **Checkpoint write bottleneck:** a 70B model in BF16 is ~140 GB. Writing a checkpoint to slow storage interrupts training for seconds to minutes.
- **Multiple jobs competing:** shared filesystems in a cluster serve dozens of jobs simultaneously. Bandwidth contention is common and hard to debug.

---

## Parallel Filesystems

Parallel filesystems stripe data across many storage nodes simultaneously, allowing all compute nodes to read/write at aggregate bandwidth. They are the standard for HPC training workloads.

### Lustre

The dominant open-source parallel filesystem. Originally from Sun/Oracle, now maintained by Whamcloud and used widely in supercomputers and large HPC clusters.

**Architecture:**

```
Clients (GPU nodes)
     │
     ▼
MGS  ─── MDS (Metadata Server) ─── MDT (Metadata Target)
              │
              ▼
         OSS (Object Storage Server) ─── OST (Object Storage Target) × N
```

- **MDS/MDT:** handles directory lookups, file creation, permissions — metadata-only operations
- **OSS/OST:** handles actual data I/O. Multiple OSSes stripe data across OSTs in parallel.
- **Striping:** files are split into chunks distributed across OSTs. A file striped across 8 OSTs can be read/written at 8× the speed of a single OST.

**Performance characteristics:**
- Aggregate throughput scales with OST count. Large deployments: hundreds of GB/s to TB/s
- Metadata performance can bottleneck small-file workloads (many files, many opens/closes)
- Latency is higher than local NVMe — typically 0.1–1ms for small IOs

**SA talking point:** Lustre is the right answer for large HPC clusters with Linux-native workloads. It requires significant operational expertise; managed options (IBM Spectrum Scale, Nebius Lustre service) reduce that burden.

### WEKA

Commercial parallel filesystem designed for cloud-native and hybrid deployments. Key differentiators:

- **Software-defined:** runs on standard NVMe servers or cloud instances, no proprietary hardware required
- **Flash-native:** optimized for NVMe; delivers low latency (~100μs) alongside high throughput
- **Kubernetes-native:** WEKA CSI driver integrates natively with K8s and GPU Operator workloads
- **Tiering:** automatically tiers warm data to object storage (S3-compatible) while keeping hot data on NVMe

WEKA is increasingly common in AI cloud environments because it deploys quickly and doesn't require HPC storage expertise to operate.

### VAST Data

VAST is a scale-out all-flash storage platform. Differentiators:

- **Universal storage:** single namespace across file, object, and database access patterns
- **QLC flash with DRAM caching:** achieves high capacity at lower cost than all-NVMe while maintaining performance
- **Global deduplication and compression:** reduces effective cost per TB significantly for training datasets with repeated samples or similar images
- **NFS and S3 simultaneously:** customers can access the same data via POSIX NFS (for training) and S3 (for data pipelines) without copying

VAST is the right conversation when a customer needs unified storage for both training and data lake workloads.

### Comparison

| | Lustre | WEKA | VAST |
|---|---|---|---|
| Latency | ~0.1–1ms | ~100μs | ~100–500μs |
| Throughput ceiling | TB/s (at scale) | Hundreds of GB/s | Hundreds of GB/s |
| POSIX compliance | Full | Full | Full |
| S3 compatibility | Via bridge | Yes | Yes |
| Kubernetes integration | Via CSI | Native CSI | Via CSI |
| Operational complexity | High | Medium | Low–Medium |
| Best fit | Large HPC, supercomputers | Cloud-native AI clusters | Unified file+object, data lakes |
| Nebius offering | Managed Lustre | — | — |

---

## Object Storage for ML

Object storage (S3-compatible: AWS S3, GCS, Nebius Object Storage) is not a parallel filesystem — it has no POSIX semantics, higher latency (~1–10ms per request), but effectively unlimited scale and low cost.

### Where Object Storage Fits in ML Pipelines

```
Raw data ingest ──→ Object Storage (S3/GCS)
                         │
                         ▼
               Dataset preprocessing
                         │
                         ▼
               Training dataset (shards) ──→ Object Storage OR Parallel FS
                         │                                   ↑
                         ▼                           (copy to parallel FS
               Training job                          if throughput limited)
                         │
                         ▼
               Checkpoints ──→ Object Storage (durable, cheap)
                         │
                         ▼
               Model artifacts ──→ Object Storage → Inference serving
```

**Rule of thumb:**
- Dataset storage at rest: object storage (cheap, durable, large)
- Active training data loading: parallel filesystem (throughput, latency)
- Checkpoints: object storage (durable, survives node failure, cheap)
- Model serving artifacts: object storage, served via NIM or vLLM model loader

### S3-Compatible Access Patterns

Training frameworks like PyTorch `torchdata` and `webdataset` can stream directly from S3/GCS using sharded dataset formats (WebDataset `.tar` shards). This eliminates the need to copy datasets to a parallel filesystem for many workloads.

When to still use a parallel filesystem for data loading:
- Random access patterns (e.g., shuffled access to a large binary dataset)
- Latency-sensitive pipelines where object storage round-trips add up
- Frameworks that require POSIX file semantics

---

## Checkpointing at Scale

Checkpointing saves model state so training can resume after a failure. In large clusters with thousands of GPUs, hardware failures are not rare events — they are expected.

### Checkpoint Frequency vs. Cost

| Checkpoint interval | Data at risk (70B model, 1s/step) | Checkpoint write time (140GB to Lustre @ 50GB/s) |
|---|---|---|
| Every 100 steps | ~100 seconds of training | ~2.8 seconds overhead |
| Every 1000 steps | ~1000 seconds (~17 min) | ~2.8 seconds overhead (amortized) |
| Every 10,000 steps | ~3 hours of training | ~2.8 seconds overhead (well amortized) |

The cost is dominated by the data-at-risk, not the checkpoint write time (assuming fast parallel storage).

### Async Checkpointing

Synchronous checkpoint: training pauses while state is written to disk. Expensive for large models on slow storage.

**Async checkpointing** copies model state to CPU RAM (fast) and writes to disk in a background thread, allowing training to continue immediately. PyTorch `torch.distributed.checkpoint` and NVIDIA's DCP (Distributed Checkpoint) support this pattern.

For very large models (hundreds of GB), even CPU copy takes time. **Distributed checkpointing** writes each rank's shard independently in parallel, reducing total checkpoint time proportionally to the number of ranks.

### Nebius Managed Storage

Nebius Cloud offers managed storage tiers relevant to ML workloads:
- **Object Storage:** S3-compatible, used for dataset storage and checkpoint archival
- **Managed Lustre (via soperator integration):** parallel filesystem for active training data access
- **Network Block Storage:** high-performance block volumes, suitable for database and small-cluster NVMe workloads

When sizing a Nebius engagement: assume datasets on Object Storage, active training data streamed via WebDataset or copied to local NVMe / managed Lustre, checkpoints written to Object Storage.

---

## Key Takeaways

- Storage is a GPU-utilization problem. Slow storage = expensive GPUs waiting.
- Parallel filesystems (Lustre, WEKA, VAST) provide POSIX semantics and high throughput for active training data. Each has a different ops/cost/performance profile.
- Object storage is the right tier for at-rest datasets, checkpoints, and model artifacts — not for random-access training I/O.
- Async distributed checkpointing is the production standard for large models; synchronous per-node checkpoints don't scale.
- Nebius sells managed Lustre and S3-compatible object storage — know both tiers before any Nebius customer conversation.
