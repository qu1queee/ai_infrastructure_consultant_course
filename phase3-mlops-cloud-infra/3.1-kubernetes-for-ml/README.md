# 3.1 Kubernetes for ML

## Learning Objectives

*This module assumes production Kubernetes experience (operators, controllers, scheduling, multi-tenancy). Focus is on what changes when GPUs and ML workloads are added.*

- Understand what the GPU Operator automates and why manual GPU node setup doesn't scale
- Know the device plugin model for GPU resource exposure
- Configure MIG and time-slicing via Kubernetes
- Understand the Network Operator's role for RDMA workloads
- Design a GPU Operator upgrade strategy suitable for production clusters

---

## What Changes When You Add GPUs

You already know Kubernetes. Here is what is different in GPU clusters:

**Driver management is non-trivial.** Unlike CPU workloads where the kernel handles everything, GPU workloads require a specific NVIDIA driver version installed as a kernel module on every node. Driver version must match across nodes in a cluster and must be compatible with the CUDA version your containers expect. On a 100-node cluster, manual driver management is operationally untenable.

**GPU resources must be exposed to the scheduler.** Kubernetes doesn't know about GPUs natively. A device plugin must run on each node, discover GPUs, and advertise them as extended resources (`nvidia.com/gpu`). Without this, `resources.limits` can't reference GPUs.

**RDMA-capable networking requires additional kernel drivers and plugins.** For InfiniBand or RoCE-based multi-node training, pods need access to RDMA devices. Standard Kubernetes networking doesn't expose these — a separate operator manages Mellanox OFED drivers, SR-IOV device plugin, and RDMA plugin.

**GPU monitoring requires a dedicated exporter.** Standard node-exporter doesn't know about GPU utilization, memory, temperature, or NVLink bandwidth. DCGM Exporter adds these metrics to your existing Prometheus stack.

The GPU Operator and Network Operator automate all of the above.

---

## GPU Operator

The GPU Operator is a Kubernetes operator (built with Operator SDK) that manages the full NVIDIA software stack on GPU nodes as Kubernetes-native resources.

### What It Manages

| Component | What it does |
|---|---|
| NVIDIA Driver | Installs and manages the NVIDIA kernel driver as a DaemonSet |
| Container Toolkit | Installs nvidia-container-runtime so containers can access GPUs |
| Device Plugin | Exposes GPUs as `nvidia.com/gpu` extended resources |
| DCGM Exporter | Exports GPU metrics to Prometheus |
| MIG Manager | Configures and reconfigures MIG partitioning |
| Node Feature Discovery | Labels nodes with GPU hardware properties |
| GPU Feature Discovery | Labels nodes with detailed GPU attributes (memory, CUDA version, etc.) |

### Installation

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace
```

A single Helm chart deploys all components. Each component runs as a DaemonSet on GPU nodes.

### ClusterPolicy

The GPU Operator is configured via a `ClusterPolicy` custom resource. This is the single source of truth for the entire NVIDIA stack on the cluster:

```yaml
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: cluster-policy
spec:
  driver:
    enabled: true
    version: "550.54.15"
  mig:
    strategy: mixed          # or: single, none
  dcgmExporter:
    enabled: true
  devicePlugin:
    enabled: true
```

**SA implication:** the GPU Operator couples driver version to operator version. Before upgrading the GPU Operator, check the compatibility matrix. A driver upgrade on a running node terminates all GPU workloads on that node — this is the core operational risk to plan around.

---

## Device Plugin and GPU Scheduling

### Requesting GPUs in Pods

Once the device plugin is running, pods request GPUs as extended resources:

```yaml
resources:
  limits:
    nvidia.com/gpu: 2    # request 2 GPUs
