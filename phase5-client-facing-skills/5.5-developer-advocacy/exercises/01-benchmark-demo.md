# Exercise 01 — Build a Token Factory Benchmark Demo

## Objective

Produce a single runnable Python script that measures TTFT, tokens/second, and $/1M tokens across at least two models on Token Factory. The script should output a formatted table a developer can share.

## What you'll build

```
Model                    | TTFT (ms) | Tokens/sec | $/1M tokens | Notes
-------------------------|-----------|------------|-------------|------
Llama 3.1-8B (FP8)      |       142 |      1,840 |       $0.06 | latency mode
Llama 3.1-70B (FP8)     |       310 |        420 |       $0.35 | latency mode
Llama 3.1-8B (FP8)      |       890 |     12,000 |       $0.01 | throughput mode (batch=32)
```

## Requirements

- Must use the Token Factory OpenAI-compatible endpoint
- Must measure real latency (no pre-cached responses)
- Must run with a single API key env var: `NEBIUS_API_KEY`
- Must work as both a standalone script and a Colab notebook
- Must include methodology comments (what batch_size, what context length, how many samples averaged)

## Deliverable

A `benchmark.py` script in a public GitHub repo, plus a `README.md` that explains:
1. How to run it
2. What the numbers mean
3. What assumptions the benchmark makes

## Why this exercise matters

This script is the single most reusable asset you will produce as a DA. Every workshop, blog post, and conference talk can reference or extend it. Building it forces you to understand Token Factory at the level required to answer developer questions credibly.

## Stretch goal

Add a cost comparison: same benchmark against self-hosted vLLM on a Nebius GPU instance. Document the total cost including instance cost vs. Token Factory API cost for a workload of 10M tokens/day.
