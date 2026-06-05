# Exercise 02 — Compute-Bound vs Memory-Bound Diagnosis

Given GPU telemetry and workload descriptions, determine whether the bottleneck is **compute** (TFLOPS saturated) or **memory bandwidth** (HBM bandwidth saturated), explain why, and recommend the right lever to pull.

---

## Background

Every GPU operation has an **arithmetic intensity**: the ratio of floating-point operations to bytes of memory traffic.

```
Arithmetic Intensity = FLOPs / Bytes read+written
```

A GPU has a peak compute rate (TFLOPS) and a peak memory bandwidth (TB/s). The **roofline model** says:

- If intensity > (Peak TFLOPS / Peak BW): you are **compute-bound** — add more compute or reduce ops
- If intensity < (Peak TFLOPS / Peak BW): you are **memory-bound** — reduce memory traffic or increase reuse

For an H100 SXM: `1,979 TFLOPS BF16 / 3.35 TB/s = ~591 FLOP/byte`. Any operation with arithmetic intensity below 591 is memory-bound on H100 in BF16.

---

## Scenario 1

**Workload:** LLM inference, Llama 2 70B in BF16, batch size 1, single H100 SXM 80GB  
**Observed metrics:**
- GPU utilization (SM active %): 15%
- Memory bandwidth utilization: 92%
- Tokens/sec: 18

**Questions:**

1. Is this workload compute-bound or memory-bound? How do you know from the metrics? _______________
2. What operation dominates the runtime and why does it have low arithmetic intensity? _______________
3. The customer wants to double tokens/sec. Which of these would actually help?
   - [ ] Upgrading to a faster GPU with 2x TFLOPS
   - [ ] Upgrading to a GPU with 2x memory bandwidth
   - [ ] Increasing batch size from 1 to 8
   - [ ] Applying INT8 weight quantization
4. Explain why batch size increase helps: _______________

---

## Scenario 2

**Workload:** Training a 7B LLM, BF16, batch size 32, single H100 SXM 80GB  
**Observed metrics:**
- GPU utilization (SM active %): 94%
- Memory bandwidth utilization: 40%
- MFU (Model FLOP Utilization): 48%

**Questions:**

1. Is this workload compute-bound or memory-bound? _______________
2. MFU is 48% — only half the theoretical peak. Before assuming the hardware is to blame, name two software-side causes: _______________
3. The customer wants to improve MFU. Which of these would help?
   - [ ] Switching from FP32 to BF16 (they haven't done this yet)
   - [ ] Enabling `torch.compile()`
   - [ ] Using FlashAttention instead of standard attention
   - [ ] Moving to a GPU with more memory capacity
4. Why does FlashAttention improve MFU even though the workload is already compute-bound? _______________

---

## Scenario 3

**Workload:** Fine-tuning a 13B model with LoRA, BF16, batch size 4, single A100 80GB  
**Observed metrics:**
- GPU utilization: 60%
- Memory bandwidth utilization: 55%
- Training loss is converging normally
- GPU memory: 71 GB / 80 GB used

**Questions:**

1. Neither compute nor memory is saturated — what does this suggest about the bottleneck? _______________
2. The team uses a custom DataLoader that preprocesses images on CPU. `nvidia-smi` shows GPU utilization dropping to near 0% every ~0.8 seconds. What's happening? _______________
3. What's the right fix? _______________
4. Memory is at 89% — they want to increase batch size but OOM. Name two techniques to reduce memory pressure without changing hardware: _______________

---

## Scenario 4 — Blank Slate

A customer shares this profiler output from `torch.profiler`:

```
----------- ------------ ------------ ------------
Name             CPU Time     CUDA Time    # Calls
----------- ------------ ------------ ------------
aten::linear     0.8ms        42.1ms       1,200
aten::softmax    0.2ms        18.7ms       600
aten::copy_      12.4ms       0.1ms        800
DataLoader       310ms        0ms          150
----------- ------------ ------------ ------------
```

1. Where is the bottleneck? _______________
2. Why is `aten::copy_` taking 12.4ms of CPU time but only 0.1ms of CUDA time? _______________
3. What does 310ms on DataLoader with 0ms CUDA time tell you? _______________
4. Prioritize the two fixes you'd recommend first: _______________

---

## Success Criteria

**Scenario 1:** Memory-bound (92% BW, 15% SM). Decoding loads full weight matrices for each token — near-zero arithmetic reuse. Useful levers: 2x BW upgrade, batch size increase (amortizes weight loads over more tokens), INT8 (halves bytes moved). More TFLOPS doesn't help.

**Scenario 2:** Compute-bound (94% SM). Low MFU causes: kernel launch overhead between small ops, unoptimized attention (quadratic memory writes). `torch.compile` and FlashAttention help MFU by fusing kernels and reducing memory round-trips in the attention block.

**Scenario 3:** CPU/IO bottleneck — GPU is starved. DataLoader is doing heavy CPU preprocessing and blocking the GPU. Fix: `num_workers`, `pin_memory=True`, `prefetch_factor`. Memory reduction techniques: gradient checkpointing, optimizer state offloading (ZeRO stage 2).

**Scenario 4:** DataLoader (310ms) is the bottleneck — GPU is idle during data loading. `aten::copy_` is a CPU-side host-to-device transfer happening synchronously. Fix: async data loading, pin_memory, prefetching.
