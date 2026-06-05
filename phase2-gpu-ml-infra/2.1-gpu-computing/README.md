# 2.1 GPU Computing

## Learning Objectives

- Explain why GPUs outperform CPUs for ML workloads and what architectural properties drive that
- Read a GPU spec sheet and extract the numbers that actually matter for a given workload
- Distinguish between NVIDIA GPU families and match each to training vs. inference use cases
- Understand numerical precision formats and their throughput/accuracy tradeoffs
- Describe the CUDA execution model at a level sufficient for customer architecture conversations

---

## CPU vs GPU Architecture

A CPU is optimized for **latency** — a small number of powerful cores, large caches, and branch prediction to execute a single thread as fast as possible. A GPU is optimized for **throughput** — thousands of smaller, simpler cores that execute many threads simultaneously.

| Property | CPU (e.g., AMD EPYC) | GPU (e.g., H100 SXM) |
|---|---|---|
| Cores | 64–128 | 16,896 CUDA cores |
| Clock speed | 3–4 GHz | ~1.8 GHz |
| L3 cache | 256–512 MB | — |
| Memory bandwidth | ~300 GB/s | 3.35 TB/s (HBM3) |
| Optimized for | Serial, low-latency | Parallel, high-throughput |

ML workloads — especially matrix multiplications in forward/backward passes — are embarrassingly parallel. Every element of an output matrix can be computed independently. A GPU maps thousands of those independent operations onto hardware threads simultaneously, which is why a single H100 can do **~2,000 TFLOPS** of BF16 matrix multiply while a top CPU manages ~10.

### Streaming Multiprocessors (SMs)

The GPU's fundamental compute unit is the **Streaming Multiprocessor (SM)**. Each SM contains:
- CUDA cores (FP32 arithmetic)
- Tensor Cores (matrix math accelerators — the ones that matter for ML)
- Shared memory and L1 cache
- A warp scheduler

**Warp:** the unit of execution inside an SM. A warp is 32 threads that execute the same instruction in lockstep (SIMD). If threads in a warp take different code paths (diverge), the warp serializes them — a key reason to write code that avoids branching on GPU.

The H100 SXM has **132 SMs**, each with 4 Tensor Core units. Keeping all SMs busy is what "GPU utilization" measures.

---

## NVIDIA GPU Families

NVIDIA sells into two main use cases: **training/large inference** (data center A/H/B series) and **inference/edge** (L and A10 series). Knowing the lineup is table stakes for any customer conversation.

### Data Center — Training and Large-Scale Inference

| GPU | Architecture | FP8 TFLOPS | BF16 TFLOPS | HBM Capacity | Memory BW | Form Factor |
|---|---|---|---|---|---|---|
| A100 | Ampere (2020) | — | 312 | 80 GB HBM2e | 2 TB/s | SXM4 / PCIe |
| H100 | Hopper (2022) | 3,958 | 1,979 | 80 GB HBM3 | 3.35 TB/s | SXM5 / PCIe |
| H200 | Hopper (2024) | 3,958 | 1,979 | 141 GB HBM3e | 4.8 TB/s | SXM5 / PCIe |
| B100 | Blackwell (2025) | 14,000 | 7,000 | 192 GB HBM3e | 8 TB/s | SXM6 |
| B200 | Blackwell (2025) | 18,000 | 9,000 | 192 GB HBM3e | 8 TB/s | SXM6 |

**Key observations:**
- H200 vs H100: same compute, 76% more memory and 43% more bandwidth. The right upgrade when your model fits compute but OOMs on memory.
- B100/B200 vs H100: ~4–9x FP8 throughput, 2.4x memory capacity. The jump that makes 405B+ models tractable on a single node.
- A100 is still widely deployed. Many customers haven't migrated yet — knowing its limits helps you make the upgrade case.

### Inference — Cost-Optimized

| GPU | Architecture | INT8 TOPS | Memory | Power | Notes |
|---|---|---|---|---|---|
| L40S | Ada Lovelace | 733 | 48 GB GDDR6 | 350W | Best price/perf for inference; no NVLink |
| A10G | Ampere | 250 | 24 GB GDDR6 | 150W | AWS standard (g5 instances) |
| A30 | Ampere | 330 | 24 GB HBM2 | 165W | Data center inference, supports MIG |

The L40S is increasingly the answer when a customer asks "what's the cheapest GPU that can serve a 7B model at production latency?" — it costs roughly half an H100 and has enough memory for most inference workloads.

### PCIe vs SXM Form Factor

| | PCIe | SXM |
|---|---|---|
| Cooling | Passive (server fan) | Direct liquid or active base-plate |
| NVLink | Optional (NVLink Bridge, 2 GPUs) | Full NVSwitch fabric (8 GPUs) |
| GPU-to-GPU BW | ~600 GB/s | 900 GB/s (H100) |
| Typical use | Cloud single-GPU instances | DGX systems, HGX nodes |

SXM is always the right answer for multi-GPU training. PCIe makes sense for inference nodes where GPUs don't need to talk to each other.

---

## Memory Hierarchy

Memory is almost always the bottleneck in LLM workloads. Understanding the hierarchy helps diagnose and design around it.

```
Registers  (~256 KB per SM)      ← fastest, per-thread
     │
Shared Memory / L1 (~228 KB/SM) ← fast, manually managed, within a thread block
     │
L2 Cache  (~50 MB on H100)      ← shared across all SMs
     │
HBM (High Bandwidth Memory)     ← main GPU memory, ~3.35 TB/s on H100
     │
PCIe / NVLink                   ← inter-GPU or CPU↔GPU transfer
     │
System DRAM / NVMe              ← storage tier, ~50 GB/s PCIe 5.0
```

