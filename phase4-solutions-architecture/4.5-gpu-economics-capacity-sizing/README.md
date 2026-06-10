# 4.5 GPU Economics & Capacity Sizing

## Learning Objectives

- Build a GPU sizing estimate from a workload description in a customer conversation
- Model TCO for a GPU cluster including compute, networking, storage, and operational cost
- Explain utilization math and why raw GPU count understates actual capacity requirements
- Make a reserved vs. spot vs. on-demand recommendation given a customer's workload patterns
- Produce a sizing deliverable (a simple calculator or table) a customer can use themselves

---

## Why This Module Exists

GPU capacity sizing is the single most common SA pre-sales deliverable. A customer says "we want to fine-tune Llama 3.1-70B on 100GB of proprietary data" or "we need to serve this model at 500 requests per second with P99 latency under 200ms" — and they expect a credible answer on GPU count, cost, and timeline within the same conversation. This module builds the structured reasoning to do that reliably.

---

## GPU Sizing Fundamentals

Sizing a GPU workload requires knowing the primary constraint. LLM workloads have two distinct phases with different constraints:

| Phase | Primary constraint | Secondary |
|---|---|---|
| Training (forward + backward pass) | Compute (TFLOPS) | Memory (model + optimizer states) |
| Inference prefill (processing the prompt) | Compute (TFLOPS) | Memory bandwidth |
| Inference decoding (generating tokens) | Memory bandwidth | Memory capacity (KV cache) |

### Memory Requirements for Common Model Sizes

| Model size | BF16 weights | FP8 weights | Optimizer states (Adam, BF16) | Activations (batch=1) |
|---|---|---|---|---|
| 7B | 14 GB | 7 GB | +28 GB (AdamW) | ~1 GB |
| 13B | 26 GB | 13 GB | +52 GB | ~2 GB |
| 70B | 140 GB | 70 GB | +280 GB | ~10 GB |
| 405B | 810 GB | 405 GB | +1,620 GB | ~60 GB |

**Optimizer state note:** AdamW stores momentum and variance for every parameter, doubling or tripling GPU memory requirements during training vs. inference. This is why training a 70B model requires 140 GB (weights) + 280 GB (optimizer) = 420 GB minimum — that's 6 H100 80GB GPUs just for parameters, before activations.

---

## Inference Capacity Sizing

### Throughput vs. Latency

Inference has two operating regimes:

- **Latency-optimized:** minimize time-to-first-token (TTFT) and time-per-output-token (TPOT). Low batch sizes, more GPUs.
- **Throughput-optimized:** maximize tokens/second/dollar. Large batch sizes via continuous batching, fewer GPUs.

The tradeoff: batching increases throughput but increases latency for individual requests.

### KV Cache Sizing

During inference, the KV cache stores attention keys and values for each token in the context window. It scales with:
- Context length
- Number of attention layers and heads
- Batch size (concurrent requests)

For Llama 3.1-70B (GQA, 8 KV heads):
- KV cache per token per layer ≈ 2 × (key + value) × head_dim × num_kv_heads × bytes/param
- At FP16: ~0.5 MB per token for the full 80-layer model
- At 8K context length with batch_size=32: ~128 GB KV cache alone

This is why inference at large context lengths requires more GPU memory than the model weights alone — and why memory capacity (not FLOPS) is the governing constraint for throughput.

### Tokens/Second Estimates per GPU

These are rough estimates for single-GPU throughput at batch_size=1 (latency mode):

| Model | GPU | Format | Tokens/sec (decode) |
|---|---|---|---|
| Llama 3.1-8B | H100 80GB | BF16 | ~2,000 |
| Llama 3.1-70B | H100 80GB (tensor parallel × 2) | BF16 | ~250 |
| Llama 3.1-70B | H100 80GB | FP8 | ~400 (single GPU, fits with quantization) |
| Llama 3.1-405B | H100 80GB × 8 | FP8 | ~80 |

For throughput mode (continuous batching, batch_size=32+), multiply by 10–30× at the cost of proportionally higher latency.

### Sizing for a Target QPS

```
Required GPU count = (QPS × avg_output_tokens) / (tokens_per_second_per_gpu × target_utilization)
```

Example: 500 QPS, avg 200 output tokens, Llama 3.1-70B on H100 FP8, 70% target utilization
```
= (500 × 200) / (400 × 0.70)
= 100,000 / 280
≈ 358 H100s
```

This is why inference at scale is expensive. Always sanity-check with the customer — "500 QPS at this model size requires ~360 H100s; is that the right scope?"

---

## Training Capacity Sizing

### GPU-Hours Estimate

Training compute is measured in **GPU-hours** (or GPU-days for large runs).

Chinchilla scaling law gives a rough compute budget:
```
Optimal training FLOPs ≈ 6 × N × D
  N = model parameters
  D = training tokens
```

