# 19 Production Cost Estimation Runbook

**Author**: IV SAI Innovation lead, August 21, 2026  
**Status**: Active analysis (interactive session)

## Purpose

Estimate production runtime costs for a shopping AI assistant application deployed on Kubernetes, using components from `src/inference_srv_py` and `src/shopping_controller`. This runbook captures scenarios, assumptions, risks, and design decisions for production deployment sizing.

## Projected User Volumes

| Metric | Value | Notes |
|--------|-------|-------|
| Daily average users | 1,000 | Unique users per day |
| Concurrent users (off-peak) | 18 | Baseline load |
| Concurrent users (peak) | 65 | Peak load scenario |
| Conversations per user/day | 1–3 avg | Shopping assistance sessions |
| Turns per conversation | 3–10 | Multi-turn dialogue |

### Derived Request Volume Estimates

| Scenario | Calculation | Daily Requests | Peak RPS (sustained 5min) |
|----------|-------------|----------------|---------------------------|
| Low | 1000 users × 1 conv × 3 turns | 3,000 | ~0.035 |
| Medium | 1000 users × 2 conv × 6 turns | 12,000 | ~0.14 |
| High | 1000 users × 3 conv × 10 turns | 30,000 | ~0.35 |
| Peak burst | 65 concurrent × 10 turns/5min | 130 req/5min | **0.43 RPS** |

**Observation**: Even at peak, this is extremely low RPS. Cost drivers will be GPU idle time and cold start latency, not throughput.

---

## Machine Topology

The production environment consists of **two distinct machines** with complementary roles. See [[18-k8s-inference-deployment-analysis-runbook]] for detailed architecture.

| Identifier | Name | Role | Hardware |
|------------|------|------|----------|
| **MI_machine** | Model Inference GPU Machine | Production inference serving | NVIDIA H100 GPU (MIG 1/7 slice) |
| **ME_machine** | Model Evals CPU Machine | Development + Eval orchestration | 8-core CPU, 32GB RAM, 7 dev accounts |

### Machine Cost Breakdown

| Machine | Component | Symbol | Monthly Cost |
|---------|-----------|--------|--------------|
| MI_machine | H100 MIG 1/7 slice | $G_{mig}$ | $400 |
| MI_machine | CPU/orchestration overhead | $G_{mi\_cpu}$ | $50 |
| ME_machine | 8-core CPU, 32GB RAM | $G_{me}$ | $150 |
| Shared | S3/MinIO storage | $G_{stor}$ | $10 |
| **Total infrastructure** | | | **$610** |

---

## Cost Model Definition

### Variables

| Symbol | Definition | Machine | Unit | Value Range |
|--------|------------|---------|------|-------------|
| $U$ | Daily active users | — | users/day | 1,000 |
| $C$ | Conversations per user per day | — | conv/user/day | 1–3 |
| $T$ | Turns per conversation | — | turns/conv | 3–10 |
| $D$ | Days per month | — | days/mo | 30 |
| $O_{avg}$ | Average output tokens per turn | MI_machine | tokens/turn | 450 |
| $I_n$ | Input tokens at turn $n$ | MI_machine | tokens | see formula |
| $f$ | Cloud fallback rate | — | % | 0.10 (10%) |
| $P_{in}$ | Cloud LLM input price | — | $/1M tokens | 0.075 |
| $P_{out}$ | Cloud LLM output price | — | $/1M tokens | 0.30 |
| $G_{ded}$ | Dedicated H100 monthly cost | MI_machine | $/mo | 2,500 |
| $G_{mig}$ | H100 MIG slice monthly cost (1/7) | MI_machine | $/mo | 400 |
| $G_{mi\_cpu}$ | MI_machine CPU overhead | MI_machine | $/mo | 50 |
| $G_{me}$ | ME_machine monthly cost | ME_machine | $/mo | 150 |
| $G_{stor}$ | Shared storage monthly cost | Both | $/mo | 10 |

### Derived Formulae

**Monthly conversations:**
$$N_{conv} = U \times C \times D$$

**Monthly requests (turns):**
$$N_{req} = N_{conv} \times T$$

