# 2.1 GPU Computing — Resources

## Documentation

- [NVIDIA H100 Tensor Core GPU Architecture Whitepaper](https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper) — Primary reference for H100 specs, Transformer Engine, NVLink 4.0. Read sections 1–4 and the Tensor Core section.
- [NVIDIA Multi-Instance GPU User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/) — Official MIG setup and profile reference. Useful when a customer asks about partitioning.
- [NVIDIA GPU Memory Architecture](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#memory-hierarchy) — CUDA C Programming Guide, Memory Hierarchy section. Authoritative on shared memory, registers, and L2.

## Articles

- [Tim Dettmers — Which GPU for Deep Learning](https://timdettmers.com/2023/01/30/which-gpu-for-deep-learning/) — Practical guide to GPU selection for ML workloads. Goes deep on memory bandwidth vs. compute tradeoffs. Read before any customer GPU sizing conversation.
- [Horace He — Making Deep Learning Go Brrrr From First Principles](https://horace.io/brrr_intro.html) — Explains the roofline model, compute-bound vs memory-bound, and why operator fusion matters. Required reading for understanding profiler output.
- [Lilian Weng — Large Transformer Model Inference Optimization](https://lilianweng.github.io/posts/2023-01-10-inference-optimization/) — Covers quantization, KV cache, batching strategies for inference. Connects GPU hardware properties to inference system design.

## Videos

- [NVIDIA GTC 2022 — H100 Architecture Deep Dive](https://www.nvidia.com/en-us/on-demand/session/gtcspring22-s41489/) — 40-minute session from NVIDIA engineers on H100 internals. Worth watching for the Transformer Engine and NVLink sections.
