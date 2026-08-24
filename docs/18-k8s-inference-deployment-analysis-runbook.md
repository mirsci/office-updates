# 18 K8s Inference Deployment Analysis Runbook

**Author**: IV SAI Innovation lead, August 20, 2026  
**Status**: Active analysis

## Purpose

This runbook captures the interactive analysis for porting `src/inference_srv_py` to a Kubernetes-based multi-tenant development environment with fractional GPU access. The target runtime uses a **fine-tuned Gemma 4 E4B** model and must support the validated use cases in [[17-multi-tenant-devenv-use-cases]].

## Target Environment Characteristics

| Capability | Provided | Constraints |
|---|---|---|
| Compute | NVIDIA GPU nodes | Fractional GPUs (likely via RunAI scheduling or MIG) |
| Storage | PVC or S3 | Must adapt from current MinIO-based object boundary |
| Runtime | KServe Python SDK | Custom runtime images required |
| Orchestration | RunAI + Canopy CLI | Limited direct GPU hardware control |
| Model governance | Custom model catalogue/registry | New integration surface |

## Current Implementation Summary

The `inference_srv_py` module provides:

| Component | Location | Description |
|---|---|---|
| HTTP server | `server.py` | ThreadingHTTPServer with `/health`, `/ready`, `/v1/chat/completions` |
| Backend service | `backend.py` | LiteRT-LM engine lifecycle, conversation session mgmt |
| GPU validation | `gpu_validation.py` | Preflight checks for Vulkan/NVIDIA readiness |
| Bootstrap | `bootstrap.py` | Model path discovery, environment bootstrap state |

Key runtime characteristics:
- Single-process Python server with thread-per-request
- OpenAI-compatible `/v1/chat/completions` contract (stateless, full message history per request)
- `EngineRuntime` abstraction with `runtime_factory` injection point (currently only `LiteRTEngineRuntime` implemented)
- Environment-driven configuration (`HYBRID_AI_HOST`, `HYBRID_AI_PORT`, model paths)
- Current GPU boundary requires narrow host contract (Vulkan ICD, no broad LD_LIBRARY_PATH)

References: [[03-dd-runtime-adapter-pattern]], [[07-dd-backend-conversation-contract]]

### Flox/Nix Environment Architecture

The current implementation uses **Flox-composed Nix environments** for reproducibility:

| Environment | Path | Purpose |
|---|---|---|
| `base` | `env/base/` | Shared tools (git, shell policy) |
| `python` | `env/python/` | Python 3.11, poetry, uv (includes base) |
| `inference-litert-linux-gpu` | `env/inference-litert-linux-gpu/` | Vulkan, NVIDIA drivers, LiteRT-LM GPU path (includes python) |
| `llmeval-framework` | `env/llmeval-framework/` | Eval harness dependencies |

**Key Flox/Nix characteristics**:
- Python venv managed under `.flox/cache/python/` via hook scripts
- GPU dependencies pinned via Nix: `vulkan-loader`, `vulkan-tools`, `linuxPackages.nvidia_x11`
- Environment variables set in manifest: `HYBRID_AI_INFERENCE_ENGINE`, `HYBRID_AI_LITERT_BACKEND`
- Activation scripts: `scripts/env/toolchain/inference_srv_py/inference_srv_py_env.sh`, `scripts/env/toolchain/inference/litert_env.sh`
- Poetry lockfile (`poetry.lock`) pins Python dependencies

**K8s deployment options**:
- Flox provides **`flox containerize`** command to directly produce OCI images from Flox environments
- Alternatively: Nix `dockerTools.buildLayeredImage` for layered images with dependency caching
- Container images derived from Flox/Nix maintain full reproducibility guarantees

**Design decisions covering K8s adaptation**:
- **DD-K8S-05**: Container Image Build Strategy — how to produce OCI images from Flox environments
- **DD-K8S-07**: Flox Environment to K8s Deployment Mapping — how manifest elements map to K8s artifacts
- **BR-K8S-07**: Environment Reproducibility Preservation — requirement to maintain Flox/Nix guarantees