**Input tokens at turn $n$** (stateless API sends full history):
$$I_n = I_0 + \sum_{k=1}^{n-1}(I_k + O_k) \approx 200 + (n-1) \times 650$$

Where $I_0 = 200$ (system prompt + first query), and each prior exchange adds ~650 tokens (user turn ~200 + assistant response ~450).

**Total input tokens per conversation:**
$$I_{conv} = \sum_{n=1}^{T} I_n = T \times I_0 + \frac{T(T-1)}{2} \times 650$$

**Total output tokens per conversation:**
$$O_{conv} = T \times O_{avg}$$

**Total tokens per conversation:**
$$\tau_{conv} = I_{conv} + O_{conv}$$

**Monthly token volume:**
$$\tau_{month} = N_{conv} \times \tau_{conv}$$

**Cloud fallback cost:**
$$C_{cloud} = f \times \tau_{month} \times \left( \frac{I_{conv}}{\tau_{conv}} \times P_{in} + \frac{O_{conv}}{\tau_{conv}} \times P_{out} \right) \times 10^{-6}$$

**Infrastructure cost (two-machine model):**
$$C_{infra} = \underbrace{(G_{mig} + G_{mi\_cpu})}_{\text{MI\_machine}} + \underbrace{G_{me}}_{\text{ME\_machine}} + G_{stor}$$

**Simplified for Scenario B (fractional GPU):**
$$C_{infra} = 400 + 50 + 150 + 10 = 610$$

**Total monthly cost:**
$$C_{total} = C_{infra} + C_{cloud}$$

### Reference Calculations

**Tokens per conversation by scenario:**

| Scenario | $T$ | $I_{conv}$ | $O_{conv}$ | $\tau_{conv}$ |
|----------|-----|------------|------------|---------------|
| Low | 3 | $3(200) + \frac{3(2)}{2}(650) = 2,550$ | $3(450) = 1,350$ | **3,900** |
| Medium | 6 | $6(200) + \frac{6(5)}{2}(650) = 10,950$ | $6(450) = 2,700$ | **13,650** |
| High | 10 | $10(200) + \frac{10(9)}{2}(650) = 31,250$ | $10(450) = 4,500$ | **35,750** |

*Note: Simplified formula assumes linear growth; actual is ~2,550 / 7,500 / 15,000 from earlier empirical estimates. Using conservative higher bounds here.*

**Monthly token volume:**

| Scenario | $N_{conv}$ | $\tau_{conv}$ | $\tau_{month}$ |
|----------|------------|---------------|----------------|
| Low | 30,000 | 3,900 | **117M** |
| Medium | 60,000 | 13,650 | **819M** |
| High | 90,000 | 35,750 | **3,218M** |

**Cloud fallback cost ($f = 0.10$, Gemini Flash):**

| Scenario | Fallback Tokens | Input ~60% | Output ~40% | **$C_{cloud}$** |
|----------|-----------------|------------|-------------|-----------------|
| Low | 11.7M | $0.53 | $1.40 | **~$2/mo** |
| Medium | 81.9M | $3.68 | $9.83 | **~$14/mo** |
| High | 321.8M | $14.48 | $38.61 | **~$53/mo** |

**Total monthly cost (Scenario B: H100 MIG, two-machine model):**

| Scenario | MI_machine | ME_machine | Storage | $C_{cloud}$ | **$C_{total}$** |
|----------|------------|------------|---------|-------------|-----------------|
| Low | $450 | $150 | $10 | $2 | **$612/mo** |
| Medium | $450 | $150 | $10 | $14 | **$624/mo** |
| High | $450 | $150 | $10 | $53 | **$663/mo** |

---

## Current Architecture Components

### 1. Inference Server (`inference_srv_py`)

**Machine**: MI_machine (GPU)

| Aspect | Current State | Production Implication |
|--------|---------------|------------------------|
| Server model | `ThreadingHTTPServer` (Python stdlib) | Single-process, thread-per-request |
| Concurrency | Thread-based, GIL-bound | Limited parallelism for I/O; GPU inference serialized |
| Runtime | LiteRT-LM with GPU backend | Requires GPU node, Vulkan ICD |
| Model | Gemma 4 E4B | ~4B params, quantized |
| API | `/v1/chat/completions` (stateless) | Full history per request |
| State | Stateless per request | Conversation state in client |