```

GPUs are allocated exclusively per pod — no sharing by default. A pod requesting 1 GPU on an 8-GPU node gets exclusive access to 1 GPU; the other 7 are available for other pods.

### Node Labels from GPU Feature Discovery

GPU Feature Discovery labels nodes with detailed attributes:
```
nvidia.com/gpu.memory=81920
nvidia.com/gpu.product=NVIDIA-H100-SXM5-80GB
nvidia.com/cuda.driver.major=550
nvidia.com/mig.capable=true
```

Use these in `nodeSelector` or `nodeAffinity` to schedule workloads on specific GPU types — critical in heterogeneous clusters (H100 nodes for training, L40S nodes for inference).

```yaml
nodeSelector:
  nvidia.com/gpu.product: NVIDIA-H100-SXM5-80GB
```

---

## MIG in Kubernetes

MIG (Multi-Instance GPU) partitions a single GPU into isolated slices. The GPU Operator's MIG Manager handles partition configuration automatically based on node labels.

### MIG Strategies

Set via `ClusterPolicy.spec.mig.strategy`:

- **`single`:** all GPUs on a node use the same MIG profile (e.g., all `1g.10gb`)
- **`mixed`:** different GPUs on the same node can have different profiles
- **`none`:** MIG disabled

### Configuring MIG Profiles

Label a node to trigger MIG Manager reconfiguration:
```bash
kubectl label node gpu-node-01 nvidia.com/mig.config=all-1g.10gb
```

MIG Manager drains the node, reconfigures the GPU hardware, and relabels the node. This process requires brief workload disruption on that node.

### Requesting MIG Slices

After configuration, MIG slices appear as distinct resources:
```yaml
resources:
  limits:
    nvidia.com/mig-1g.10gb: 1    # request one 1g.10gb MIG slice
```

### Time-Slicing vs MIG

| | Time-Slicing | MIG |
|---|---|---|
| Isolation | None (shared GPU) | Hardware-level |
| Memory | Shared, no limits | Fixed HBM slice |
| Production safety | No (dev/test only) | Yes |
| Configuration | ConfigMap | Node label → MIG Manager |
| Use case | Dev/test multi-tenancy | Production multi-tenancy, inference serving |

Time-slicing (`devicePlugin.config.sharing.timeSlicing`) allows multiple pods to share a GPU. There is no memory isolation — one pod can OOM another. Do not use for production multi-tenant workloads.

---

## Network Operator

The Network Operator manages the host networking stack required for RDMA-based GPU communication. Without it, pods cannot perform InfiniBand or RoCE RDMA — multi-node training falls back to TCP, with severe performance degradation.

### What It Manages

| Component | What it does |
|---|---|
| MLNX OFED | Mellanox OpenFabrics Enterprise Distribution — kernel drivers for ConnectX NICs |
| SR-IOV Device Plugin | Exposes SR-IOV virtual functions as pod resources |
| RDMA Shared Device Plugin | Exposes RDMA devices to pods |
| MACVLAN / IPVLAN | Secondary network interfaces for RDMA traffic |

### NicClusterPolicy

Analogous to GPU Operator's `ClusterPolicy`:

```yaml
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata:
  name: nic-cluster-policy
spec:
  ofedDriver:
    image: mofed
    version: "24.01-0.3.3.1"
  rdmaSharedDevicePlugin:
    config: |
      {
        "resourceList": [{
          "resourceName": "rdma_shared_device_a",
          "rdmaHcaMax": 63,
          "selectors": {"vendors": ["15b3"]}
        }]
      }