**HBM vs GDDR6:**
- HBM stacks DRAM dies vertically on the same package as the GPU die (using an interposer). This gives very wide memory buses (5,120-bit on H100) and very high bandwidth.
- GDDR6 is the same memory used in consumer GPUs. Cheaper and higher capacity per dollar, but 3–5x lower bandwidth. Fine for inference (memory reads, not writes in tight loops), not for training.

**Why memory bandwidth matters more than FLOPS for LLMs:**
During inference auto-regressive decoding, the model generates one token at a time. Each step loads the entire set of model weights from HBM to compute a single output. A 70B model in BF16 is 140 GB — at H100 bandwidth that's ~42ms per token just in weight loading, giving a theoretical ceiling of ~24 tokens/sec per H100. Arithmetic intensity is low; the GPU is waiting on memory, not compute.

---

## Precision Formats

Tensor Cores support multiple numerical formats. Choosing the right one is a core tradeoff between throughput and model accuracy.

| Format | Bits | Exponent | Mantissa | H100 TFLOPS | Notes |
|---|---|---|---|---|---|
| FP32 | 32 | 8 | 23 | ~67 | Training baseline; never use for LLM inference |
| TF32 | 19 | 8 | 10 | ~989 | PyTorch default matmul since A100; same range as FP32, less precision |
| BF16 | 16 | 8 | 7 | 1,979 | Same range as FP32, less mantissa. Training standard. |
| FP16 | 16 | 5 | 10 | 1,979 | Less range than BF16 — gradient overflow without loss scaling |
| FP8 (E4M3) | 8 | 4 | 3 | 3,958 | Forward pass; high precision within narrow range |
| FP8 (E5M2) | 8 | 5 | 2 | 3,958 | Backward pass; more range for gradients |
| INT8 | 8 | — | — | 3,958 | Post-training quantization; slight accuracy hit |
| INT4 | 4 | — | — | ~7,900 | Aggressive quantization; noticeable degradation on small models |

**Rule of thumb:**
- Training: BF16 mixed precision (weights FP32, matmuls BF16). FP8 training available in Transformer Engine (H100+) with modest accuracy tradeoff.
- Inference: INT8 or FP8 for throughput-optimized serving; BF16 when accuracy matters most.
- Always check: does the serving framework (vLLM, TensorRT-LLM) support the quantization level you need?

---

## CUDA Execution Model (Conceptual)

You don't need to write CUDA to be effective in this role, but you need to understand the execution model to read profiler output and understand framework bottlenecks.

**The hierarchy:**

```
Grid
└── Blocks (assigned to SMs)
    └── Warps (32 threads, scheduled together)
        └── Threads (execute the same instruction)
```

A **kernel** is a function that runs on the GPU. When you call `torch.matmul()`, PyTorch dispatches one or more CUDA kernels. Each kernel launch defines a grid of thread blocks; the GPU scheduler assigns blocks to available SMs.

**Key concepts for customer conversations:**

- **Occupancy:** how many warps are active on an SM at once, relative to the maximum. Low occupancy = SMs underutilized = throughput left on the table. Common cause: too-large shared memory allocation per block.
- **Memory coalescing:** threads in a warp should access contiguous memory addresses so the load can be serviced in a single transaction. Scattered access pattern = many transactions = memory-bound.
- **Kernel launch overhead:** ~10–50μs per launch. For very small operations, the overhead dominates compute time. This is why fused kernels (e.g., FlashAttention) exist — they replace many small kernel launches with one large one.

When a customer says "GPU utilization is 30%," the next question is: are SMs idle (compute-bound, not enough parallelism), or are they stalled waiting for memory (memory-bound, need more bandwidth or better access patterns)?

---

## MIG — Multi-Instance GPU

Available on A100 and H100. MIG partitions a single GPU into up to **7 isolated instances**, each with its own:
- Compute (fraction of SMs)
- Memory (HBM slice)
- Memory bandwidth (fraction of HBM BW)

**When to recommend MIG:**
- Inference serving with small models (7B and under) where a full GPU is wasted
- Multi-tenant environments where isolation is required
- Dev/test workloads that don't need a full GPU

**MIG profiles on H100 80GB SXM:**

| Profile | SMs | Memory | Instances |
|---|---|---|---|
| 1g.10gb | 1/7 | 10 GB | up to 7 |
| 2g.20gb | 2/7 | 20 GB | up to 3 |
| 3g.40gb | 3/7 | 40 GB | up to 2 |
| 7g.80gb | 7/7 | 80 GB | 1 (no MIG) |

MIG instances are hard partitions — they don't share compute dynamically. If one instance is idle, another can't use its SMs. For variable workloads, consider MPS (Multi-Process Service) instead — it time-shares the GPU without isolation guarantees.

---

## Key Takeaways

- GPUs win on ML because matrix multiplications are embarrassingly parallel and GPU memory bandwidth is 10x CPU bandwidth.
- For LLM inference, **memory capacity and bandwidth** are the primary constraints; arithmetic throughput is rarely the limiter.
- For LLM training, **NVLink bandwidth and compute (TFLOPS at your target precision)** are what to optimize.
- Know the current NVIDIA lineup cold: H100, H200, B100/B200 for training; L40S for cost-optimized inference.
- BF16 for training, INT8/FP8 for inference — and always verify framework support before committing to a precision.
- MIG is the right answer when a customer wants to serve small models cheaply without buying more GPUs.