### 2. Shopping Controller (`shopping_controller`)

**Machine**: ME_machine (CPU) — dev/eval only, not deployed to production

**Note**: DSPy/shopping_controller is **excluded from production architecture**. Production uses direct LiteRT inference only.

| Aspect | Dev/Eval State | Production State |
|--------|----------------|------------------|
| Framework | DSPy + BAML validation | **Not deployed** |
| Routing | 3-way classification | Single LLM call |
| Agents | Multi-agent orchestration | **Excluded** |
| LLM Backend | LiteLLM adapter | LiteRT-LM direct |

This simplification removes the DSPy call amplification concern.

---

## Architecture Scenarios for Cost Analysis

### Scenario A: Self-Hosted GPU (On-Prem K8s)

**Topology**: K8s cluster with dedicated GPU nodes running LiteRT-LM.

| Component | Sizing | Monthly Cost Estimate |
|-----------|--------|----------------------|
| GPU node | 1× NVIDIA H100 80GB | ~$2,500/mo (cloud equiv) or ~$35K CapEx |
| CPU workers | 2× 4vCPU/8GB | ~$100/mo |
| Storage | 50GB SSD (models) | ~$10/mo |
| Network | Internal only | Negligible |
| **Subtotal** | | **~$2,610/mo** |

**H100 Pricing Context**:

| Provider | H100 Type | Hourly | Monthly (730h) |
|----------|-----------|--------|----------------|
| GCP | H100 80GB | ~$3.50 | ~$2,555 |
| AWS | p5.xlarge (H100) | ~$4.00 | ~$2,920 |
| Lambda Labs | H100 PCIe | ~$2.50 | ~$1,825 |
| On-prem | CapEx | — | ~$35,000 one-time |

**Assumptions**:
- GPU node always on (cannot scale to zero easily)
- Model loaded once, amortized across all sessions
- No autoscaling—fixed capacity
- H100 massively overprovisioned for 4B model (could serve 10× load)

**Risks**:
- [ ] GPU utilization extremely low (<1% at 1000 DAU)—severe overprovisioning
- [ ] CapEx lock-in if self-hosted (~3yr payback vs cloud)
- [ ] Cold start latency if pod evicted

---

### Scenario B: Fractional GPU (RunAI / MIG Scheduling)

**Topology**: Shared GPU pool with fractional allocation via RunAI scheduler.

**H100 MIG Partitioning**:

| MIG Profile | VRAM | Compute (SMs) | Monthly Cost Est. |
|-------------|------|---------------|-------------------|
| 1g.10gb | 10GB | 1/7 | ~$360–400/mo |
| 2g.20gb | 20GB | 2/7 | ~$720–800/mo |
| 3g.40gb | 40GB | 3/7 | ~$1,100–1,200/mo |
| 7g.80gb | 80GB | Full | ~$2,500/mo |

*Gemma 4 E4B (quantized) requires ~6–8GB VRAM → fits in 1g.10gb slice*

| Component | Sizing | Monthly Cost Estimate |
|-----------|--------|----------------------|
| Fractional GPU | 1/7× H100 (1g.10gb MIG) | ~$360–400/mo |
| CPU workers | 2× 4vCPU/8GB | ~$100/mo |
| Scheduler overhead | RunAI license | Enterprise pricing |
| **Subtotal** | | **~$460–500/mo** + licensing |

**Assumptions**:
- Workload can tolerate MIG scheduling latency (<100ms)
- 4B quantized model fits in 10GB MIG slice
- Fair-share scheduling acceptable for latency SLA
- MIG isolation provides predictable performance

**Risks**:
- [ ] MIG slice memory pressure if model exceeds 8GB
- [ ] Scheduling contention during peak hours
- [ ] Vendor lock-in to RunAI APIs
- [ ] MIG reconfiguration requires GPU quiesce

---

### Scenario C: Managed Inference API (Cloud LLM) — **Fallback Only**

**Topology**: Cloud endpoint (Vertex AI, Bedrock, OpenAI) as fallback for edge failures.

