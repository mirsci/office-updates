Validating design -> costing is a byproduct
Design to become more mature

joint dev env design -> focus on model inference with fractional GPUs for custom model weights -> costing model to token economics
Deep down into design to figure some key things 

Cost model emerges from design; cannot be defined separate or a-priori

### MIG vs Time-Slicing Trade-offs

RunAI supports two fractional GPU modes with different characteristics:

| Mode | Isolation | Latency Predictability | Memory Guarantee | H100 Support |
|------|-----------|------------------------|------------------|--------------|
| **MIG** (Multi-Instance GPU) | Hardware | ✅ High | ✅ Dedicated slice | ✅ Yes |
| **Time-slicing** | Software | ⚠️ Variable | ❌ Shared | ✅ Yes |

**Recommendation**: MIG preferred for latency-sensitive inference (1.5s SLA).

**H100 MIG partition sizes**:

| Profile | VRAM | Compute (SMs) | Notes |
|---------|------|---------------|-------|
| 1g.10gb | 10GB | 1/7 | Target for Gemma 4 E4B (~6-8GB) |
| 2g.20gb | 20GB | 2/7 | Fallback if 1g.10gb unavailable |
| 3g.40gb | 40GB | 3/7 | Overkill for current model |
| 7g.80gb | 80GB | Full | Dedicated allocation |

**RunAI scheduler architecture** (MI_machine):


### Inference Engine ↔ RunAI Relationship

