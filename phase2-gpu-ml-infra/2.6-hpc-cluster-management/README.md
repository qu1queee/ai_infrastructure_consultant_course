# 2.6 HPC Cluster Management

## Learning Objectives

- Explain SLURM's architecture using Kubernetes as a mental model
- Trace a job from submission to completion and identify where things can fail
- Describe what enroot and Pyxis add to the container story in HPC environments
- Know what NVIDIA Base Command Manager (BCM) manages and where it stops
- Advise a customer on when to use SLURM vs Kubernetes vs both

---

## The Mental Model: SLURM Is the K8s of HPC

If you know Kubernetes, SLURM's architecture maps almost directly. The main conceptual shift: SLURM is designed for **batch jobs** (finite work, start-to-finish, strict resource accounting) rather than **long-running services** (always-on, health-checked, declarative state).

| Kubernetes | SLURM |
|---|---|
| kube-scheduler | slurmctld (controller daemon) |
| kubelet | slurmd (node daemon) |
| etcd | slurmdbd (accounting DB) |
| Namespace | Account / Partition |
| Pod | Job step (srun) |
| Job / CronJob | sbatch script / job array |
| Resource limits (CPU, mem) | --ntasks, --mem, --gres |
| Node selector / taints | Partition, constraints, GRES |
| PodSpec YAML | Batch script (#SBATCH directives) |
| kubectl get pods | squeue |
| kubectl describe pod | scontrol show job |

One key difference: SLURM has no concept of desired state reconciliation. If a node dies, the job fails — there is no automatic rescheduling to a new node. Fault tolerance is the application's responsibility (checkpointing).

---

## SLURM Architecture

### Core Daemons

**slurmctld** — the central controller. Runs on a dedicated head node (or HA pair). Manages:
- Job queue and scheduling decisions
- Node state tracking
- Partition and QOS configuration
- Communication with all slurmd daemons

**slurmd** — runs on every compute node. Manages:
- Local resource monitoring (CPUs, GPUs, memory)
- Job step execution
- Heartbeat to slurmctld

**slurmdbd** — the accounting daemon. Stores job history, user/account quotas, utilization data in a database (MariaDB/MySQL). Optional but standard in production.

**slurmrestd** — REST API frontend for programmatic job submission. Increasingly used for integration with ML platforms.

### Configuration

Everything is in `slurm.conf` — node definitions, partition configs, scheduler parameters, GPU (GRES) configurations. This is the equivalent of K8s admission controllers + scheduler configuration bundled into one file. Changes require a daemon restart or `scontrol reconfigure`.

---

## Job Lifecycle

```
User: sbatch train.sh
       ↓
slurmctld receives job → assigns JobID → state: PENDING
       ↓
Scheduler finds matching nodes (partition, GRES, priority)
       ↓
slurmctld allocates resources → state: RUNNING
       ↓
slurmd on each node executes job steps
       ↓
Job finishes / fails → state: COMPLETED / FAILED
       ↓
slurmdbd records accounting data
```

### Key Commands

```bash
sbatch train.sh              # submit a batch job
squeue                       # list all jobs in queue
squeue -u $USER              # your jobs only
scontrol show job <JobID>    # detailed job info
scancel <JobID>              # cancel a job
sacct -j <JobID>             # accounting/history after completion
sinfo                        # node and partition status
srun --pty bash              # interactive session on a compute node
```

### A Typical Batch Script

```bash
#!/bin/bash
#SBATCH --job-name=llm-train
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=8
#SBATCH --gres=gpu:8
#SBATCH --partition=gpu-high
#SBATCH --time=48:00:00
#SBATCH --account=ml-research

srun --container-image=nvcr.io/nvidia/pytorch:24.01-py3 \
     --container-mounts=/data:/data \
     python train.py --config config.yaml
```

The `--gres=gpu:8` requests 8 GPUs per node. SLURM binds them to the job exclusively — no other job shares those GPUs during this allocation.

---

## GPU Resource Management (GRES)

GRES (Generic Resource) is SLURM's extension point for any resource that isn't CPU or memory. GPUs are configured as GRES.

In `slurm.conf`:
```
GresTypes=gpu
NodeName=node[01-16] Gres=gpu:h100:8
```

In a job script:
```
#SBATCH --gres=gpu:h100:2     # request 2 H100s specifically
#SBATCH --gres=gpu:2          # any 2 GPUs
```

**GPU binding** (`--gpu-bind`) controls which GPUs map to which tasks. For multi-GPU training, `--gpu-bind=closest` ensures each MPI rank gets the GPU nearest to its CPU socket, minimizing PCIe contention.

### MIG with SLURM

SLURM supports MIG slices as named GRES types:
```
Gres=gpu:1g.10gb:7    # 7 MIG instances of 1g.10gb profile
```

Each MIG instance is an independently allocatable resource. A job can request `--gres=gpu:1g.10gb:2` to get 2 MIG slices.

---

## Partitions and QOS

**Partitions** are node pools with associated policies — the equivalent of K8s node pools with taints:
- Different hardware tiers (H100 vs A100 nodes)
- Different time limits (`MaxTime`)
- Different priority levels

**QOS (Quality of Service)** adds a second dimension of policy on top of partitions:
- Job priority multipliers
- Maximum running jobs per user
- GPU-hour limits per account

A well-configured production cluster has partitions for hardware types and QOS for organizational priorities (e.g., `urgent`, `normal`, `background`).

**Fairshare scheduling** is SLURM's mechanism for multi-tenant fairness — accounts that have used fewer resources recently get higher priority. The equivalent of Kubernetes resource quotas combined with a scheduler priority class, but tracked historically.

---

## Containers: enroot + Pyxis

Kubernetes has containerd/CRI. HPC clusters have **enroot** + **Pyxis**.

### enroot

A container runtime built by NVIDIA for HPC. Key design constraints that drove it:
- HPC environments are **unprivileged** — users cannot run Docker (requires root/daemon)
- Container images need to run as the user, not root
- No persistent daemon (Docker requires dockerd)

enroot converts Docker/OCI container images into unprivileged filesystem sandboxes:

```bash
enroot import docker://nvcr.io/nvidia/pytorch:24.01-py3
enroot create nvidia+pytorch+24.01-py3.sqsh
enroot start --mount /data:/data nvidia+pytorch+24.01-py3 python train.py
```

Images are stored as `.sqsh` (SquashFS) files — immutable, efficient, and mountable without root.

### Pyxis

Pyxis is a SPANK (SLURM Plug-in Architecture for Node and job Kontrol) plugin that integrates enroot into SLURM. It adds container flags directly to `sbatch`/`srun`:

```bash
srun --container-image=nvcr.io/nvidia/pytorch:24.01-py3 \
     --container-mounts=/data:/data,/scratch:/scratch \
     python train.py
```

Pyxis handles image import and container creation automatically per job step. From a user perspective, it is the HPC equivalent of `image:` in a pod spec.

**SA angle:** customers coming from Docker/K8s find Pyxis familiar. The main education needed is why Docker itself doesn't work in their cluster (no root, no daemon) and why enroot + Pyxis is the architectural equivalent.

---

## MPI — What You Need to Know

MPI (Message Passing Interface) is the communication standard that distributed HPC programs use to pass data between processes on different nodes. It is not a scheduler, not a container runtime — it is a library your training code links against.

Key facts for SA conversations:

- `mpirun` (or `mpiexec`) launches N processes, one per rank, potentially across multiple nodes
- With SLURM, `srun --mpi=pmix` replaces `mpirun` — SLURM manages process launch and rank assignment
- MPI uses RDMA (via InfiniBand or RoCE) for inter-node communication — this is the reason network latency matters so much for MPI-based training jobs
- PyTorch distributed training can use MPI as a backend, though NCCL is now more common for GPU workloads

When a customer says "our MPI job is slow," the first question is: is it network-bound (check InfiniBand/RoCE config) or compute-bound (check GPU utilization)?

---

## NVIDIA Base Command Manager (BCM)

BCM is a cluster lifecycle management platform that ships with NVIDIA DGX systems. It is not a job scheduler — it is the layer below SLURM and Kubernetes.

### What BCM Does

| Category | What it manages |
|---|---|
| Provisioning | Bare-metal OS imaging, PXE boot, node kickstart |
| Drivers | NVIDIA driver deployment and version management across all nodes |
| Firmware | GPU firmware, NIC firmware, BMC firmware updates |
| Health monitoring | GPU, CPU, memory, NIC, storage health; fault detection |
| Cluster inventory | Hardware inventory, topology discovery |
| Software stacks | Base OS, container runtime, MLNX OFED (InfiniBand drivers) |

### What BCM Does Not Do

BCM does not schedule jobs. It provides the base layer that SLURM or Kubernetes sits on top of:

```
BCM: provisions nodes, installs OS, drivers, firmware, monitors health
      ↓
SLURM / K8s: schedules workloads on those nodes
      ↓
enroot/Pyxis or container runtime: runs containerized jobs
```

### SA Framing

BCM is relevant in customer conversations when:
- Customer is buying DGX systems (BCM is included)
- Customer asks "how do we manage driver updates across 100 nodes?" — BCM answers this
- Customer has an existing IT provisioning tool (Ansible, Puppet) — BCM integrates or can be scoped out

BCM is not relevant for:
- Cloud GPU deployments (managed by cloud provider)
- Non-NVIDIA hardware
- Customers with strong existing bare-metal provisioning (Cobbler, Foreman, etc.) who want to use their own tooling

---

## SLURM vs Kubernetes — When to Recommend Which

This is the most common architecture question in AI infrastructure. The honest answer is: many large customers run both.

| | SLURM | Kubernetes |
|---|---|---|
| Best for | Batch training, HPC, MPI workloads | Serving, inference, mixed workloads |
| Scheduling model | Batch queue, fairshare | Declarative, continuous reconciliation |
| Fault tolerance | Application-level (checkpointing) | Platform-level (pod restart) |
| Container story | enroot + Pyxis | Standard OCI + containerd |
| GPU support | GRES, MIG | GPU Operator, device plugin |
| Multi-tenancy | Account/partition hierarchy | Namespace + RBAC |
| Networking | Deep MPI/RDMA integration | Via Network Operator |
| Typical users | Research institutions, HPC centers | Cloud-native orgs, ML platforms |

**The common pattern:** SLURM for training jobs (batch, MPI, HPC-native tooling), Kubernetes for model serving and ML platform tooling. A unified data plane (shared storage) connects them.

When a customer asks "should we use SLURM or K8s?" — push back and ask what their primary workload is, what their ops team knows, and whether they're doing research-style training or production serving. The answer is usually "both, with SLURM for training and K8s for serving."

---

## Key Takeaways

- SLURM is a batch job scheduler with direct K8s parallels: slurmctld ↔ scheduler, slurmd ↔ kubelet, partition ↔ node pool.
- Jobs are submitted via `sbatch` scripts with `#SBATCH` directives; GPUs are requested via `--gres=gpu:N`.
- enroot + Pyxis is the HPC container story — Docker doesn't work in unprivileged HPC environments.
- MPI is the communication library for distributed jobs; it relies on RDMA for performance.
- BCM manages the layer below SLURM/K8s: provisioning, drivers, firmware, health. It does not schedule jobs.
- Recommend both SLURM and Kubernetes for customers with mixed training and serving workloads.