**Pricing model** (see [Cost Model Definition](#cost-model-definition)):

| Provider | $P_{in}$ ($/1M tokens) | $P_{out}$ ($/1M tokens) | $\bar{P}$ weighted |
|----------|------------------------|-------------------------|-------------------|
| Gemini Flash | $0.075 | $0.30 | ~$0.165 |
| GPT-4o-mini | $0.15 | $0.60 | ~$0.33 |
| Claude Haiku | $0.25 | $1.25 | ~$0.65 |

#### Token Estimates Reference

**Output tokens by response type** ($O_{avg} = 450$ is weighted average):

| Response Type | Description | Output Tokens |
|---------------|-------------|---------------|
| Clarification | "What size? What color preference?" | 30–80 |
| Single product | JSON: name, brand, price, features, URL | 100–150 |
| 3-product comparison | 3× product cards + intro/summary | 400–600 |
| 5-product list | 5× product cards + pagination | 700–1,000 |

**Input tokens grow with conversation history** (formula: $I_n = 200 + (n-1) \times 650$):

| Turn $n$ | $I_n$ | Cumulative $\sum I$ |
|----------|-------|---------------------|
| 1 | 200 | 200 |
| 3 | 1,500 | 2,550 |
| 6 | 3,450 | 10,950 |
| 10 | 6,050 | 31,250 |

#### Cloud Cost by Scenario (Gemini Flash, $f=0.10$)

Applying formula: $C_{cloud} = f \times \tau_{month} \times \bar{P} \times 10^{-6}$

| Scenario | $\tau_{month}$ | $C_{cloud}$ (10%) | $C_{cloud}$ (100%) |
|----------|----------------|-------------------|-------------------|
| Low | 117M | **$2** | **$19** |
| Medium | 819M | **$14** | **$135** |
| High | 3,218M | **$53** | **$531** |

*Note: Higher-tier models (GPT-4o, Claude Sonnet) would be 2–10× more expensive.*

**Assumptions**:
- Cloud model performance acceptable for fallback
- Latency (~200–500ms) tolerable for fallback UX
- No fine-tuning required (or hosted fine-tune available)

**Risks**:
- [ ] Vendor pricing changes
- [ ] Data residency/compliance
- [ ] No offline capability (mobile use case)

---

### Scenario D: Hybrid (Edge + Cloud Fallback)

**Topology**: On-device inference for simple queries, cloud fallback for complex routing.

| Component | Sizing | Formula | Monthly Cost |
|-----------|--------|---------|--------------|
| Edge runtime | User devices | N/A | $0 |
| Cloud fallback ($f=0.2$) | 20% of traffic | $0.2 \times 819M \times 0.165 \times 10^{-6}$ | ~$27/mo (medium) |
| Orchestration | API Gateway | Fixed | ~$20/mo |
| **Subtotal** | | | **~$47/mo** |

**Assumptions**:
- 80% of queries handled on-device
- User devices have sufficient compute
- Acceptable latency variance between paths

**Risks**:
- [ ] Device capability fragmentation
- [ ] Complex testing matrix
- [ ] UX consistency across paths

---

## Consolidated Cost Summary

**Production architecture**: LiteRT-LM on H100 (primary) + Cloud LLM (fallback)  
**Machine topology**: MI_machine (GPU inference) + ME_machine (CPU dev/eval)  
**GPU**: NVIDIA H100 80GB (1/7 MIG slice)  
**Cost model**: See [Cost Model Definition](#cost-model-definition) for formulae

### Two-Machine Cost Breakdown

| Machine | Component | Monthly Cost |
|---------|-----------|--------------|
| MI_machine | H100 MIG 1/7 slice | $400 |
| MI_machine | CPU/orchestration | $50 |
| ME_machine | 8-core CPU, 32GB RAM | $150 |
| Shared | S3/MinIO storage | $10 |
| **Subtotal (infrastructure)** | | **$610** |

### Summary by Architecture Scenario

Using medium traffic ($C=2$, $T=6$, $\tau_{month} \approx 819M$ tokens):

| Scenario | MI_machine | ME_machine | Cloud | **$C_{total}$** | Meets SLA? |
|----------|------------|------------|-------|-----------------|------------|
| A: Dedicated H100 | $2,550 | $150 | +$14 | **$2,714** | ✅ (overprovisioned) |
| B: Fractional H100 (1/7 MIG) | $450 | $150 | +$14 | **$624** | ✅ |
| C: Cloud only ($f=1.0$) | $0 | $150 | +$139 | **$289** | ⚠️ Higher latency |
| D: Edge + Cloud ($f=0.2$) | $0 | $20 | +$28 | **$48** | ⚠️ Device-dependent |

Where $\bar{P} = 0.6 \cdot P_{in} + 0.4 \cdot P_{out} \approx 0.165$ $/M tokens (weighted average).

### Scenario B Cost by Traffic Level (Two-Machine Model)

| Traffic | MI_machine | ME_machine | Storage | $C_{cloud}$ (10%) | **$C_{total}$** |
|---------|------------|------------|---------|-------------------|-----------------|
| Low | $450 | $150 | $10 | $2 | **$612** |
| Medium | $450 | $150 | $10 | $14 | **$624** |
| High | $450 | $150 | $10 | $53 | **$663** |

**Sensitivity: Fallback rate ($f$)**

| $f$ | Low $C_{cloud}$ | Med $C_{cloud}$ | High $C_{cloud}$ |
|-----|-----------------|-----------------|------------------|
| 5% | $1 | $7 | $27 |
| 10% | $2 | $14 | $53 |
| 20% | $4 | $28 | $106 |
| 50% | $10 | $70 | $265 |

### H100 vs Alternative GPUs (MI_machine only)

| GPU | VRAM | $G_{ded}$ | $G_{mig}$ (1/7) | Notes |
|-----|------|-----------|-----------------|-------|
| H100 80GB | 80GB | ~$2,500 | ~$400 | Required spec |
| A100 80GB | 80GB | ~$1,500 | ~$250 | Older gen, MIG capable |
| L4 24GB | 24GB | ~$350 | N/A | No MIG support |

### Recommendation

**Scenario B (Fractional H100 MIG)** with two-machine model + 10% cloud fallback:
- Formula: $C_{total} = (G_{mig} + G_{mi\_cpu}) + G_{me} + G_{stor} + f \cdot \tau_{month} \cdot \bar{P}$
- Range: **$612–663/mo** across traffic scenarios
- MI_machine: ~$450/mo for inference (75% of infra cost)
- ME_machine: ~$150/mo for dev + eval (25% of infra cost)
- Cloud fallback provides resilience at marginal cost (<10% of total)
- Requires validation: model fits in 10GB MIG slice

---

## Latency SLA Requirements

| Metric | Target | Implication |
|--------|--------|-------------|
| LLM inference latency | **1.5–2.0s** | Tight; rules out cold-start-from-zero |
| Time to first token | <500ms | Requires warm GPU pod |
| End-to-end response | <3s | Network + inference + serialization |

**Implication**: GPU pods must remain warm. Scale-to-zero is **not viable** with this SLA. Minimum 1 replica always running.

---

## Implicit Assumptions to Validate

| # | Assumption | Validation Method |
|---|------------|-------------------|
| A1 | Average conversation fits in context window | Measure actual conversation token lengths |
| A2 | 65 concurrent users means 65 simultaneous requests | Profile actual request overlap distribution |
| A3 | Assistant responses average 450 tokens (product-heavy) | Corpus analysis of shopping responses |
| A4 | Shopping domain doesn't require retrieval augmentation | Evaluate catalog size and search patterns |
| A5 | Gemma 4 E4B fits in H100 1g.10gb MIG slice (<10GB) | Profile actual VRAM usage |
| A6 | H100 MIG warm-start latency <500ms | Benchmark LiteRT-LM on H100 MIG |
| A7 | Model quality at 4B params is sufficient | Eval metrics from `llmeval_framework` |
| A8 | 10% fallback rate to cloud is sufficient | Monitor edge failure rates |
| A9 | H100 MIG provides sufficient compute for 1.5s inference | Benchmark token/s on 1g.10gb profile |
| A10 | ME_machine can support 7 concurrent dev shells | Profile CPU/RAM usage per dev |

---

## Design Decisions Surfaced

| Decision | Machine | Options | Current Lean | Rationale |
|----------|---------|---------|--------------|-----------|
| DD-01: Machine topology | Both | Single / Two-machine | **Two-machine** | Separate GPU (MI) from dev/eval (ME) |
| DD-02: GPU hardware | MI_machine | H100 / A100 / L4 | **H100** | Required spec; MIG support |
| DD-03: GPU allocation model | MI_machine | Dedicated / Fractional / Cloud | **Fractional (1/7 MIG)** | ~$450/mo vs $2,550/mo |
| DD-04: Statelessness boundary | MI_machine | Client-side history / Server sessions | Client-side | Simpler scaling, OpenAI-compat |
| DD-05: Production LLM stack | MI_machine | DSPy multi-agent / Direct inference | **Direct inference** | DSPy excluded; single LLM call |
| DD-06: Scaling strategy | MI_machine | Fixed / HPA / Serverless | **HPA min=1** | Cannot scale to zero (SLA) |
| DD-07: Cloud fallback role | Both | Primary / Fallback / None | **Fallback only** | Edge failures, capacity overflow |
| DD-08: Fallback trigger | Both | Latency / Error / Load | Error + Load >80% | Preserve SLA during spikes |
| DD-09: Dev shell isolation | ME_machine | Per-user accounts / Containers | **Per-user accounts** | Simpler, Linux native |

---

## Business Requirements Implications

| BR Reference | Machine | Implication for Cost |
|--------------|---------|---------------------|
| [[05-br-conversation-oriented-inference-experience]] | MI_machine | Must amortize runtime prep; favors persistent GPU |
| [[03-br-apple-deployment-authority]] | ME_machine | Mobile edge path may reduce cloud dependency |
| [[04-br-linux-sandbox-approximation-requirements]] | MI_machine | GPU isolation adds overhead |

---

## Open Questions (Interactive Analysis)

1. **Model selection trade-offs**: Is Gemma 4 E4B the right size, or would a smaller/larger model shift cost-quality balance?

2. **Session affinity**: Would sticky sessions to warm GPU pods improve latency enough to justify complexity?

3. **Batch inference windows**: Could async/batch processing reduce peak GPU requirements?

4. **Fallback threshold tuning**: At what load percentage should traffic shift to cloud? 70%? 80%?

5. **Context window management**: At 2,500 tokens by turn 10, should we implement sliding window or summarization?

6. **Product data retrieval**: Are product descriptions/images fetched separately (reducing LLM output) or embedded in response?

7. **H100 MIG availability**: Is 1g.10gb MIG profile available in target K8s cluster? Or only larger slices?

---

## Next Analysis Steps

- [ ] Benchmark LiteRT-LM warm inference latency on H100 1g.10gb MIG (target <1.5s)
- [ ] Profile Gemma 4 E4B VRAM usage on H100 MIG slice
- [ ] Validate 4B model fits in 10GB MIG slice with headroom
- [ ] Build token consumption model from shopping conversation corpus
- [ ] Define fallback trigger thresholds (error rate, load %)
- [ ] Evaluate context window management strategy (sliding vs summarization)
- [ ] Measure actual concurrent request overlap at 65 peak users
- [ ] Compare H100 MIG vs dedicated H100 latency characteristics

---

## Appendix: Pricing References (August 2026)

| Provider | Model | Input $/1M tokens | Output $/1M tokens |
|----------|-------|-------------------|-------------------|
| GCP Vertex | Gemini 1.5 Flash | $0.075 | $0.30 |
| GCP Vertex | Gemini 1.5 Pro | $1.25 | $5.00 |
| AWS Bedrock | Claude 3 Haiku | $0.25 | $1.25 |
| AWS Bedrock | Claude 3 Sonnet | $3.00 | $15.00 |
| OpenAI | GPT-4o-mini | $0.15 | $0.60 |
| OpenAI | GPT-4o | $2.50 | $10.00 |

*Note: Self-hosted GPU cost competitive below ~100M tokens/month at current volumes.*

---

## References

- [[17-multi-tenant-devenv-use-cases]] - Target deployment environment
- [[18-k8s-inference-deployment-analysis-runbook]] - K8s porting analysis
- [[03-dd-runtime-adapter-pattern]] - Runtime abstraction layer
- [[07-dd-backend-conversation-contract]] - API contract
