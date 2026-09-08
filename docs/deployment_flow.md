**Deploying a Gemma 4 Small Language Model (SLM) with vLLM on OpenShift, automated via GitHub Actions**

Gemma 4 (released 2026) is Google’s efficient open-weight multimodal family. “SLM” typically refers to the smaller variants such as **Gemma 4 E2B / E4B** or the **12B** model. These are well-suited for single-GPU or modest multi-GPU serving. Official model IDs on Hugging Face include `google/gemma-4-E4B-it`, `google/gemma-4-12B-it`, etc. vLLM has day-0 / early support for Gemma 4.

There are two main deployment styles on OpenShift:

1. **Plain OpenShift** (Deployment + Service + Route) — simpler, works without OpenShift AI.
2. **OpenShift AI + KServe** (ServingRuntime + InferenceService) — recommended for production; provides better lifecycle management, autoscaling options, and integration with the OpenShift AI dashboard.

The process below covers both, with a strong focus on **Kubernetes/OpenShift namespaces (Projects)** and full GitHub Actions automation.

### 1. Prerequisites

- OpenShift 4.14+ cluster with GPU nodes.
- NVIDIA GPU Operator installed and working (`nvidia.com/gpu` resource available).
- (Recommended) OpenShift AI operator installed if you want the KServe path.
- Hugging Face token (store as a Kubernetes Secret; some Gemma variants may require acceptance of the license).
- `oc` / `kubectl` access.
- A GitHub repository containing your manifests.
- An OpenShift service account (or kubeconfig) with rights to create resources in the target namespace, stored as a GitHub secret.

### 2. Role of the Kubernetes Namespace (OpenShift Project)

A **Namespace** (called a **Project** in OpenShift) is a virtual cluster that provides:

- **Isolation** — Resources (pods, services, secrets, PVCs, routes) belonging to the Gemma serving stack cannot accidentally interfere with other teams or workloads.
- **RBAC boundary** — You attach ServiceAccounts, Roles, and RoleBindings only inside this namespace. The GitHub Actions service account only needs permissions here.
- **Resource quotas & limit ranges** — Prevent a misconfigured vLLM pod from consuming the entire cluster’s GPUs or memory.
- **Network policies** — Restrict which other namespaces can call the model endpoint.
- **Multi-tenancy** — Different teams can run their own Gemma / vLLM instances in separate projects without colliding on names (e.g., `Service` named `vllm`).
- **OpenShift extras** — Projects add self-service features, project-level admins, and tighter integration with OpenShift AI “Data Science Projects”.

**Best practice**: Always create a dedicated project for the model serving workload, e.g. `gemma4-slm` or `vllm-gemma4`. Never deploy into `default` or shared system namespaces.

```bash
oc new-project gemma4-slm
# or
oc create namespace gemma4-slm
```

All subsequent resources must include `namespace: gemma4-slm` (or be applied with `-n gemma4-slm`).

### 3. Step-by-Step Deployment Process

#### Step 1: Create the Namespace / Project
```bash
oc new-project gemma4-slm
```
**Conceptual role**: Establishes the isolation boundary described above. All later objects live inside it.

#### Step 2: Create a ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vllm-sa
  namespace: gemma4-slm
```
**Why**: Pods run as this identity. You can later bind it to Security Context Constraints (SCCs) such as `anyuid` or a custom SCC if the vLLM image needs specific privileges (common issue with random UIDs on OpenShift).

#### Step 3: Create Hugging Face Token Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hf-token-secret
  namespace: gemma4-slm
type: Opaque
stringData:
  token: "hf_xxxxxxxxxxxxxxxx"   # never commit this
```
**Why**: vLLM needs the token to download gated or rate-limited models from Hugging Face. The token is injected as an environment variable.

#### Step 4: Create a PersistentVolumeClaim (model cache)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vllm-models-pvc
  namespace: gemma4-slm
spec:
  accessModes:
    - ReadWriteOnce          # or ReadWriteMany if you have RWX storage
  resources:
    requests:
      storage: 50Gi          # adjust for the chosen Gemma 4 variant
  storageClassName: <your-storage-class>   # e.g. ocs-storagecluster-ceph-rbd
```
**Why**: Model weights are large. Caching them on a PVC avoids re-downloading on every pod restart and speeds up cold starts.

#### Step 5A: Plain Deployment style (works on any OpenShift)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-gemma4
  namespace: gemma4-slm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vllm-gemma4
  template:
    metadata:
      labels:
        app: vllm-gemma4
    spec:
      serviceAccountName: vllm-sa
      containers:
      - name: vllm
        image: vllm/vllm-openai:latest          # or a Gemma-4-optimized tag / Red Hat AI Inference Server image
        args:
          - "--model=google/gemma-4-E4B-it"     # or google/gemma-4-12B-it
          - "--host=0.0.0.0"
          - "--port=8000"
          - "--max-model-len=8192"              # tune to your GPU memory
          - "--gpu-memory-utilization=0.90"
          - "--dtype=auto"
          - "--trust-remote-code"
        env:
        - name: HUGGING_FACE_HUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: hf-token-secret
              key: token
        ports:
        - containerPort: 8000
        resources:
          limits:
            nvidia.com/gpu: "1"
            memory: 24Gi
            cpu: "8"
          requests:
            nvidia.com/gpu: "1"
            memory: 16Gi
            cpu: "4"
        volumeMounts:
        - name: model-storage
          mountPath: /root/.cache/huggingface
        - name: dshm
          mountPath: /dev/shm
      volumes:
      - name: model-storage
        persistentVolumeClaim:
          claimName: vllm-models-pvc
      - name: dshm
        emptyDir:
          medium: Memory
          sizeLimit: 2Gi
      # Optional: nodeSelector / tolerations for GPU nodes
```