For Llama 3.1-70B (70B params, 15T tokens):
```
FLOPs ≈ 6 × 70 × 10⁹ × 15 × 10¹² = 6.3 × 10²⁴ FLOPs
```

H100 SXM at ~50% MFU (Model FLOP Utilization, typical for well-optimized distributed training):
```
Effective H100 throughput ≈ 1979 × 10¹² FLOPs/s × 0.50 = 989 × 10¹² FLOPs/s
GPU-seconds = 6.3 × 10²⁴ / (989 × 10¹²) ≈ 6.4 × 10⁹ seconds
```

On 1,024 H100s:
```
Training time ≈ 6.4 × 10⁹ / 1,024 ≈ 6.2 × 10⁶ seconds ≈ 72 days
```

This matches the reported ~90-day timeline for Llama 3 training — close enough for a customer conversation.

### MFU — Why It Matters

**Model FLOP Utilization (MFU)** measures what fraction of theoretical GPU FLOPS the training job actually uses. 50% is a good target; below 30% indicates a problem (network bottleneck, memory bottleneck, poor batch size).

Low MFU = you need proportionally more GPUs for the same job. Improving MFU from 30% to 50% has the same effect as adding 67% more GPUs.

---

## TCO Modeling

Total Cost of Ownership goes beyond GPU rental rates.

### Cloud GPU Cost Components

| Component | Typical share of total cost |
|---|---|
| GPU compute (on-demand) | 60–75% |
| High-speed networking (IB/RoCE) | 10–15% |
| Storage (parallel FS + object) | 5–10% |
| CPU instances (management, preprocessing) | 3–5% |
| Egress / data transfer | 2–5% |
| Engineering / operational overhead | 15–25% (hidden cost) |

### Nebius vs. Hyperscaler Pricing

Nebius Cloud positions on price-performance for GPU compute specifically. As of 2025:
- H100 SXM on-demand pricing is materially lower than AWS/Azure/GCP equivalents
- Spot pricing available for fault-tolerant training workloads (checkpointing required)
- No egress fees for data within the Nebius network — relevant for data pipeline costs

Always pull current pricing before any customer conversation — GPU spot markets move fast.

### Reserved vs. Spot vs. On-Demand

| Purchase type | Cost | Risk | Best for |
|---|---|---|---|
| Reserved (1-year) | ~40% discount vs on-demand | Committed spend | Stable inference serving, recurring training |
| Spot / preemptible | ~60–80% discount | Job interruption | Training with checkpointing, batch jobs |
| On-demand | List price | None | Variable inference load, PoCs |

For a customer running continuous inference serving: reserved makes sense after a stable load period. For batch fine-tuning jobs: spot with async checkpointing every 15–30 minutes is typically 3–5× cheaper than on-demand.

---

## Utilization Math

Raw GPU count is not the same as effective capacity. Clusters are never 100% utilized.

Typical utilization by workload:
- Research/dev (many users, interactive): 20–40%
- Production training (scheduled jobs): 60–75%
- Production inference (auto-scaled serving): 50–70% average (peaks drive provisioning)

**Effective capacity formula:**
```
GPUs required = (Peak GPU demand) / (Target average utilization)
```

A 100-GPU peak demand at 70% target utilization → provision 143 GPUs.

Over-provisioning for reliability: add 10–20% headroom above the peak/utilization calculation for maintenance, failures, and burst.

---

## Sizing Deliverable Format

A standard SA sizing deliverable is a one-page table covering:

| Parameter | Value |
|---|---|
| Model | Llama 3.1-70B |
| Use case | Fine-tuning on 50GB proprietary dataset |
| Required GPU | H100 80GB SXM |
| GPU count (training) | 8 × H100 (1 node) |
| Training time estimate | 12–18 hours |
| Training cost (spot) | ~$150–250 |
| Required GPU (inference) | H100 80GB or L40S |
| GPU count (inference, 50 QPS) | 4 × H100 or 8 × L40S |
| Monthly inference cost (reserved) | ~$8,000–12,000 |
| Storage | 100 GB dataset + 200 GB checkpoints → Object Storage |
| Networking | Single-node (no IB required for 8-GPU fine-tuning) |

Always include assumptions. Customers remember the number; you need to be able to explain what drove it.

---

## Key Takeaways

- GPU sizing starts with identifying the constraint: compute (training, prefill) or memory bandwidth (decoding).
- KV cache is often larger than model weights at production batch sizes — size for it explicitly.
- MFU tells you how efficiently a training job uses its GPUs; improving MFU is cheaper than buying more GPUs.
- TCO is 30–40% more than compute costs alone — networking, storage, and ops add up.
- Spot + checkpointing is 3–5× cheaper than on-demand for training workloads. This is a concrete, repeatable recommendation.
- Always present sizing as a table with explicit assumptions. The customer will share it internally.
