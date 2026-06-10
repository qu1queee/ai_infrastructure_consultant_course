# 1.5 GenAI Application Layer

## Learning Objectives

- Explain RAG architectures at the component level and identify where each component can fail at scale
- Make and defend a fine-tuning vs. prompting decision given a customer's accuracy, latency, and cost constraints
- Describe agent and MCP patterns and map them to real customer use cases
- Position NVIDIA NIM and NeMo correctly in a customer conversation (what they solve, when to recommend them)
- Explain Nebius AI Studio's capabilities relative to building a self-hosted inference stack

---

## Why This Module Exists

SA roles at AI clouds are no longer purely infrastructure conversations. Customers increasingly arrive with a workload description — "we want to build a RAG system over our internal docs" or "we want to fine-tune a model on our support tickets" — and expect the SA to engage with that workload, not just size the GPUs underneath it. This module builds the application-layer vocabulary needed for those conversations.

---

## RAG Architectures

**Retrieval-Augmented Generation (RAG)** extends an LLM by retrieving relevant context from an external knowledge store at inference time, rather than encoding all knowledge in model weights.

### Core Components

```
Query
  │
  ▼
[Embedding Model]  ──→  Query Vector
  │
  ▼
[Vector Database]  ──→  Top-K Retrieved Chunks
  │
  ▼
[Context Assembly] ──→  Prompt = System + Retrieved Context + User Query
  │
  ▼
[LLM]              ──→  Response
```

| Component | Common Choices | SA Considerations |
|---|---|---|
| Embedding model | `text-embedding-3-small`, `bge-large`, `e5-mistral` | Latency at scale; co-locate with inference to avoid network round-trips |
| Vector database | Qdrant, Weaviate, pgvector, Pinecone | Managed vs. self-hosted; ANN index type (HNSW vs. IVF) affects recall/latency tradeoff |
| Chunking strategy | Fixed-size, sentence-boundary, semantic | Chunk size affects recall; too small = fragmented context, too large = diluted relevance |
| Reranker | cross-encoder, Cohere Rerank | Adds latency, significantly improves precision for larger top-K retrieval |

### Where RAG Fails at Scale

- **Embedding model becomes a throughput bottleneck** — customers often overlook that every query needs an embedding. Size this independently from the LLM.
- **Vector index doesn't fit in RAM** — HNSW graphs are memory-hungry. 1M vectors at 1536 dimensions ≈ 6 GB. Plan for this.
- **Stale index** — retrieval quality degrades as the knowledge base is updated but the index isn't. Ask customers about their ingestion cadence.
- **Context window exhaustion** — large top-K with long chunks can exceed the model's context length. Know the limits.

---

## Fine-Tuning vs. Prompting

This is one of the most common questions SAs field. The answer is almost always: **start with prompting**.

| Dimension | Prompting / RAG | Fine-Tuning |
|---|---|---|
| Time to production | Hours | Days to weeks |
| Cost | Inference cost only | Training cost + inference cost |
| Knowledge update | Swap the retrieval index | Retrain or LoRA adapt |
| Style/format control | Moderate (via system prompt) | Strong |
| Latency | Higher (context window) | Lower (no retrieval step) |
| When to choose | Most cases; always try first | Custom style/domain, low-latency serving, privacy (no RAG store needed) |

**LoRA / QLoRA** are the dominant fine-tuning approaches in 2025 — they adapt a small fraction of model weights rather than the full parameter set. Know this when a customer mentions "fine-tuning" — they almost certainly mean LoRA.

### NVIDIA NeMo

NeMo is NVIDIA's framework for LLM training and fine-tuning. Relevant for:
- Customers who want to fine-tune on-premises with full data privacy
- Large-scale supervised fine-tuning and RLHF pipelines
- NVIDIA BCP (Base Command Platform) integration

---

## Agent and MCP Patterns

**Agents** are LLM-driven systems that use tools (function calls) to take actions — browsing the web, querying a database, running code — rather than just generating text.

### The Tool Use Loop

```
User Input
    │
    ▼
LLM decides: respond directly OR call a tool
    │
    ├── Tool call → execute tool → inject result → LLM continues
    │
    └── Final response
```

**MCP (Model Context Protocol)** is an open standard for connecting LLM applications to tools and data sources. Relevant for customers building agent systems on top of cloud-hosted models.

SA talking point: agent frameworks (LangGraph, LlamaIndex, CrewAI) are often overkill for simple tool use. The underlying mechanism is just structured prompting and function calling — customers benefit from understanding that before adopting a framework.

---

## NVIDIA NIM

**NIM (NVIDIA Inference Microservices)** packages optimized inference engines (TensorRT-LLM) with a standard OpenAI-compatible API into containers that deploy on Kubernetes.

| Property | Value |
|---|---|
| API compatibility | OpenAI `/v1/chat/completions` |
| Optimization | TensorRT-LLM under the hood; FP8/INT8 quantization pre-applied |
| Supported models | Llama 3, Mistral, Mixtral, Nemotron, and others |
| Deployment target | K8s + NVIDIA GPU Operator |

NIM is the right answer when a customer wants: fast time-to-production for inference, a standard API, and doesn't want to manage TRT-LLM optimization themselves.

---

## Nebius AI Studio

Nebius AI Studio provides managed inference endpoints for popular open models (Llama, Mistral, etc.) with an OpenAI-compatible API, removing the need to manage GPU instances, model loading, or scaling.

**SA positioning:** AI Studio is the fastest path from "we want to use Llama 3.1-70B" to production. It's the right starting point for customers who want to validate an application before committing to a self-hosted inference stack. If they need custom models, fine-tuned weights, or specific hardware control, they migrate to managed Kubernetes with NIM or a self-hosted vLLM stack.

---

## Key Takeaways

- RAG is the default pattern for grounding LLMs in proprietary knowledge. Know the component failure modes before any customer conversation.
- Fine-tuning is almost never the right first step. Prompting + RAG first, fine-tune when you have a clear gap that prompting can't close.
- Agent/MCP patterns are increasingly expected in SA conversations — know the mental model even if you're not building them.
- NIM bundles TRT-LLM optimization into a deployable container. It's the fastest path to optimized inference on K8s.
- Nebius AI Studio and NIM occupy adjacent spaces: Studio for managed endpoints, NIM for self-hosted K8s deployments.