Then create a Service and Route:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vllm-gemma4
  namespace: gemma4-slm
spec:
  selector:
    app: vllm-gemma4
  ports:
  - port: 80
    targetPort: 8000
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: vllm-gemma4
  namespace: gemma4-slm
spec:
  to:
    kind: Service
    name: vllm-gemma4
  port:
    targetPort: 8000
  tls:
    termination: edge
```

#### Step 5B: OpenShift AI / KServe style (preferred for production)

**ServingRuntime** (defines the vLLM engine):

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  name: vllm-runtime
  namespace: gemma4-slm
spec:
  supportedModelFormats:
  - name: vllm
    version: "1"
    autoSelect: true
  multiModel: false
  containers:
  - name: kserve-container
    image: quay.io/modh/vllm:latest          # or Red Hat AI Inference Server image
    command: ["python", "-m", "vllm.entrypoints.openai.api_server"]
    args:
    - "--model=/mnt/models"
    - "--port=8080"
    - "--max-model-len=8192"
    - "--gpu-memory-utilization=0.90"
    resources:
      limits:
        nvidia.com/gpu: "1"
    ports:
    - containerPort: 8080
```

**InferenceService**:

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: gemma4-slm
  namespace: gemma4-slm
  annotations:
    serving.kserve.io/deploymentMode: RawDeployment   # or Serverless
spec:
  predictor:
    model:
      modelFormat:
        name: vllm
      runtime: vllm-runtime
      storageUri: pvc://vllm-models-pvc/   # or oci://... ModelCar, or hf://...
      # Alternative: let vLLM download directly with HF token via env
```

KServe creates the underlying Deployment, Service, and (optionally) Route for you.

### 4. Automating Everything with GitHub Actions

Create `.github/workflows/deploy-gemma4.yml`:

```yaml
name: Deploy Gemma 4 SLM with vLLM to OpenShift

on:
  push:
    branches: [main]
    paths:
      - 'manifests/**'
      - '.github/workflows/deploy-gemma4.yml'
  workflow_dispatch:          # manual trigger

env:
  OPENSHIFT_SERVER: ${{ secrets.OPENSHIFT_SERVER }}
  OPENSHIFT_TOKEN: ${{ secrets.OPENSHIFT_TOKEN }}
  PROJECT: gemma4-slm

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install oc CLI
        uses: redhat-actions/openshift-tools-installer@v1
        with:
          oc: latest

      - name: Log in to OpenShift
        run: |
          oc login --token=${{ secrets.OPENSHIFT_TOKEN }} --server=${{ secrets.OPENSHIFT_SERVER }}

      - name: Create / switch to project
        run: |
          oc new-project $PROJECT || oc project $PROJECT

      - name: Create HF token secret (if not exists)
        run: |
          oc create secret generic hf-token-secret \
            --from-literal=token=${{ secrets.HF_TOKEN }} \
            -n $PROJECT --dry-run=client -o yaml | oc apply -f -

      - name: Apply manifests
        run: |
          oc apply -f manifests/ -n $PROJECT
          # or kustomize: oc apply -k overlays/prod

      - name: Wait for deployment to be ready
        run: |
          oc rollout status deployment/vllm-gemma4 -n $PROJECT --timeout=600s
          # or for KServe:
          # oc wait --for=condition=Ready inferenceservice/gemma4-slm -n $PROJECT --timeout=600s

      - name: Smoke test
        run: |
          ROUTE=$(oc get route vllm-gemma4 -n $PROJECT -o jsonpath='{.spec.host}')
          curl -s https://$ROUTE/v1/models | jq .
          # or a simple chat completion test
```

**Required GitHub Secrets**:
- `OPENSHIFT_SERVER`
- `OPENSHIFT_TOKEN` (service account token with edit rights in the project)
- `HF_TOKEN`

### 5. Verification & Operations

```bash
# Check pods
oc get pods -n gemma4-slm

# Check logs
oc logs -f deploy/vllm-gemma4 -n gemma4-slm

# Get the endpoint
oc get route -n gemma4-slm

# Test OpenAI-compatible API
curl https://<route>/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "google/gemma-4-E4B-it", "messages": [{"role": "user", "content": "Hello!"}]}'
```

### Key Conceptual Takeaways

| Component          | Purpose                                      | Why it matters |
|--------------------|----------------------------------------------|----------------|
| **Namespace/Project** | Isolation, RBAC, quotas, multi-tenancy     | Core security & operational boundary |
| ServiceAccount     | Pod identity                                 | Controls SCC and permissions |
| Secret             | Sensitive credentials                        | Never bake tokens into images |
| PVC                | Persistent model cache                       | Fast restarts, cost control |
| Deployment / InferenceService | Desired state of the vLLM process     | Declarative, self-healing |
| Service + Route    | Stable network endpoint                      | Internal & external access |
| GitHub Actions     | CI/CD automation                             | Reproducible, auditable deployments |

This process gives you a production-ready, fully automated pipeline for serving a Gemma 4 SLM with vLLM on OpenShift while keeping strong isolation via a dedicated namespace. Start with the plain Deployment approach for simplicity, then migrate to OpenShift AI + KServe when you need advanced serving features.
