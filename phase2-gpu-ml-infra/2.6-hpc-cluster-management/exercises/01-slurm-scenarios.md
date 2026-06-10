# Exercise 01 — SLURM Operations & Architecture Scenarios

## Scenario A — Trace a Job

A user submits the following script on a 4-node SLURM cluster (each node has 8 × H100):

```bash
#!/bin/bash
#SBATCH --job-name=gpt-train
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=8
#SBATCH --gres=gpu:8
#SBATCH --partition=gpu-large
#SBATCH --time=72:00:00
#SBATCH --account=research-team-a

srun --container-image=nvcr.io/nvidia/pytorch:24.01-py3 \
     --container-mounts=/scratch/data:/data \
     torchrun --nproc_per_node=8 --nnodes=4 train_gpt.py
```

**Answer these questions:**

1. This job requests 4 nodes with 8 GPUs each. How many total GPU allocations is SLURM tracking?

2. The job enters state PENDING after submission. Name 3 possible reasons it might not start immediately.

3. Map each `#SBATCH` directive to its Kubernetes equivalent:
   - `--nodes=4` ↔ _______________
   - `--gres=gpu:8` ↔ _______________
   - `--partition=gpu-large` ↔ _______________
   - `--account=research-team-a` ↔ _______________
   - `--time=72:00:00` ↔ _______________

4. The `--container-image` flag is handled by _______________. What does this tool do that Docker cannot in this environment?

5. After 48 hours, the job dies because slurmctld restarts. What is the job state? Does SLURM automatically restart it? What should the user have done?

6. Write the `squeue` command to show only this user's running jobs, formatted with JobID, partition, name, state, and time used.

---

## Scenario B — Cluster Architecture Design

A university research center is building an AI cluster. They have:
- 20 DGX H100 nodes (160 GPUs total)
- 200 researchers across 5 departments
- Workloads: large LLM training (days-long jobs), short exploratory experiments, and model evaluation scripts
- Existing IT team knows Kubernetes but not SLURM
- Compliance requirement: department-level usage accounting and budget caps

**Design their cluster management architecture:**

1. SLURM or Kubernetes? Or both? Justify your recommendation for this specific customer.

2. If using SLURM, design the partition structure:
   - What partitions would you create?
   - What `MaxTime` would you set per partition?
   - What GRES would each node advertise?

3. Design the account/QOS hierarchy:
   - How would you represent 5 departments?
   - How do you enforce per-department GPU-hour budgets?
   - How would you implement priority so exploratory jobs don't starve long training runs?

4. Container strategy:
   - Why can't researchers just use Docker to run their training containers?
   - What is the correct container stack for this cluster?
   - How does a researcher use their existing Docker image in a job?

5. BCM relevance: the cluster comes with DGX systems. How does BCM fit into the architecture? What does it handle that SLURM does not?

---

## Scenario C — SLURM vs Kubernetes: The Customer Debate

A customer's ML platform team and HPC team are having an internal debate. Platform team wants Kubernetes for everything. HPC team wants SLURM for everything. You are the SA facilitating the architecture review.

**For each workload, recommend SLURM, Kubernetes, or both, and justify:**

| Workload | Recommendation | Justification |
|---|---|---|
| LLM pre-training, 128 GPUs, MPI-based, 2-week jobs | | |
| REST inference API, auto-scaled, 1–8 GPUs | | |
| Nightly batch embedding generation, 16 GPUs, 6 hours | | |
| Interactive Jupyter notebooks for ML engineers | | |
| Hyperparameter sweep, 1000 short jobs, 1 GPU each | | |
| Model fine-tuning, PyTorch DDP, 8 GPUs, 4 hours | | |

**What is the single architecture pattern you would recommend to bridge both teams?**

---

## Success Criteria

**Scenario A:**
1. 32 GPU allocations total
2. Not enough GPUs free in partition; other jobs ahead in queue with higher priority; time limit exceeds partition MaxTime
3. nodes=4 ↔ replicas; gres ↔ resources.limits; partition ↔ nodeSelector/taint; account ↔ namespace/quota; time ↔ activeDeadlineSeconds
4. Pyxis; enroot converts Docker images to unprivileged sandboxes (no root, no daemon required)
5. FAILED; no auto-restart; should checkpoint periodically and resubmit with `--open-mode=append`
6. `squeue -u $USER -t RUNNING -o "%.10i %.12P %.20j %.8T %.12M"`

**Scenario B:**
1. Both — SLURM for training jobs (MPI, long-running, HPC-native), K8s for serving/platform tooling
2. Partitions: `gpu-small` (≤2h, exploration), `gpu-medium` (≤24h), `gpu-large` (≤7d, training); all nodes advertise `gpu:h100:8`
3. One SLURM account per department; SACCTMgr GrpTRESMins for budget caps; QOS with priority tiers
4. No root/daemon in HPC → enroot + Pyxis; `--container-image=` in sbatch
5. BCM handles OS imaging, driver deployment, firmware — SLURM schedules workloads on nodes BCM provisions

**Scenario C:**
- LLM pre-training → SLURM (MPI, long jobs, RDMA)
- Inference API → K8s (serving, autoscaling)
- Batch embedding → SLURM or K8s (batch-friendly either way)
- Jupyter → K8s (interactive, service-oriented)
- HP sweep → SLURM (job arrays are native)
- Fine-tuning → either, slight K8s edge if cloud-native org
- Architecture pattern: shared storage (parallel filesystem), SLURM for training, K8s for serving
