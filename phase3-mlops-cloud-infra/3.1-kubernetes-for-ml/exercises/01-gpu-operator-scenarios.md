# Exercise 01 — GPU Operator & Kubernetes for ML Scenarios

*These exercises assume you know Kubernetes well. Focus on what's new: GPU Operator, MIG, Network Operator, DCGM.*

---

## Scenario A — GPU Operator Rollout

A customer has a 50-node Kubernetes cluster (30 GPU nodes with H100, 20 CPU nodes). They currently manage NVIDIA drivers manually — a team member SSH's into each node and runs `apt install nvidia-driver-550`. They have frequent issues:
- Driver version drift between nodes (some on 525, some on 550)
- Driver updates require 2-hour maintenance windows per node
- New nodes take 4 hours to onboard correctly

**Design a GPU Operator migration plan:**

1. What is the first thing you check before deploying the GPU Operator onto nodes that already have manually-installed drivers? What is the risk if you skip this?

2. Write the `ClusterPolicy` spec fragment that:
   - Pins the driver to version `550.54.15`
   - Enables DCGM Exporter
   - Sets MIG strategy to `mixed`

3. The GPU Operator does a rolling DaemonSet update. On a node currently running 3 training jobs, what happens when the driver DaemonSet pod is updated? What should you do before the rollout reaches that node?

4. After rollout, a researcher reports: "My pod is stuck in `Pending`. Events show `0/30 nodes are available: 30 Insufficient nvidia.com/gpu`." The GPU Operator pods all show `Running`. What are the 3 most likely causes?

5. Design a safe upgrade process from GPU Operator v23.9 to v24.3 for a production cluster running 24/7 training jobs. What is the minimum number of steps? What is the risk at each step?

---

## Scenario B — MIG Configuration

A customer is running inference for multiple teams. They have 4 × H100 SXM 80GB nodes. Requirements:
- Team A: needs 2 isolated GPU environments each with 40GB (for 30B model inference)
- Team B: needs 7 isolated GPU environments each with ~10GB (for 7B model inference, 7 engineers)
- Team C: needs full H100 for a 70B model

**Answer:**

1. Which MIG profiles satisfy Team A's requirement? Write the kubectl label command to apply it to `node-01`.

2. Which MIG profile satisfies Team B's requirement? What is the maximum number of instances on one H100?

3. Team C needs a full GPU. What MIG profile (or lack thereof) do you assign?

4. Can you mix Team A, B, and C requirements on the same node? What `mig.strategy` would you use?

5. A Team B engineer reports: "My pod requesting `nvidia.com/mig-1g.10gb: 1` is Pending despite the node having available MIG instances." Name 2 things to check.

6. What is the operational impact of changing a node's MIG profile on running pods on that node?

---

## Scenario C — Multi-node Training Broken

A customer is running a 4-node PyTorch DDP training job on Kubernetes. Each node has 8 × H100 SXM connected via NDR InfiniBand. The job runs but throughput is 15% of expected. GPU utilization is 22%.

The pods are scheduled on all 4 nodes. `kubectl describe pod` shows the pod has `nvidia.com/gpu: 8` allocated. DCGM shows GPUs are not faulted.

**Diagnose:**

1. You check NCCL logs and see: "Using socket transport, not IB transport." What does this mean, and why is it happening?

2. What component is missing from this cluster that would enable RDMA for pods? Name the operator and the specific sub-component responsible for exposing RDMA devices.

3. Write the pod spec fragment showing how a pod requests an RDMA device once the operator is deployed.

4. The Network Operator is now deployed. NCCL logs still show socket transport. What are the next 2 things to check?

5. After fixing RDMA, GPU utilization rises to 82%. The remaining 18% is investigated — DCGM shows `DCGM_FI_DEV_SM_ACTIVE` at 60% while `DCGM_FI_DEV_GPU_UTIL` is 82%. What does this discrepancy suggest?

---

## Scenario D — DCGM Monitoring Setup

A customer has GPU Operator deployed with DCGM Exporter. They want to be alerted when:
- Any GPU exceeds 85°C
- GPU memory is more than 90% used for more than 5 minutes
- GPU utilization drops below 40% during a scheduled training window

**Answer:**

1. What Prometheus metric name maps to GPU temperature?

2. Write the PromQL alert expression for the temperature condition.

3. Write the PromQL alert expression for the memory utilization condition (hint: use `DCGM_FI_DEV_FB_USED` and `DCGM_FI_DEV_FB_FREE`).

4. The customer wants a Grafana dashboard that shows per-GPU utilization, memory usage, and NVLink bandwidth for all nodes. Where can they get a pre-built DCGM dashboard rather than building from scratch?

---

## Success Criteria

**Scenario A:**
1. Check if driver was installed via package manager or runfile; if GPU Operator tries to install its own driver while one exists, it may conflict or fail
2. `driver: {enabled: true, version: "550.54.15"}`, `dcgmExporter: {enabled: true}`, `mig: {strategy: mixed}`
3. Running GPU pods are terminated; drain the node first
4. GPU Operator driver pod not Ready; device plugin not registered; node has `NoSchedule` taint from GPU Operator during driver install
5. Drain node → helm upgrade → watch DaemonSet rollout → validate one node → unpause rollout

**Scenario B:**
1. `3g.40gb` (3/7 SMs, 40GB); `kubectl label node node-01 nvidia.com/mig.config=all-3g.40gb`
2. `1g.10gb`; up to 7 per H100
3. No MIG (`7g.80gb` or MIG disabled)
4. Yes with `strategy: mixed`; label each GPU differently
5. MIG Manager may not have finished reconfiguring; device plugin cache not refreshed
6. Running pods on that node are evicted (MIG reconfig requires GPU reset)

**Scenario C:**
1. NCCL fell back to TCP because no RDMA device is available to pods
2. Network Operator; RDMA Shared Device Plugin (or SR-IOV device plugin)
3. `resources: limits: rdma/rdma_shared_device_a: 1`
4. Check pod has RDMA device allocated; check `NCCL_IB_HCA` env var points to correct HCA
5. Memory-bound operations — SMs are active but stalled waiting for HBM data, not compute-limited

**Scenario D:**
1. `DCGM_FI_DEV_GPU_TEMP`
2. `DCGM_FI_DEV_GPU_TEMP > 85`
3. `DCGM_FI_DEV_FB_USED / (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE) > 0.9` for `5m`
4. NVIDIA/dcgm-exporter GitHub repo → `grafana/` directory
