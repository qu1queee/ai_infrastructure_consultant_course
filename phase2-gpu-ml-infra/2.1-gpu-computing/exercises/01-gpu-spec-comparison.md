# Exercise 01 — GPU Spec Comparison

Match GPU hardware to customer workloads. For each scenario, choose the best GPU from the candidates, justify your choice with at least two specific spec numbers, and identify the primary constraint the workload is hitting.

---

## Scenario A — LLM Pre-training

A customer is pre-training a 70B parameter LLM from scratch using BF16 mixed precision. They want to minimize time-to-train on a fixed 8-GPU node budget. Model parallelism across all 8 GPUs is required.

**Candidates:** A100 SXM4 80GB · H100 SXM5 80GB · L40S PCIe 48GB

**Your answer:**

1. Best GPU: _______________
2. Spec numbers that drive the decision:
   - _______________
   - _______________
3. Primary constraint (circle one): **Compute** · **Memory capacity** · **Memory bandwidth** · **Interconnect BW**
4. Why the other candidates don't fit:
   - A100: _______________
   - L40S: _______________

---

## Scenario B — Real-time Inference (Single Model)

A customer is deploying a single Llama 3 8B model for real-time chat. They need ≤50ms time-to-first-token (TTFT) at p99, serving ~200 concurrent users. Budget is the primary concern — they want to minimize GPU cost per request.

**Candidates:** H100 SXM5 80GB · L40S PCIe 48GB · A10G PCIe 24GB

**Your answer:**

1. Best GPU: _______________
2. Spec numbers that drive the decision:
   - _______________
   - _______________
3. Primary constraint: **Compute** · **Memory capacity** · **Memory bandwidth** · **Cost**
4. Would MIG be useful here? Why or why not: _______________

---

## Scenario C — Batch Inference at Scale

A customer runs nightly batch jobs: embedding generation over a 500M-document corpus using a 335M-parameter embedding model (BERT-large class). Throughput matters, not latency. They want to process the full corpus in under 6 hours.

**Candidates:** H100 SXM5 80GB · L40S PCIe 48GB · A100 PCIe 80GB

**Your answer:**

1. Best GPU: _______________
2. Spec numbers that drive the decision:
   - _______________
   - _______________
3. Is this workload compute-bound or memory-bound? Explain briefly: _______________
4. Would INT8 quantization help here? Why or why not: _______________

---

## Scenario D — Multi-tenant Dev Environment

A platform team wants to give 7 ML engineers each their own isolated GPU environment for experimentation — primarily running 7B-class models interactively. They have one H100 SXM 80GB available and don't want to buy more hardware.

**Your answer:**

1. Feature to use: _______________
2. Which MIG profile would you assign to each engineer, and why: _______________
3. What's the tradeoff vs. giving each engineer a full A10G instead: _______________

---

## Success Criteria

- Scenario A: H100 SXM · NVLink bandwidth and BF16 TFLOPS are the cited reasons · A100 is slower compute + lower BW · L40S lacks NVLink and has no HBM
- Scenario B: L40S · cost per token and sufficient memory for 8B in BF16 (16 GB, well within 48 GB) · H100 is over-provisioned · A10G may hit memory limits under high concurrency with KV cache
- Scenario C: L40S or A100 · small model = compute-bound at batch size · INT8 would increase throughput further since accuracy matters less for embeddings
- Scenario D: MIG · 1g.10gb profile (7 instances) · tradeoff is fixed allocation vs. elastic single-GPU