**Key insight**: RunAI operates at the **infrastructure/scheduling layer**, while inference engines (LiteRT-LM, vLLM) operate at the **application layer**. They interact through standard NVIDIA GPU interfaces, not proprietary APIs.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Application Layer                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  inference_srv_py                                            │   │
│  │  ┌─────────────────┐  OR  ┌─────────────────┐               │   │
│  │  │  LiteRT-LM      │      │  vLLM           │               │   │
│  │  │  - Vulkan/GPU   │      │  - CUDA/GPU     │               │   │
│  │  │  - TFLite/GGUF  │      │  - HF/SafeTens. │               │   │
│  │  └────────┬────────┘      └────────┬────────┘               │   │
│  └───────────┼────────────────────────┼────────────────────────┘   │
│              │                        │                             │
│              │  GPU API calls         │                             │
│              │  (Vulkan ICD / CUDA)   │                             │
│              ▼                        ▼                             │
├─────────────────────────────────────────────────────────────────────┤
│  Container Runtime Layer (containerd / cri-o)                       │
│  - GPU device mounted via NVIDIA Container Toolkit                  │
│  - /dev/nvidia0 (or MIG device) exposed to container               │
├─────────────────────────────────────────────────────────────────────┤
│  RunAI Scheduling Layer (transparent to application)                │
│  - Allocates MIG slice or time-slice to pod                         │
│  - Enforces GPU memory limits                                       │
│  - Manages preemption and priority                                  │
├─────────────────────────────────────────────────────────────────────┤
│  Hardware Layer                                                     │
│  - H100 GPU with MIG partitions                                     │
│  - NVIDIA driver exposes partitions as separate devices             │
└─────────────────────────────────────────────────────────────────────┘
```

**How each engine interacts with RunAI-managed GPU**:

| Aspect | LiteRT-LM | vLLM | RunAI Role |
|--------|-----------|------|------------|
| **GPU API** | Vulkan (via ICD) | CUDA | None — standard APIs |
| **Device discovery** | `VK_ICD_FILENAMES` env var | `CUDA_VISIBLE_DEVICES` | RunAI sets env vars + device mounts |
| **Memory allocation** | Manual VRAM management | PagedAttention (dynamic) | Enforces limits via MIG or cgroups |
| **Model loading** | Local file (GGUF/TFLite) | HuggingFace/SafeTensors | None — app responsibility |
| **Batching** | None (single request) | Continuous batching | None — app responsibility |
| **Resource request** | Pod spec: `nvidia.com/gpu: 1` | Pod spec: `nvidia.com/gpu: 1` | Interprets request, allocates slice |

**What RunAI does NOT control**:
- Which inference engine runs (LiteRT-LM vs vLLM)
- How the engine uses GPU memory internally
- Model loading strategy (S3, PVC, baked-in)
- Request batching or queuing

**What RunAI DOES control**:
- Which GPU partition is assigned to the pod
- Maximum GPU memory available (via MIG slice boundaries)
- Scheduling priority and preemption
- Pod placement on GPU nodes

**Engine-specific RunAI considerations**:

| Engine | MIG Compatibility | Time-Slicing | VRAM Footprint | Notes |
|--------|-------------------|--------------|----------------|-------|
| **LiteRT-LM** | ✅ Works | ⚠️ Latency variance | ~6-8GB (Gemma 4B) | Vulkan ICD must be discoverable in container |
| **vLLM** | ✅ Works | ⚠️ Memory contention | ~8-10GB (Gemma 4B) | PagedAttention may improve utilization but adds overhead |

**Implications for deployment**:

1. **Engine choice is independent of RunAI**: Both engines work with RunAI-allocated GPUs via standard NVIDIA interfaces
2. **Container must include GPU userspace libs**: Vulkan ICD for LiteRT-LM, CUDA libs for vLLM
3. **RunAI sees pod, not engine**: GPU metrics from RunAI show pod-level utilization, not engine internals
4. **Model fits in slice**: Both engines must fit Gemma 4 E4B in 10GB MIG slice (A5 assumption)


### Costing Progression: Fractional GPU Model → Token Economics

This runbook starts with infrastructure-first costing assumptions (fractional GPU slice pricing) and must end with product-facing token economics for approval.

| Stage | Source Artifact | Output Metric | Primary Consumer |
|---|---|---|---|
| Infra baseline | [[19-production-cost-estimation-runbook]] | Monthly fixed/variable envelope by machine and GPU slice | Architecture + Finance |
| Runtime evidence | S1/S3/S7 measurements | Effective throughput, tail latency, VRAM headroom, uptime assumptions | Engineering |
| Workload mapping | A1-A4, A7 traffic/content assumptions + eval traces | Tokens/request and request concurrency envelope | Product + Engineering |
| Token economics synthesis | Phase 4 task + G9 package | Cost per 1K tokens, cost per conversation, monthly token capacity at SLA | Product Management |

**Required conversion outputs for G9**:
- `C_token_1k`: cost per 1,000 output-equivalent tokens under target SLA
- `C_conv`: cost per median and P95 conversation
- `Cap_month_tokens`: monthly token capacity at accepted latency and reliability
- Sensitivity bands for key assumptions (A2, A3, A5, A9, A11)
- Comparison against fallback/provider baseline where applicable

**Why this is necessary**:
Product approval is typically token-economics driven, while infrastructure planning starts as resource-economics. The roadmap explicitly bridges these two views so PM can judge necessity and sequencing of the technical work.

### Roadmap Visualization

```
Week 1          Week 2          Week 3          Week 4          Week 5
│               │               │               │               │
├─ S1: GPU+Nix ─┤               │               │               │
│   [CRITICAL]  ├─ S2: Model ───┤               │               │
│   DD-14/03    │   Loading     ├─ S5: Multi ───┤               │
│               ├─ S3: vLLM ────┤   Version     │               │
│               │   DD-06/14    ├─ S6: Dev ─────┤               │
│               ├─ S4: KServe ──┤   Workflow    ├─ S7: Eval ────┤
│               │               │   DD-13       │   CPU Deploy  │
│               │               │               ├─ Phase 4: ────┤
│               │               │               │   DD-15/16    │
▼               ▼               ▼               ▼               ▼
G1: GPU+Build   G2: Cold-Start  G4: Protocol   G5: Multi-Ver   Deploy
G1b: Container  G3: Engine                     G6: Eval CPU    G7/G8/G9
         │                                                      │
         └────────────────── DD-K8S-14 validated ────────────────────┘

---------------------------------

Goal
Implement a multi-tenant common dev environment, to deploy SLM for inference, model evals flows, fine-tuning processes

Scope
- Use both OTB and fine-tuned Gemma 4 E4B for inference, baselining and model evaluations
- Primary environment: shared Linux machine + remote GPU machine
- Secondary targets: macOS (dev approximation), iOS (via Apple hardware)

Use cases (technical)
1. Each dev can run parallel/multi-user model inference and Gemma 4 LLM evals in developer mode
2. A developer can access shared model artifacts and eval datasets without per-dev download
3. A developer can reproduce a baseline eval run and compare against team-shared references
4. A developer can run inference against multiple fine-tuned model versions and compare their behaviour
5. A developer can run data synthesis workflows independently of the inference server
6. A developer can clean and reset their local environment without affecting teammates
7. The team can validate GPU readiness before scheduling expensive inference or eval runs