```

### SA-Level Understanding

Most customers running multi-node training on InfiniBand or RoCE in Kubernetes need the Network Operator. If a customer reports slow multi-node training on K8s despite fast hardware, the first question is: is the Network Operator deployed and are pods actually using RDMA devices?

Check with:
```bash
kubectl get nnp            # NicNodePolicy status
kubectl describe pod <training-pod> | grep rdma   # is RDMA device allocated?
```

---

## Topology-Aware Scheduling

For multi-GPU jobs, GPU placement matters: two GPUs connected via NVSwitch (same node) are far more efficient than two GPUs connected via InfiniBand (different nodes).

### Topology Manager

Kubernetes Topology Manager aligns CPU, memory, and device allocation to the same NUMA node. For GPU workloads:
```
kubelet --topology-manager-policy=best-effort  # or: restricted, single-numa-node
```

`single-numa-node` enforces that all resources (GPU + CPU + memory) are on the same NUMA domain — critical for latency-sensitive inference.

### Gang Scheduling

Standard Kubernetes scheduler allocates pods one at a time. A multi-pod training job can partially schedule (some pods running, some pending) and deadlock. **Gang scheduling** ensures all pods in a group are allocated simultaneously or not at all.

**Volcano** is the de facto gang scheduler for ML workloads on Kubernetes. It adds `Job` and `Queue` CRDs and integrates with SLURM-style priority and fairshare concepts.

---

## DCGM Exporter and GPU Monitoring

DCGM Exporter (deployed by GPU Operator) adds NVIDIA GPU metrics to Prometheus. Your existing Prometheus + Grafana stack picks these up with no additional configuration beyond adding the scrape target.

### Key Metrics

| Metric | What it measures |
|---|---|
| `DCGM_FI_DEV_GPU_UTIL` | GPU compute utilization (0–100%) |
| `DCGM_FI_DEV_FB_USED` | GPU framebuffer (HBM) used, MB |
| `DCGM_FI_DEV_FB_FREE` | GPU framebuffer free, MB |
| `DCGM_FI_DEV_POWER_USAGE` | Power draw, Watts |
| `DCGM_FI_DEV_GPU_TEMP` | GPU temperature, Celsius |
| `DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL` | NVLink bandwidth, MB/s |
| `DCGM_FI_DEV_SM_ACTIVE` | Fraction of SMs active (more granular than GPU_UTIL) |

**GPU utilization below 70% on a training job** is a red flag. Common causes:
- Data loading bottleneck (CPU can't feed GPU fast enough)
- Small batch size (GPU not amortizing kernel launch overhead)
- Network-stalled all-reduce (RDMA not working)
- Memory-bound operation (inspect `SM_ACTIVE` vs `GPU_UTIL`)

NVIDIA provides pre-built Grafana dashboards for DCGM metrics — available in the NVIDIA/dcgm-exporter GitHub repo.

---

## GPU Operator Upgrade Strategy

This is the operational concern most SA conversations eventually reach. GPU Operator upgrades involve driver upgrades, which terminate running GPU workloads.

### Safe Upgrade Pattern

1. **Check compatibility matrix** — GPU Operator version → driver version → CUDA version → framework version. All must be compatible with customer's workloads.
2. **Stage in non-prod first** — validate driver upgrade doesn't break workloads.
3. **Drain one node** — cordon + drain to evict all GPU workloads.
4. **Upgrade GPU Operator** — `helm upgrade` updates the ClusterPolicy and triggers DaemonSet rollout.
5. **Observe DaemonSet rollout** — GPU Operator uses a rolling update, upgrading one node at a time by default.
6. **Validate on upgraded node** — run a test workload before allowing the rollout to continue.
7. **Monitor DCGM metrics** — watch for driver errors, memory errors, or utilization anomalies post-upgrade.

### Upgrade Risk

Driver upgrades on active GPU nodes kill all running pods on that node. For training jobs without checkpointing, this means job loss. SA recommendation: ensure workloads checkpoint regularly before scheduling a driver upgrade window.

---

## Key Takeaways

- The GPU Operator automates the entire NVIDIA software stack via a single `ClusterPolicy` — driver, device plugin, DCGM exporter, MIG manager. One Helm chart replaces manual node setup.
- GPUs are exclusive extended resources (`nvidia.com/gpu: N`). Sharing requires MIG (hardware isolation) or time-slicing (no isolation — dev/test only).
- The Network Operator is required for RDMA in Kubernetes — without it, multi-node training cannot use InfiniBand or RoCE.
- DCGM Exporter integrates with your existing Prometheus/Grafana stack. Low GPU utilization on a training job is always worth investigating.
- GPU Operator upgrades involve driver changes that evict running pods — plan upgrade windows and ensure workloads checkpoint.