References: [[devenv_portable_workflow]], [[15-common-development-environment-for-llm-delivery]], [Flox Nix and Containers blog](https://flox.dev/blog/nix-and-containers-why-not-both)

---

## Analysis Scope

### In-Scope

1. KServe runtime image adaptation
2. Model artifact distribution via S3/PVC
3. Multi-tenant request isolation
4. GPU resource boundary for fractional allocation
5. Integration with model registry
6. Support for validated use cases:
   - UC_DEV_04: Parallel multi-user inference in developer mode
   - UC_DEV_09: Multi-version fine-tuned model comparison (side-by-side eval)
   - UC_DEV_13: GPU readiness validation before expensive runs
7. Multi-engine support (LiteRT-LM + vLLM) behind unified OpenAI-compatible contract
8. Eval suite deployment on CPU-only pods (cost optimization)

### Out-of-Scope (for this phase)

1. Apple-platform deployment
2. On-device inference runtime
3. Full CI/CD pipeline design
4. Cost optimization

---

## Identified Business Requirements (Draft)

These requirements are captured here during analysis. Once validated, they would normally be promoted to `docs/business-domain/`.

### Business Requirement Summary

| ID | Title | Key Concern |
|---|---|---|
| BR-K8S-01 | Multi-Tenant Inference Isolation | Request/session isolation with fractional GPU |
| BR-K8S-02 | Multi-Version Fine-Tuned Model Governance | Version control, registry, side-by-side comparison |
| BR-K8S-03 | GPU Resource Boundary Portability | Preserve narrow Vulkan bridge in container |
| BR-K8S-04 | Development Integration Environment Parity | Support UC_DEV_04, UC_DEV_08 workflows |
| BR-K8S-05 | Backend Contract Stability | Preserve `/v1/chat/completions` for clients/eval |
| BR-K8S-06 | Multi-Engine Inference Support | LiteRT-LM + vLLM behind unified contract |
| BR-K8S-07 | Environment Reproducibility Preservation | Maintain Flox/Nix guarantees in K8s |
| BR-K8S-08 | Eval Suite CPU Separation | Run eval on CPU pods, inference on GPU — cost optimization |

---

### BR-K8S-01: Multi-Tenant Inference Isolation

**Intent**: Each development team member must be able to run inference workloads without interfering with other team members' sessions.

**Constraints**:
- Fractional GPU means VRAM is shared at hardware level
- K8s namespace or pod-level isolation must provide request isolation
- Session state (conversation context) must not leak between tenants

**Acceptance criteria**:
1. Concurrent requests from different users produce independent results
2. A crashed or slow request from one user does not block others
3. Resource exhaustion by one tenant triggers graceful degradation, not cluster-wide failure

**Open questions**:
- [ ] Does RunAI provide per-user quotas or is this namespace-level only?
- [ ] How does KServe handle request affinity for stateful conversations?

---

### BR-K8S-02: Multi-Version Fine-Tuned Model Governance

**Intent**: Multiple fine-tuned versions of Gemma 4 E4B must be deployable, version-controlled, and comparable side-by-side.

**References**: UC_DEV_09 in [[17-multi-tenant-devenv-use-cases]] validates multi-version comparison on current infrastructure.

**Existing capability** (from UC_DEV_09):
- `HYBRID_AI_MODEL_PATH` env var selects checkpoint at server startup
- Multiple server instances can run side-by-side on different ports
- Eval suite targets server via `HYBRID_AI_BACKEND_BASE_URL`
- Model identity pinned in `volumes/models/litert-lm/litert-lm.model*` metadata

**K8s adaptation required**:
- Model artifacts in S3/PVC instead of local `volumes/models/`
- Model registry replaces local metadata files
- Pod-level model version selection (env var or label)
- Side-by-side deployment for A/B eval without port management

**Constraints**:
- Model size is multi-GB (Gemma 4 E4B)
- Model updates must be atomic/versioned
- Model registry integration required
- Must support both baseline and fine-tuned variants simultaneously

**Acceptance criteria**:
1. Model version is traceable from inference response to registry artifact
2. Model rollback is possible without redeploying infrastructure
3. Model loading uses cached artifacts to minimize cold-start latency
4. Two model versions can be deployed simultaneously for A/B comparison
5. Eval suite can target specific model version via URL/routing

**Open questions**:
- [ ] What is the model registry API surface?
- [ ] Does the registry support immutable versioning or mutable tags?
- [ ] Is there a preferred artifact format (GGUF, TFLite, SafeTensors)?
- [ ] How are model versions routed — separate services, labels, or headers?

---

### BR-K8S-03: GPU Resource Boundary Portability

**Intent**: The current narrow GPU bridge model (Vulkan ICD discovery, no LD_LIBRARY_PATH mutation) must be preserved or adapted to the K8s GPU operator model.

**Constraints**:
- K8s NVIDIA device plugin manages GPU allocation
- RunAI may abstract GPU scheduling
- Container runtime (containerd/cri-o) determines device exposure

**Acceptance criteria**:
1. GPU validation preflight passes within container
2. LiteRT-LM engine initializes without host linker workarounds
3. Vulkan ICD is discoverable via standard container paths

**Open questions**:
- [ ] Does the K8s cluster use NVIDIA device plugin or RunAI's native resource allocation?
- [ ] Are fractional GPUs via MIG or time-slicing?
- [ ] What base image is required for LiteRT-LM + Vulkan?

---

### BR-K8S-04: Development Integration Environment Parity

**Intent**: The K8s deployment must support the same developer workflows validated in [[17-multi-tenant-devenv-use-cases]], particularly UC_DEV_04 (parallel inference) and UC_DEV_08 (eval execution).

**Constraints**:
- Developers may not have direct `kubectl` access
- Canopy CLI is the primary interaction surface
- Port forwarding or ingress required for local dev clients

**Acceptance criteria**:
1. Developer can target their namespace/pod for eval runs
2. Backend contract (`/v1/chat/completions`) is unchanged
3. Health/readiness probes work with K8s liveliness checks

**Open questions**:
- [ ] What is the Canopy CLI workflow for deploying updated images?
- [ ] Is there a dev-sandbox vs shared-staging separation?
- [ ] Can developers override model version for A/B testing?

---

### BR-K8S-05: Backend Contract Stability

**Intent**: The existing backend contract (HTTP endpoints, error semantics, streaming behavior) must be preserved to maintain compatibility with Swift app clients and eval harnesses.

**References**: [[07-dd-backend-conversation-contract]], [[08-dd-streaming-chat-semantics]], [[10-dd-backend-error-semantics]]

**Constraints**:
- KServe custom runtime may impose its own endpoint conventions
- Load balancing may affect streaming SSE behavior

**Acceptance criteria**:
1. `/v1/chat/completions` request/response schema unchanged
2. `/health` and `/ready` semantics match K8s probe expectations
3. Error codes and messages follow existing contract

**Open questions**:
- [ ] Does KServe require specific endpoint paths (e.g., `/v1/models/{model_name}:predict`)?
- [ ] How does KServe handle long-running streaming responses?

---

### BR-K8S-06: Multi-Engine Inference Support

**Intent**: The deployment must support both LiteRT-LM and vLLM inference engines concurrently, with engine selection per-request or per-deployment.

**Rationale**:
- LiteRT-LM: Optimized for on-device and edge scenarios, current baseline engine
- vLLM: High-throughput serving, better GPU utilization via PagedAttention, production-grade batching

**Constraints**:
- Both engines must satisfy the same `EngineRuntime` contract
- Model formats may differ (LiteRT-LM uses TFLite/GGUF, vLLM uses HuggingFace/SafeTensors)
- GPU memory management differs significantly between engines

**Acceptance criteria**:
1. A single deployment can route requests to either engine based on configuration
2. Engine selection is transparent to clients (same `/v1/chat/completions` contract)
3. Model registry tracks which engine(s) a model artifact supports
4. Health/ready probes report engine-specific status

**Open questions**:
- [ ] Is engine selection per-pod (sidecar model) or per-request (router model)?
- [ ] Does the fine-tuned Gemma 4 E4B have artifacts for both engines?
- [ ] How does vLLM integrate with RunAI fractional GPU allocation?
- [ ] Should there be separate scaling policies per engine type?

**Design impact**: Requires DD-K8S-06 (Multi-Engine Abstraction Layer).

---

### BR-K8S-07: Environment Reproducibility Preservation

**Intent**: The K8s deployment must preserve the reproducibility guarantees currently provided by Flox/Nix environments.

**Current state**:
- Local dev uses Flox-composed Nix environments with pinned closures
- `poetry.lock` pins Python dependencies
- Nix manifest pins system dependencies (Vulkan, NVIDIA libs)
- Any developer running `flox activate` gets identical environment

**K8s adaptation required**:
- Container image must derive from same dependency pins
- Build process must be reproducible (same inputs → same image)
- Runtime behavior must match local dev behavior

**Constraints**:
- K8s ops teams may not have Nix expertise
- CI/CD may not have Nix available
- Image size and build time must be acceptable

**Acceptance criteria**:
1. Container image produces identical inference results to local Flox environment
2. Dependency versions in container match `poetry.lock` and Nix manifest
3. GPU initialization behavior matches local GPU validation
4. Build process is documented and reproducible by ops team

**Open questions**:
- [ ] Is Nix available in CI/build environment?
- [ ] What is ops team's familiarity with Nix vs traditional Docker?
- [ ] Can we provide both Nix-native and Dockerfile builds?

---

### BR-K8S-08: Eval Suite CPU Separation

**Intent**: The LLM eval suite (`llmeval_suite`, `llmeval_framework`) must run on CPU-only pods while inference runs on GPU pods, optimizing GPU cost.

**Rationale**:
- GPU time is expensive (fractional GPU still has cost)
- Eval suite only makes HTTP calls to inference endpoint — no local GPU needed
- Eval workloads can be parallelized on cheap CPU nodes
- GPU pods should be reserved for inference serving, not eval orchestration

**Current architecture** (local dev):
```
┌─────────────────┐      HTTP       ┌─────────────────┐
│  llmeval_suite  │ ──────────────► │ inference_srv_py │
│  (CPU process)  │  /v1/chat/...   │   (GPU process)  │
└─────────────────┘                 └─────────────────┘
```

**K8s target architecture**:
```
┌─────────────────────┐
│  Eval Pod (CPU)     │
│  - llmeval_suite    │      HTTP       ┌─────────────────────┐
│  - llmeval_framework│ ──────────────► │ Inference Pod (GPU) │
│  - DeepEval         │  /v1/chat/...   │ - inference_srv_py  │
│  - pytest           │                 │ - LiteRT-LM/vLLM    │
└─────────────────────┘                 └─────────────────────┘
```

**Constraints**:
- Eval pod must be able to reach inference service (K8s Service DNS)
- `HYBRID_AI_BACKEND_BASE_URL` must point to internal K8s service
- Eval results must be stored/exported (S3, PVC, or external)
- Multiple eval pods can target same inference service

**Acceptance criteria**:
1. Eval suite runs successfully on CPU-only pod (no GPU resource request)
2. Eval pod can reach inference service via K8s internal DNS
3. Eval results are accessible after pod completion (not lost)
4. Multiple concurrent eval runs do not conflict
5. GPU cost is incurred only by inference pods, not eval pods

**Cost model**:
| Workload | Resource | Cost driver |
|---|---|---|
| Inference serving | GPU (fractional) | Per-GPU-hour |
| Eval orchestration | CPU only | Per-CPU-hour (much cheaper) |
| Model loading | GPU memory | One-time per pod startup |
| Eval data storage | S3/PVC | Per-GB-month |

**Design impact**: Requires DD-K8S-08 (Eval Suite Deployment Architecture).

**Open questions**:
- [ ] Should eval pods be Jobs or Deployments?
- [ ] How are eval results collected (S3 upload, PVC, stdout)?
- [ ] Can eval pods scale independently of inference pods?
- [ ] How does this interact with UC_DEV_08 (eval execution workflow)?

---

## Identified Design Decisions (Draft)

These decisions are captured here during analysis. Once validated, they would normally be promoted to `docs/design-domain/`.

### Design Decision Summary

| ID | Title | Scope | Status |
|---|---|---|---|
| DD-K8S-01 | KServe Custom Runtime Adapter | How to adapt server.py for KServe | Open |
| DD-K8S-02 | Model Artifact Loading Strategy | S3 vs PVC for model distribution | Open |
| DD-K8S-03 | GPU Fractional Allocation Boundary | How to request/validate fractional GPU | Open |
| DD-K8S-04 | Conversation Session Handling | Session state in K8s | **Resolved** — already stateless |
| DD-K8S-05 | Container Image Build Strategy | How to produce OCI from Flox | Open — `flox containerize` preferred |
| DD-K8S-06 | Multi-Engine Abstraction Layer | LiteRT-LM + vLLM support | Open |
| DD-K8S-07 | Flox Environment to K8s Mapping | How manifest maps to K8s artifacts | Open |
| DD-K8S-08 | Eval Suite Deployment Architecture | CPU-only eval pods targeting GPU inference | Open |

---

### DD-K8S-01: KServe Custom Runtime Adapter Pattern

**Scope**: Define how `inference_srv_py` adapts to KServe's custom runtime model.

**Decision space**:

| Option | Pros | Cons |
|---|---|---|
| A. KServe `ServingRuntime` with custom Python image | Full control, preserves existing code | Must implement KServe protocol, more image maintenance |
| B. KServe InferenceService with transformer | Uses KServe predictor pattern | May require model server abstraction we don't use |
| C. Raw K8s Deployment behind Istio gateway | No KServe overhead | Loses KServe autoscaling, canary, model mgmt features |

**Leaning**: Option A — custom ServingRuntime preserves the existing backend service while gaining KServe's operational features.

**Open questions**:
- [ ] What KServe version is available in the cluster?
- [ ] Is KServe ModelMesh in use or standalone InferenceService?

---

### DD-K8S-02: Model Artifact Loading Strategy

**Scope**: Define how model artifacts are loaded into inference pods.

**Decision space**:

| Option | Pros | Cons |
|---|---|---|
| A. Model baked into container image | Simple deployment, no runtime download | Large images, slow builds, no dynamic model selection |
| B. S3 download at pod init (initContainer) | Separates model from code, enables versioning | Cold-start latency, requires S3 credentials |
| C. PVC with pre-populated model (ReadOnlyMany) | Shared across pods, instant mount | Requires PVC provisioning workflow, storage class dependency |
| D. S3 with local node cache (e.g., CSI driver) | Combines S3 flexibility with local cache speed | Infrastructure complexity, cache invalidation |

**Leaning**: Option B or C depending on cluster storage capabilities. Option B preferred for flexibility.

**Open questions**:
- [ ] Is ReadOnlyMany PVC supported in the cluster?
- [ ] What is acceptable cold-start latency?
- [ ] How large is the fine-tuned model artifact?

---

### DD-K8S-03: GPU Fractional Allocation Boundary

**Scope**: Define how fractional GPU resources are requested and validated.

**Decision space**:

| Option | Pros | Cons |
|---|---|---|
| A. RunAI fractional GPU scheduling | Managed by platform, transparent to app | Depends on RunAI configuration, may have overhead |
| B. MIG partitioning (static slices) | Hardware isolation, predictable VRAM | Requires A100/H100, admin setup, less flexible |
| C. Time-slicing (NVIDIA device plugin) | Works on any GPU, no hardware requirements | No memory isolation, noisy neighbor risk |

**Leaning**: Accept RunAI's allocation model (likely Option A) with explicit memory request documentation.

**Design constraint**: GPU validation in `gpu_validation.py` must work within container with allocated fraction, not assume full GPU.

**Open questions**:
- [ ] What fraction size is typical (e.g., 0.25, 0.5 GPU)?
- [ ] Does Gemma 4 E4B fit in fractional VRAM allocation?
- [ ] How do we detect/report VRAM pressure?

---

### DD-K8S-04: Conversation Session Handling

**Scope**: Confirm conversation state model is compatible with horizontally scalable K8s deployment.

**Current model**: **Already stateless** — `complete_chat()` creates an ephemeral `ConversationSession` per request, client provides full message history, session is closed after each response.

```python
# From backend.py - the endpoint is explicitly stateless
def complete_chat(self, messages: list[dict[str, str]]) -> dict[str, object]:
    """Stateless chat completion with full message context.

    This is an OpenAI-compatible endpoint where the client provides the full
    conversation history. The backend creates an ephemeral conversation,
    sends the combined context, and returns the result.
    """
    # ... creates ephemeral session, gets response, closes session
```

**K8s compatibility**: ✅ **No changes required**

The current stateless design is ideal for K8s horizontal scaling:
- No sticky sessions needed
- No session affinity required
- Any pod can handle any request
- Pod restarts don't lose state
- Autoscaling is straightforward

**Design validation**:
- [x] Client sends full message history per request ✅
- [x] Backend creates ephemeral session per request ✅
- [x] Session closed immediately after response ✅
- [x] No server-side conversation storage ✅

**Note**: The `ConversationSession` abstraction exists for LiteRT-LM engine interaction, but it's ephemeral (created and destroyed per HTTP request), not persistent across requests.

---

### DD-K8S-05: Container Image Build Strategy (Flox/Nix Native)

**Scope**: Define how to leverage Flox/Nix native container build capabilities for K8s deployment.

**Current state**: 
- Local dev uses Flox-composed Nix environments (`env/inference-litert-linux-gpu/`)
- Python deps pinned via `poetry.lock`
- GPU deps pinned via Nix (`vulkan-loader`, `nvidia_x11`)
- **Flox provides `flox containerize` to directly produce OCI images**

**Decision space**:

| Option | Pros | Cons |
|---|---|---|
| A. **`flox containerize`** | Simplest path, direct env→container, matches dev workflow exactly | Single layer (larger per-update), requires Flox in CI |
| B. **Nix `dockerTools.buildLayeredImage`** | Per-dependency layers, efficient caching, smaller updates | Requires Nix flake, more complex setup |
| C. **nix2container** (nix-community) | Streaming layers, best layer optimization | Learning curve, less documented |
| D. **Traditional Dockerfile** | Familiar to ops, works with any CI | Loses Nix reproducibility, drift risk |
| E. **Hybrid: Nix/Flox base + Dockerfile app layer** | Two build systems, but pragmatic bridge | Partial reproducibility |

**Leaning**: Option A (`flox containerize`) for initial deployment — simplest path with full reproducibility. Evaluate Option B if layer caching becomes important for CI/CD efficiency.

**`flox containerize` workflow**:
```bash
# From env/inference-litert-linux-gpu/
flox containerize --file hybrid-ai-inference.tar
# Or direct to Docker
flox containerize | docker load
```

**Advantages of `flox containerize`**:
1. Exact same environment as local dev (manifest.toml → container)
2. No Dockerfile or Nix expression required
3. All Nix-pinned GPU deps (vulkan-loader, nvidia_x11) included
4. Python venv with poetry.lock pins included
5. Hook scripts and profile settings preserved

**Considerations**:
- Produces monolithic single-layer image (all deps in one layer)
- Image size may be larger than traditional multi-stage Docker builds
- Each update rebuilds entire layer (vs layered approach caching unchanged deps)
- Requires Flox installed in CI/build environment

**KServe compatibility**:
- KServe expects specific health/ready probe paths — our server provides `/health`, `/ready` ✅
- KServe may inject sidecars — Flox-built images are standard OCI ✅
- NVIDIA device plugin in K8s handles GPU device mounting — container has userspace libs ✅

**Open questions**:
- [ ] Is Flox available in CI/build environment?
- [ ] What is acceptable image size? (Flox images may be larger)
- [ ] Does layer caching matter enough to warrant `dockerTools.buildLayeredImage`?
- [ ] What registry does the cluster use (Harbor, ECR, GCR, internal)?

---

### DD-K8S-06: Multi-Engine Abstraction Layer

**Scope**: Define how the backend service supports multiple inference engines (LiteRT-LM, vLLM) behind a unified contract.

**Current state**: `BackendService` uses `EngineRuntime` protocol with `runtime_factory` injection. Only `LiteRTEngineRuntime` is implemented.

**Decision space**:

| Option | Pros | Cons |
|---|---|---|
| A. Engine per pod (deployment-time selection) | Simple routing, engine-specific scaling | Separate deployments per engine, more resources |
| B. Engine per request (runtime routing) | Single deployment, dynamic selection | Complex routing, potential resource contention |
| C. Sidecar pattern (engine as separate container) | Clean separation, independent lifecycle | Inter-container latency, more complex pods |
| D. Separate services behind gateway | Full isolation, independent scaling | More infrastructure, routing complexity |

**Leaning**: Option A for initial deployment — simpler operations, clear resource boundaries. Option D as evolution target if multi-engine becomes high-frequency.

**Implementation impact**:
1. Add `VLLMEngineRuntime` implementing `EngineRuntime` protocol
2. Engine selection via environment variable (`HYBRID_AI_ENGINE=litert|vllm`)
3. Model loading strategy differs: LiteRT-LM uses local file, vLLM uses HuggingFace model ID
4. GPU memory management: vLLM uses PagedAttention, may interact differently with fractional GPU

**vLLM-specific considerations**:
- vLLM has its own OpenAI-compatible server — could wrap or embed?
- vLLM supports continuous batching — scaling characteristics differ
- vLLM tensor parallelism may conflict with fractional GPU model

**Open questions**:
- [ ] Should vLLM run as embedded engine or as separate server process?
- [ ] Does vLLM's OpenAI server satisfy our contract directly?
- [ ] How does vLLM model loading interact with S3/PVC strategy?
- [ ] What is the VRAM footprint difference between engines for same model?

---

### DD-K8S-07: Flox Environment to K8s Deployment Mapping

**Scope**: Define how Flox environment configuration maps to K8s deployment artifacts.

**Current Flox architecture**:
```
env/base (git, shell tools)
    └── env/python (Python 3.11, poetry, uv)
            └── env/inference-litert-linux-gpu (Vulkan, NVIDIA, LiteRT hooks)
```

**Direct path via `flox containerize`**:
```bash
cd env/inference-litert-linux-gpu
flox containerize --file hybrid-ai-inference.tar
# Or pipe directly to Docker/Podman
flox containerize | docker load
```

**Dependency sources preserved by `flox containerize`**:

| Source | Content | Preservation |
|---|---|---|
| `env/*/manifest.toml` | Nix packages (vulkan-loader, nvidia_x11, python311) | ✅ Direct — becomes container content |
| `[vars]` section | Environment variables | ✅ Direct — baked into image |
| `[hook]` section | Activation scripts | ✅ Direct — runs on container start |
| `[profile]` section | Shell profile scripts | ✅ Direct — available in container |
| `src/inference_srv_py/poetry.lock` | Python packages | ⚠️ Requires poetry install in hook |

**K8s deployment mapping**:

| Flox manifest element | K8s equivalent |
|---|---|
| `[install]` packages | Container image contents (via `flox containerize`) |
| `[vars]` | Can be overridden via Pod env / ConfigMap |
| `[hook].on-activate` | Container entrypoint |
| `[profile].common` | Available in exec shells |
| `HYBRID_AI_HOST` | Always `0.0.0.0` in container |
| `HYBRID_AI_PORT` | Container port (default 8080) |
| `HYBRID_AI_MODEL_PATH` | Volume mount path |

**Python environment consideration**:

The current Flox setup uses poetry to create a venv in `.flox/cache/python/`. For containerization:
- Option A: Run `poetry install` in `[hook].on-activate` (slower startup)
- Option B: Pre-install in container build step (faster startup)
- Option C: Use poetry2nix to make Python deps Nix-native (best reproducibility)

**Leaning**: Option B for production (pre-install), Option A acceptable for dev images.

**Open questions**:
- [ ] Is Flox available in CI, or must container builds happen locally?
- [ ] Does cluster allow arbitrary images or require approved base list?
- [ ] What is acceptable image size? (Flox images may be larger than minimal Docker images)
- [ ] Should we publish GPU base image separately for caching?

---

### DD-K8S-08: Eval Suite Deployment Architecture

**Scope**: Define how the eval suite runs on CPU-only pods while targeting GPU-based inference services.

**Current local architecture**:
- `llmeval_suite` and `llmeval_framework` are Python modules in `src/`
- Run via pytest with DeepEval integration
- Target inference via `HYBRID_AI_BACKEND_BASE_URL` environment variable
- No GPU required for eval execution (only for inference)

**K8s deployment options**:

| Option | Pros | Cons |
|---|---|---|
| A. **K8s Job per eval run** | Clean lifecycle, easy to track completion | Job overhead per run, result collection needed |
| B. **Long-running eval Deployment** | Always ready, lower latency | Idle resource cost, state management |
| C. **Argo Workflows / Tekton** | Full workflow orchestration, parallel eval | Infrastructure complexity |
| D. **CI/CD triggered pods** | Integrates with existing CI | Depends on CI having K8s access |

**Leaning**: Option A (K8s Job) for initial implementation — clean lifecycle, results via S3 or stdout.

**Eval container image**:
- Derive from `env/llmeval-framework` via `flox containerize`
- No GPU dependencies (no vulkan-loader, nvidia_x11)
- Include: Python, pytest, DeepEval, llmeval_suite, llmeval_framework
- Significantly smaller than inference image

**K8s manifest sketch**:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: eval-run-sd005
spec:
  template:
    spec:
      containers:
      - name: eval
        image: hybrid-ai-eval:latest
        env:
        - name: HYBRID_AI_BACKEND_BASE_URL
          value: "http://inference-svc:8080"  # K8s internal DNS
        command: ["pytest", "-m", "sd005", "--tb=short"]
        resources:
          requests:
            cpu: "1"
            memory: "4Gi"
          limits:
            cpu: "2"
            memory: "8Gi"
          # NO GPU REQUESTED
      restartPolicy: Never
  backoffLimit: 2
```

**Service discovery**:
- Inference service exposed as K8s Service (e.g., `inference-svc`)
- Eval pods use internal DNS: `http://inference-svc:8080`
- No ingress needed for eval-to-inference traffic (cluster internal)

**Result collection options**:
| Option | Mechanism | Complexity |
|---|---|---|
| A. S3 upload in job | Add boto3, upload results | Medium |
| B. PVC shared storage | Mount results PVC | Low |
| C. Stdout + log aggregation | Capture via Loki/ELK | Low |
| D. DeepEval cloud dashboard | Native DeepEval feature | External dependency |

**Leaning**: Option B (PVC) for simplicity, Option A (S3) if results need external access.

**Parallel eval execution**:
- Multiple eval Jobs can run simultaneously
- Each targets same inference service
- Inference service must handle concurrent requests (already does — ThreadingHTTPServer)
- Consider load impact on inference service during heavy eval

**Open questions**:
- [ ] What eval result format is required (JUnit XML, JSON, custom)?
- [ ] Should eval jobs have resource quotas per developer/namespace?
- [ ] How do we track eval history across runs?
- [ ] Can eval pods access model registry to verify model version?

---

## Exploration Areas

### Area 1: KServe Custom Runtime Implementation

**Goal**: Understand the KServe custom runtime protocol and map existing server.py to it.

**Actions**:
- [ ] Review KServe v2 inference protocol
- [ ] Map `/v1/chat/completions` to KServe endpoint conventions
- [ ] Determine if KServe health/ready probes differ from current implementation
- [ ] Evaluate KServe autoscaling behavior for LLM workloads

### Area 2: RunAI and Canopy CLI Workflow

**Goal**: Understand the developer workflow for deploying and managing inference workloads.

**Actions**:
- [ ] Document Canopy CLI commands for deployment, logs, port-forward
- [ ] Understand RunAI project/namespace model
- [ ] Determine quota and resource limit configuration
- [ ] Map to existing use case UC_DEV_01 (isolated dev environment)

### Area 3: Model Registry Integration

**Goal**: Understand the custom model catalogue API and integration requirements.

**Actions**:
- [ ] Document registry API for model listing, download, versioning
- [ ] Determine authentication model (service account, OIDC, etc.)
- [ ] Map to BR-K8S-02 acceptance criteria
- [ ] Evaluate how model version is exposed in inference responses

### Area 4: Storage Boundary Portability

**Goal**: Understand how current MinIO/DVC storage patterns map to K8s storage.

**Actions**:
- [ ] Review `docs/design-domain/18-dd-llmevals-minio-s3-access-boundary.md`
- [ ] Determine if S3-compatible API is exposed in target cluster
- [ ] Evaluate PVC StorageClass options for model artifacts
- [ ] Consider cache tiering for frequently-accessed models

### Area 5: Multi-Engine Runtime Integration

**Goal**: Understand how to support both LiteRT-LM and vLLM engines in K8s deployment.

**Actions**:
- [ ] Evaluate vLLM's native OpenAI-compatible server vs embedding in our backend
- [ ] Compare VRAM footprint: LiteRT-LM vs vLLM for Gemma 4 E4B
- [ ] Test vLLM with fractional GPU (RunAI or MIG)
- [ ] Implement `VLLMEngineRuntime` class satisfying `EngineRuntime` protocol
- [ ] Determine model artifact format requirements for each engine
- [ ] Evaluate continuous batching benefits for multi-tenant scenario

**Key vLLM integration questions**:
- vLLM already provides OpenAI-compatible `/v1/chat/completions` — wrap or replace our server?
- vLLM's PagedAttention may improve multi-tenant GPU utilization significantly
- vLLM tensor parallelism: compatible with fractional GPU or requires full GPU?

### Area 6: Flox Containerization Validation

**Goal**: Validate `flox containerize` produces working container images that preserve Flox environment guarantees.

**Actions**:
- [ ] Run `flox containerize` from `env/inference-litert-linux-gpu`
- [ ] Verify GPU closure (Vulkan, NVIDIA libs) present in container
- [ ] Validate GPU initialization inside container
- [ ] Compare container behavior to local Flox environment
- [ ] Document image size and build time
- [ ] Evaluate `dockerTools.buildLayeredImage` if layer caching needed
- [ ] Determine CI requirements for Flox-based builds

**Key considerations**:
- `flox containerize` produces single-layer image (all deps in one layer)
- Nix closure includes exact versions from manifest.toml
- Container entrypoint should run activation hooks
- Image size may be larger than traditional Docker builds — measure and evaluate

**Validation criteria**:
- `gpu_validation.py` passes in container
- Inference results identical to local Flox environment
- No LD_LIBRARY_PATH workarounds needed

**Reference**: [Flox blog: Nix and Containers](https://flox.dev/blog/nix-and-containers-why-not-both)

### Area 7: Eval Suite CPU Deployment

**Goal**: Validate eval suite runs on CPU-only pods while targeting GPU inference service.

**Actions**:
- [ ] Build eval container from `env/llmeval-framework` via `flox containerize`
- [ ] Verify no GPU dependencies in eval image (should be smaller than inference image)
- [ ] Test K8s Job execution targeting inference service
- [ ] Validate result collection (S3, PVC, or stdout)
- [ ] Measure eval execution time and resource usage
- [ ] Test parallel eval jobs targeting same inference service

**Key validation criteria**:
- Eval Job completes successfully with no GPU resource request
- Eval results match local execution results
- Inference service handles concurrent eval requests
- Results are accessible after Job completion

**Cost validation**:
- Document CPU-hour cost vs GPU-hour cost
- Estimate savings from separating eval from inference
- Consider eval pod scaling vs inference pod scaling

---

## Next Steps

See **De-Risking Roadmap** below for sequenced approach.

---

## De-Risking Roadmap

### Rationale

This roadmap prioritizes **de-risking** over feature completeness. Each phase validates a critical assumption before committing resources to dependent work. Spikes are time-boxed experiments that produce go/no-go decisions.

### Risk Assessment

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| **R1: GPU init fails in container** | Blocker — cannot deploy | Medium | Spike S1 validates LiteRT-LM + Vulkan in container |
| **R2: Model load latency unacceptable** | Blocks scaling — cold-start too slow | Medium | Spike S2 measures S3/PVC options |
| **R3: Fractional GPU insufficient for model** | Redesign — need full GPU or smaller model | Medium | Spike S1 includes VRAM measurement |
| **R4: vLLM incompatible with fractional GPU** | Limits engine choice | Low-Medium | Spike S3 tests vLLM allocation |
| **R5: KServe protocol mismatch** | Rework — endpoint adaptation needed | Low | Spike S4 maps protocol requirements |
| **R6: Multi-version routing complex** | Delay — but UC_DEV_09 pattern proven | Low | Leverage existing env-var pattern |
| **R7: Flox unavailable in CI** | Must use Dockerfile fallback, lose some reproducibility | Medium | S1 tests both paths, document CI requirements |
| **R8: Flox container image too large** | Slower pulls, more storage cost | Low-Medium | Measure in S1, evaluate `buildLayeredImage` if needed |
| **R9: Eval→Inference network latency** | Slower evals or timeouts | Low | S7 validates HTTP round-trip in cluster; inference is already slow (LLM) |

### Phase 1: Foundation Validation (Week 1-2)

**Goal**: Answer "can we run at all?" before investing in architecture.

#### Spike S1: GPU Boundary & Container Build Validation [CRITICAL PATH]

**Time-box**: 4 days  
**Owner**: TBD  
**Blocks**: All subsequent work

**Objective**: Validate LiteRT-LM engine initialization in a container with fractional GPU allocation, using `flox containerize` as primary approach.

**Deliverables**:

1. **`flox containerize` image** (primary path):
   - Build from `env/inference-litert-linux-gpu`
   - Command: `flox containerize | docker load`
   - Verify GPU closure (vulkan-loader, nvidia_x11) present
   - Document image size and build time

2. **Dockerfile image** (fallback path):
   - Base: `nvidia/cuda:12.x-runtime-ubuntu22.04`
   - Install system deps matching Nix manifest
   - Poetry install from lockfile
   - Document version drift risks

3. **Validation for both images**:
   - Test script that initializes engine and runs single inference
   - VRAM consumption measurement for Gemma 4 E4B
   - Parity check: compare inference output to local Flox environment

4. **Documentation**:
   - Required GPU fraction (0.25, 0.5, 1.0)
   - Build instructions for both paths
   - Recommendation for primary approach

**Success criteria**:
- [ ] `gpu_validation.py` passes inside both container types
- [ ] LiteRT-LM engine initializes without host linker workarounds
- [ ] Single inference request completes successfully
- [ ] VRAM usage documented and within fractional allocation
- [ ] `flox containerize` image produces same inference results as local Flox env
- [ ] Build process documented and reproducible

**Failure paths**:
- If `flox containerize` fails but Dockerfile works → proceed with Dockerfile, document reproducibility tradeoff
- If both fail on GPU → escalate to vLLM or full GPU allocation
- If Flox unavailable in CI → Dockerfile becomes primary, add validation gates

#### Spike S2: Model Loading Latency

**Time-box**: 2 days  
**Owner**: TBD  
**Depends on**: S1 (need working container)

**Objective**: Measure cold-start latency for model loading from S3 vs PVC.

**Deliverables**:
1. S3 download timing (initContainer pattern)
2. PVC mount timing (ReadOnlyMany if available)
3. Local SSD cache timing (node-local path)
4. Recommendation for model loading strategy

**Success criteria**:
- [ ] Cold-start latency documented for each option
- [ ] Acceptable latency threshold defined (e.g., <60s)
- [ ] Recommended strategy selected (DD-K8S-02 resolved)

### Phase 2: Engine & Protocol Validation (Week 2-3)

**Goal**: Validate multi-engine support and KServe integration.

#### Spike S3: vLLM Fractional GPU Compatibility

**Time-box**: 3 days  
**Owner**: TBD  
**Depends on**: S1

**Objective**: Validate vLLM with fractional GPU and compare to LiteRT-LM.

**Deliverables**:
1. vLLM container with Gemma 4 E4B (HuggingFace format)
2. VRAM consumption comparison: vLLM vs LiteRT-LM
3. Throughput comparison: requests/sec, latency percentiles
4. `VLLMEngineRuntime` prototype (if viable)

**Success criteria**:
- [ ] vLLM initializes with fractional GPU
- [ ] Performance characteristics documented
- [ ] Engine selection recommendation (DD-K8S-06 informed)

**Decision point**: If vLLM performs significantly better with fractional GPU, consider it as primary engine for K8s deployment while keeping LiteRT-LM for on-device path.

#### Spike S4: KServe Protocol Mapping

**Time-box**: 2 days  
**Owner**: TBD  
**Depends on**: S1

**Objective**: Map existing server.py to KServe custom runtime requirements.

**Deliverables**:
1. KServe v2 protocol gap analysis
2. Endpoint mapping: `/v1/chat/completions` → KServe conventions
3. Health/ready probe compatibility check
4. Minimal ServingRuntime manifest

**Success criteria**:
- [ ] Protocol gaps documented
- [ ] Adaptation strategy defined (wrapper vs native)
- [ ] Sample manifest deployed to dev cluster

### Phase 3: Multi-Version & Integration (Week 3-4)

**Goal**: Validate multi-version deployment and developer workflow.

#### Spike S5: Multi-Version Model Deployment

**Time-box**: 2 days  
**Owner**: TBD  
**Depends on**: S2 (model loading), S4 (KServe)

**Objective**: Deploy two model versions side-by-side and validate UC_DEV_09 workflow in K8s.

**Deliverables**:
1. Two-version deployment manifest (baseline + fine-tuned)
2. Routing strategy (separate services or labels)
3. Eval suite targeting both versions
4. Model registry integration spec

**Success criteria**:
- [ ] Two versions running simultaneously
- [ ] Eval suite can target each version via URL
- [ ] Model version traceable in inference response

#### Spike S6: Developer Workflow Validation

**Time-box**: 2 days  
**Owner**: TBD  
**Depends on**: S4 (KServe), S5 (multi-version)

**Objective**: Validate Canopy CLI workflow for common developer tasks.

**Deliverables**:
1. Deployment workflow documentation
2. Log access and debugging workflow
3. Port-forward or ingress configuration for local clients
4. Comparison to UC_DEV_01/UC_DEV_04 local workflow

**Success criteria**:
- [ ] Developer can deploy custom image
- [ ] Developer can access logs and debug
- [ ] Local eval harness can target K8s deployment
- [ ] Workflow documented in runbook

#### Spike S7: Eval Suite CPU Deployment Validation

**Time-box**: 2 days  
**Owner**: TBD  
**Depends on**: S1 (container build), S4 (KServe)

**Objective**: Validate eval suite runs on CPU-only pods targeting GPU inference service, confirming GPU cost savings.

**Deliverables**:
1. Eval container image from `env/llmeval-framework` (no GPU deps)
2. K8s Job manifest for eval execution
3. Service discovery validation (eval pod → inference service)
4. Result collection mechanism (S3, PVC, or stdout)
5. Cost comparison: CPU eval vs GPU eval

**Success criteria**:
- [ ] Eval container image built successfully (smaller than inference image)
- [ ] Eval Job runs with no GPU resource request
- [ ] Eval Job successfully calls inference service via K8s DNS
- [ ] Eval results are accessible after Job completion
- [ ] Parallel eval Jobs do not conflict
- [ ] GPU cost incurred only by inference pods

**Validation tests**:
- Run SD-005 eval scenario from CPU pod
- Compare results to local execution
- Measure CPU/memory usage during eval
- Test multiple concurrent eval Jobs

### Phase 4: Production Readiness (Week 4-5)

**Goal**: Harden for multi-tenant production use.

**Tasks** (not spiked — dependent on earlier phases):

1. **Autoscaling configuration**: Define scaling policy based on S3 findings (cold-start tolerance)
2. **Resource quotas**: Implement per-tenant limits based on S1 VRAM findings
3. **Observability**: Metrics, logging, tracing integration
4. **CI/CD integration**: Image build and deployment automation
5. **Documentation**: Runbook updates, developer onboarding guide

### Decision Gates

| Gate | Criteria | Blocks |
|---|---|---|
| **G1: GPU Viable** | S1 succeeds with acceptable VRAM | All subsequent phases |
| **G1b: Container Build Selected** | S1 determines Nix2Container vs Dockerfile | CI/CD integration, image strategy |
| **G2: Cold-Start Acceptable** | S2 latency < threshold | Production deployment |
| **G3: Engine Selected** | S3 informs LiteRT-LM vs vLLM choice | DD-K8S-06 resolution |
| **G4: Protocol Compatible** | S4 gaps are tractable | KServe deployment |
| **G5: Multi-Version Works** | S5 validates UC_DEV_09 pattern | A/B testing capability |
| **G6: Eval CPU Separation Works** | S7 validates eval→inference HTTP calls | Cost-optimized eval deployment |

### Roadmap Visualization

```
Week 1          Week 2          Week 3          Week 4          Week 5
│               │               │               │               │
├─ S1: GPU+Nix ─┤               │               │               │
│   [CRITICAL]  ├─ S2: Model ───┤               │               │
│   Container   │   Loading     ├─ S5: Multi ───┤               │
│               ├─ S3: vLLM ────┤   Version     │               │
│               │               ├─ S6: Dev ─────┤               │
│               ├─ S4: KServe ──┤   Workflow    ├─ S7: Eval ────┤
│               │               │               │   CPU Deploy  │
│               │               │               ├─ Phase 4: ────┤
│               │               │               │   Production  │
▼               ▼               ▼               ▼               ▼
G1: GPU+Build   G2: Cold-Start  G4: Protocol   G5: Multi-Ver   Deploy
G1b: Nix/Docker G3: Engine                     G6: Eval CPU    
```

### Critical Path

**S1 → (S2 | S3 | S4) → S5 → (S6 | S7) → Production**

S1 (GPU Boundary + Container Build) is the critical path blocker. It validates:
1. LiteRT-LM GPU initialization in container
2. Nix2Container viability vs Dockerfile fallback
3. Flox/Nix environment parity in container

S7 (Eval CPU Deployment) runs in parallel with S6, depends on S1+S4 completion:
- Requires working inference container (S1)
- Requires KServe HTTP endpoint (S4)

If S1 fails, all other work is wasted. Execute S1 first and validate before committing to remaining spikes.

**Flox/Nix bridge decision**: S1 determines whether we maintain full reproducibility (Nix2Container) or accept some drift (Dockerfile). This affects CI/CD integration and long-term maintenance.

---

## Companion Documents

- [[17-multi-tenant-devenv-use-cases]] — Validated developer use cases
- [[07-dd-backend-conversation-contract]] — Backend contract semantics
- [[13-dd-linux-backend-runtime-adapter]] — Current Linux runtime adapter
- [[15-common-development-environment-for-llm-delivery]] — Current shared environment model
- [[devenv_portable_workflow]] — Flox/Nix environment architecture and design decisions
- `env/inference-litert-linux-gpu/manifest.toml` — GPU environment Nix packages
- `env/python/manifest.toml` — Python environment composition

---

## Interactive Analysis Questions

The following questions structure the interactive analysis. Answers will inform BR and DD refinements.

### Environment Questions

1. **KServe version and configuration**: Is this KServe 0.11+? Is ModelMesh enabled or standalone InferenceService?

2. **RunAI fractional GPU model**: How are fractional GPUs allocated — MIG, time-slicing, or RunAI's virtual GPU?

3. **Storage primitives**: What StorageClasses are available? Is there an S3-compatible endpoint (internal MinIO, cloud S3, etc.)?

4. **Model registry**: What is the API surface? Is it a custom internal registry or based on MLflow/Seldon/other?

5. **Container registry**: Where do custom runtime images get pushed? What base images are pre-approved?

### Workflow Questions

6. **Canopy CLI scope**: What operations does Canopy CLI support — deployment, logs, exec, port-forward?

7. **Namespace isolation**: Is each developer assigned a namespace, or is there a shared namespace with pod-level isolation?

8. **CI/CD integration**: How are images built and deployed — GitOps, Jenkins, Tekton, manual push?

### Build & Reproducibility Questions

9. **Nix in CI**: Is Nix available in the CI/CD environment? Can we use Nix2Container builds?

10. **Base image policy**: Does the cluster require pre-approved base images, or can we use custom Nix-built images?

11. **Reproducibility requirements**: How important is exact reproducibility vs ops-familiar Dockerfile approach?

### Scale Questions

12. **Expected concurrency**: How many concurrent inference requests should a single pod handle?

13. **Cold-start tolerance**: What is acceptable pod startup time (including model load)?

14. **Autoscaling requirements**: Should pods scale to zero, or maintain a warm minimum?

---

## Analysis Log

| Date | Topic | Finding |
|---|---|---|
| 2026-08-20 | Initial analysis | Created runbook, identified 5 BR and 5 DD candidates |
| 2026-08-20 | Retrofit | Corrected implementation summary (OpenAI contract, EngineRuntime abstraction exists). Added BR-K8S-06 (multi-engine support) and DD-K8S-06 (multi-engine abstraction layer) for LiteRT-LM + vLLM requirement. |
| 2026-08-20 | Multi-version | Expanded BR-K8S-02 to reference UC_DEV_09 multi-version pattern. Current implementation already supports multi-version via env vars — K8s adaptation is routing/registry integration. |
| 2026-08-20 | De-risking roadmap | Added phased roadmap with 6 time-boxed spikes. S1 (GPU boundary) is critical path blocker — validate first before committing to architecture. |
| 2026-08-20 | Flox/Nix bridge | Added BR-K8S-07 (environment reproducibility), DD-K8S-07 (Flox/Nix to K8s mapping). S1 tests container builds. Added R7/R8 risks for container parity. |
| 2026-08-20 | Session model | Corrected DD-K8S-04: Implementation is **already stateless** (`complete_chat` creates ephemeral session per request). No K8s changes required for session handling — ideal for horizontal scaling. |
| 2026-08-20 | Flox containerize | Corrected DD-K8S-05: **`flox containerize` directly produces OCI images** (not a bridge needed). Per [Flox blog](https://flox.dev/blog/nix-and-containers-why-not-both), this is the simplest path. Updated S1 to use `flox containerize` as primary approach. |
| 2026-08-20 | Architecture coverage | Added explicit DD references to Flox/Nix Architecture section (DD-K8S-05, DD-K8S-07, BR-K8S-07). Added summary tables for BR and DD sections. |
| 2026-08-20 | Eval CPU separation | Added BR-K8S-08 (eval suite CPU separation) and DD-K8S-08 (eval suite deployment architecture). Eval runs as K8s Job on CPU-only pods, calls inference service via HTTP. Added S7 spike and G6 decision gate. Cost optimization: eval pods require 0 GPU allocation. |
| | | |

