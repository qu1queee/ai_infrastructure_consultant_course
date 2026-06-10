# Exercise 01 — Multi-node Network Diagnosis & Design

## Scenario A — Diagnose the Slow Cluster

A customer has a 16-node cluster, each node with 8 × H100 SXM (128 GPUs total). They are training a 70B model with data parallelism across all 16 nodes. GPU utilization sits at 28% during training. The hardware is spec'd correctly and all GPUs show healthy in device checks.

**Your task:** work through the diagnostic tree and identify the most likely root causes. For each, state what you would check and what the fix would be.

---

### Diagnostic Step 1 — Is it the network?

The customer reports: `NCCL_DEBUG=INFO` logs show "Timeout on communicator." The all-reduce is taking ~18 seconds per step on a 70B model.

Calculate the expected all-reduce time:
- Model size: 70B params × 2 bytes (BF16) = _____ GB
- Theoretical transfer at 400 Gbps NDR InfiniBand: _____ seconds
- Actual observed: 18 seconds
- Conclusion: is this network-bound or something else? _______________

---

### Diagnostic Step 2 — What's the network config?

You find out:
- Nodes are connected via 25GbE Ethernet (standard switches, not RoCE-configured)
- There is no InfiniBand in this cluster
- `NCCL_SOCKET_IFNAME` is set to the 25GbE interface

Questions:
1. Is 25GbE Ethernet sufficient for 16-node H100 training? Justify with numbers.
2. What is the theoretical all-reduce ceiling at 25Gbps?
3. What is NCCL doing with standard Ethernet vs RDMA? What is the performance difference?

---

### Diagnostic Step 3 — Recommend the fix

The customer has budget to improve networking. They ask: "Should we add InfiniBand or RoCE?"

Answer:
1. What questions do you ask before recommending one over the other?
2. Your recommendation if:
   - Customer has Ethernet-skilled ops team, cost is a constraint: _______________
   - Customer is a research institution, performance is paramount: _______________
3. What PFC/ECN configuration is required for RoCE? Why does standard Ethernet not work?
4. Sketch the architecture of a rail-optimized topology for their 16-node cluster. How many leaf switches? How many HCAs per node?

---

## Scenario B — Greenfield Architecture

A new customer is building a 32-node DGX H100 cluster for large-scale LLM pre-training. They have no existing HPC infrastructure. Budget is $40M for the full cluster including networking.

**Design the network architecture:**

1. Intra-node interconnect:
   - What technology handles GPU-to-GPU communication within a node?
   - What is the bandwidth?
   - Who configures this — the customer, or does it come with the hardware?

2. Inter-node interconnect:
   - Recommend: InfiniBand or RoCE? Justify.
   - What InfiniBand generation would you specify (HDR/NDR/XDR)?
   - How many HCAs per DGX H100 node?
   - What network topology would you recommend for 32 nodes?

3. For the recommended topology:
   - How many leaf switches are required?
   - Is the fabric blocking or non-blocking?
   - What is the per-GPU network bandwidth?

4. NCCL configuration:
   - What environment variable tells NCCL to use InfiniBand over the Ethernet interface?
   - What is the significance of `NCCL_IB_DISABLE=0`?

---

## Success Criteria

**Scenario A:**
- Step 1: 140 GB; at 400 Gbps ~2.8s; 18s >> 2.8s, clearly network-bound
- Step 2: 25GbE is unusable for this scale; ~44s theoretical; NCCL falls back to TCP, 10–50x worse latency than RDMA
- Step 3: ask about ops team skills and budget; RoCE for Ethernet-first; IB for research/performance; PFC pauses frames on RDMA traffic class, ECN signals congestion; 4 HCAs per node (rail-optimized), 4 leaf switches minimum

**Scenario B:**
- NVSwitch (900 GB/s, comes configured in DGX)
- NDR InfiniBand, greenfield research cluster; 8 × NDR HCAs per DGX H100; fat-tree
- 2 spine + N leaf switches for non-blocking fat-tree at 32 nodes
- `NCCL_IB_HCA`, `NCCL_IB_DISABLE=0` enables IB path
