# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A step-by-step configuration guide (and companion manifests) for setting up Red Hat OpenShift AI (RHOAI) to deploy AI models using vLLM ServingRuntime for KServe on OpenShift. There is no build system, test suite, or application code — the deliverables are YAML manifests and documentation.

## Applying Manifests

All commands use the OpenShift CLI (`oc`). Apply manifests in numbered order:

```bash
# Step 1 — GPU support (NFD + NVIDIA GPU Operator)
oc apply -f manifests/01/nfd-operator.yaml
oc get pods -n openshift-nfd -w          # wait until ready
oc apply -f manifests/01/nfd-instance.yaml
oc apply -f manifests/01/nvidia-gpu-operator.yaml

# Retrieve and apply ClusterPolicy (generated into scratch/, not committed)
oc get csv -n nvidia-gpu-operator \
  -l operators.coreos.com/gpu-operator-certified.nvidia-gpu-operator \
  -ojsonpath='{.items[0].metadata.annotations.alm-examples}' | \
  jq '.[0]' > scratch/nvidia-gpu-clusterpolicy.json
oc apply -f scratch/nvidia-gpu-clusterpolicy.json

# Step 2 — KServe dependencies (Service Mesh, Serverless, Authorino, Cert-Manager, JobSet, Connectivity Link, LeaderWorkerSet)
oc apply -f manifests/02/servicemesh-subscription.yaml
oc apply -f manifests/02/serverless-operator.yaml
oc apply -f manifests/02/authorino-subscription.yaml
oc apply -f manifests/02/cert-manager-operator.yaml
oc apply -f manifests/02/jobset-operator-subscription.yaml
oc apply -f manifests/02/jobset-operator.yaml
oc apply -f manifests/02/connectivity-link-operator.yaml
oc patch console.operator.openshift.io cluster --type=json \
  -p '[{"op":"add","path":"/spec/plugins/-","value":"kuadrant-console-plugin"}]'
oc apply -f manifests/02/leader-worker-set-subscription.yaml
oc apply -f manifests/02/leader-worker-set-operator.yaml

# Step 3 — RHOAI Operator + DataScienceCluster
oc apply -f manifests/03/rhoai-operator.yaml
oc apply -f manifests/03/rhoai-operator-dsc.yaml

# Step 4 — MySQL backend for Model Registry (Kustomize)
oc apply -k manifests/04/

# Step 5 — Enable LlamaStack Operator + create llm-d inference Gateway
oc patch DataScienceCluster default-dsc \
  --type=merge \
  --patch-file manifests/05/rhoai-dsc-enable-llamastack.yaml
oc apply -f manifests/05/openshift-ai-inference-gateway.yaml

# Step 6 — GPU HardwareProfile for RHOAI workloads
oc apply -f manifests/06/gpu-hardware-profile.yaml

# Step 7 — Deploy a LLMInferenceService (llm-d)
# Create the HuggingFace token secret first (required for hf:// model URIs)
cp user.env.example user.env        # fill in your HF_TOKEN
oc create secret generic huggingface-token \
  -n <your-namespace> \
  --from-env-file=user.env
# Apply the inference service manifest
oc apply -f manifests/07/llm-inference.yaml                        # Qwen3-0.6B from HuggingFace
# or
oc apply -f manifests/07/llm-inference-qwen25-7b-instruct.yaml     # Qwen2.5-7B-Instruct from Red Hat OCI registry
```

The `scratch/` directory is gitignored and used for generated/temporary cluster artifacts.
The `user.env` file is gitignored — copy `user.env.example` and fill in your token; never commit `user.env`.

## Manifest Structure

| Directory | Purpose |
|-----------|---------|
| `manifests/01/` | NFD Operator, NVIDIA GPU Operator, optional GPU sample workload |
| `manifests/02/` | KServe prerequisite operators: Service Mesh 3, Serverless, Authorino, Cert-Manager, JobSet, Red Hat Connectivity Link, LeaderWorkerSet |
| `manifests/03/` | RHOAI Operator (`rhods-operator`) and `DataScienceCluster` CRD |
| `manifests/04/` | MySQL deployment for Model Registry backend (Kustomize, targets `rhoai-model-registries` namespace) |
| `manifests/05/` | DSC patches and llm-d Gateway: LlamaStack enablement patch + `openshift-ai-inference` GatewayClass/Gateway |
| `manifests/06/` | GPU `HardwareProfile` for RHOAI workloads (`gpu-profile` in `redhat-ods-applications`) |
| `manifests/07/` | `LLMInferenceService` examples: Qwen3-0.6B (HuggingFace) and Qwen2.5-7B-Instruct (Red Hat OCI) |

## Key Configuration Details

- **RHOAI channel**: `stable-3.x` — must be set explicitly; the default `stable` channel still pins to v2.x.
- **Service Mesh**: uses `servicemeshoperator3` (not the legacy v2 operator name).
- **DataScienceCluster (`manifests/03/rhoai-operator-dsc.yaml`)**: controls which RHOAI components are `Managed` vs `Removed`. KServe raw deployment is set to `Headless`; Kueue and MLflow are `Removed` by default.
- **MySQL image**: `registry.redhat.io/rhel10/mysql-84` — data path is `/var/lib/mysql/data` (Red Hat image convention, not upstream `/var/lib/mysql`). All credentials come from the `mysql-credentials` Secret.
- **Model Registry namespace**: `rhoai-model-registries` (matches `registriesNamespace` in the DSC).
- **LlamaStack**: enabled via a `--type=merge` patch to `DataScienceCluster`; do NOT use `oc patch --patch-file` without `--type=merge` — the CRD only supports merge-patch and json-patch, not strategic-merge-patch.
- **llm-d Gateway** (`manifests/05/openshift-ai-inference-gateway.yaml`): `LLMInferenceService` defaults to a Gateway named `openshift-ingress/openshift-ai-inference` which must be created manually. Uses a dedicated `openshift-ai-inference` GatewayClass (controller: `openshift.io/gateway-controller/v1`) separate from the RHOAI data-science gateway. TLS reuses the `data-science-gateway-service-tls` secret. `allowedRoutes.from: All` permits HTTPRoutes from any model namespace.
- **GPU HardwareProfile** (`manifests/06/gpu-hardware-profile.yaml`): creates `gpu-profile` in `redhat-ods-applications`; referenced by `LLMInferenceService` via the `opendatahub.io/hardware-profile-name` annotation.
- **HuggingFace token**: `hf://` model URIs in `LLMInferenceService` require a `huggingface-token` Secret (key: `HF_TOKEN`) in the model namespace. Without it, anonymous downloads are rate-limited and the `storage-initializer` init container stalls silently. Copy `user.env.example` → `user.env`, fill in the token, then `oc create secret generic huggingface-token -n <ns> --from-env-file=user.env`. The `user.env` file is gitignored.

## Contributing

Work on the `contribution` branch (see `contribute.md`). PRs go to `IsaiahStapleton/rhoai-config-guide` upstream. The `scratch/` directory must not be committed.
