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
oc apply -f manifests/02/kuadrant-instance.yaml   # activates Kuadrant; required for AuthPolicy enforcement
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
# 7a — Create the dybbol project (makes it visible in RHOAI dashboard as a Data Science Project)
oc apply -f manifests/07/dybbol-project.yaml

# 7b — Create the HuggingFace token secret (required for hf:// model URIs only)
cp user.env.example user.env        # fill in your HF_TOKEN
oc create secret generic huggingface-token \
  -n dybbol \
  --from-env-file=user.env
# Apply the inference service manifest
oc apply -f manifests/07/llm-inference.yaml                        # Qwen3-0.6B from HuggingFace
# or
oc apply -f manifests/07/llm-inference-qwen25-7b-instruct.yaml     # Qwen2.5-7B-Instruct from Red Hat OCI registry

# Step 8 — Enable dashboard features
# 8a — Gen AI Studio (UI flag in OdhDashboardConfig)
oc patch OdhDashboardConfig odh-dashboard-config \
  -n redhat-ods-applications \
  --type=merge \
  --patch-file manifests/08/odh-dashboard-config-enable-aistudio.yaml
# 8b — Additional UI features: Training Jobs, LLM Gateway field, MCP Catalog, Prompt Management
oc patch OdhDashboardConfig odh-dashboard-config \
  -n redhat-ods-applications \
  --type=merge \
  --patch-file manifests/08/odh-dashboard-config-enable-features.yaml
# 8c — MLflow: (1) enable operator in DSC; (2) create server instance (operator does not auto-create it)
oc patch DataScienceCluster default-dsc \
  --type=merge \
  --patch-file manifests/08/rhoai-dsc-enable-mlflow.yaml
oc apply -f manifests/08/mlflow-instance.yaml
# 8d — Models-as-a-Service (MaaS): requires gateway + DSC change + UI flag
oc apply -f manifests/08/maas-gateway.yaml           # GatewayClass + maas-default-gateway (required before DSC reconcile)
oc patch DataScienceCluster default-dsc \
  --type=merge \
  --patch-file manifests/08/rhoai-dsc-enable-maas.yaml
oc patch OdhDashboardConfig odh-dashboard-config \
  -n redhat-ods-applications \
  --type=merge \
  --patch-file manifests/08/odh-dashboard-config-enable-maas.yaml

# Step 9 — MaaS infrastructure prerequisites: PostgreSQL + Authorino + optional User Workload Monitoring
# Note: Steps 8d gateway/DSC/UI must be applied before or after Step 9 — both must be complete for
# ModelsAsServiceReady to become True. The order between 8d and 9 does not matter.

# 9a — Deploy PostgreSQL and create the maas-db-config Secret
oc apply -f manifests/09/maas-postgresql.yaml
oc get pods -n redhat-ods-applications -l app=maas-postgresql -w   # wait until Running
oc apply -f manifests/09/maas-db-config.yaml

# 9b — Deploy Authorino instance in redhat-ods-applications
oc apply -f manifests/09/authorino-instance.yaml
oc get pods -n redhat-ods-applications -l app.kubernetes.io/name=authorino -w

# 9c — (Optional) Enable User Workload Monitoring
oc apply -f manifests/09/cluster-monitoring-config.yaml

# Verify ModelsAsService is Ready (allow 2-3 min after all prerequisites exist)
oc get DataScienceCluster default-dsc \
  -o jsonpath='{.status.conditions[?(@.type=="ModelsAsServiceReady")]}'

# 9d — Create DNS record so maas.<apps-domain> routes to the maas-default-gateway ELB
# The wildcard *.apps DNS points to the OpenShift Router, not the maas-default-gateway ELB.
# Without this record the dashboard API keys panel returns 503 (wrong backend).
MAAS_ELB=$(oc get svc maas-default-gateway-maas-gateway-class -n openshift-ingress \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
APPS_DOMAIN=$(oc get ingresses.config cluster -o jsonpath='{.spec.domain}')
cat <<EOF | oc apply -f -
apiVersion: ingress.operator.openshift.io/v1
kind: DNSRecord
metadata:
  name: maas-api-dns
  namespace: openshift-ingress-operator
spec:
  dnsManagementPolicy: Managed
  dnsName: "maas.${APPS_DOMAIN}."
  recordTTL: 30
  recordType: CNAME
  targets:
    - ${MAAS_ELB}
EOF
# Verify DNS is published to Route53 (status.zones[*].conditions[*].type=Published)
oc get dnsrecord maas-api-dns -n openshift-ingress-operator \
  -o jsonpath='{.status.zones[0].conditions[0]}'

# 9e — Register the deployed model with MaaS by creating a MaaSModelRef
# MaaSModelRef is the bridge between LLMInferenceService and the MaaS control plane.
# Without it, Subscriptions and Authorization Policies cannot reference any model.
oc apply -f manifests/09/maas-model-ref.yaml
oc get maasmodelref qwen25-7b-instruct -n dybbol \
  -o jsonpath='{.status.phase}'   # expect: Ready

# After MaaSModelRef is Ready, complete the admin flow in the dashboard:
#   Settings → Subscriptions → Create subscription (assign model to user groups)
#   Settings → Authorization policies → Create authorization policy (enforce access)
#   Gen AI Studio → API keys → Create API key (user generates a Bearer token)
# Test the MaaS endpoint with the issued key:
APPS_DOMAIN=$(oc get ingresses.config cluster -o jsonpath='{.spec.domain}')
curl -H "Authorization: Bearer <api-key>" \
  https://maas.${APPS_DOMAIN}/v1/models
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
| `manifests/07/` | `dybbol` project namespace + `LLMInferenceService` examples: Qwen3-0.6B (HuggingFace) and Qwen2.5-7B-Instruct (Red Hat OCI) |
| `manifests/08/` | Dashboard feature enablement: Gen AI Studio, additional UI flags, MLflow (operator + instance), and MaaS (gateway + DSC + UI flag) |
| `manifests/09/` | MaaS infrastructure prerequisites: PostgreSQL deployment + credentials, `maas-db-config` Secret, Authorino instance, optional User Workload Monitoring, DNS record template for maas gateway, `MaaSModelRef` to register the deployed model |

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
- **OdhDashboardConfig opt-in flags**: RHOAI 3.4 has many boolean flags in `OdhDashboardConfig.spec.dashboardConfig` that are hidden by default. The two categories are: `disable*` flags (default `false` = feature visible; set `true` to hide) and non-`disable` flags (opt-in; must be set `true` to appear). All the features in `manifests/08/` use one or both resources. Verify all current flags with `oc get OdhDashboardConfig odh-dashboard-config -n redhat-ods-applications -o jsonpath='{.spec.dashboardConfig}'`.
- **Gen AI Studio** (`manifests/08/odh-dashboard-config-enable-aistudio.yaml`): sets `genAiStudio: true` in the `OdhDashboardConfig` resource — **not** in `DataScienceCluster`. This is a common point of confusion: the DSC controls which operator components run; `OdhDashboardConfig` controls which UI features are visible. The flag defaults to `false` in RHOAI 3.4, so "AI Studio" never appears in the dashboard navigation until it is explicitly set. To verify: `oc get OdhDashboardConfig odh-dashboard-config -n redhat-ods-applications -o jsonpath='{.spec.dashboardConfig}'`. Apply with `--type=merge`.
- **Additional UI flags** (`manifests/08/odh-dashboard-config-enable-features.yaml`): enables `trainingJobs`, `llmGatewayField`, `mcpCatalog`, and `promptManagement` in one patch. All are opt-in booleans in `OdhDashboardConfig`. Prerequisites already met: `trainer: Managed`, llm-d gateway deployed, `llamastackoperator: Managed`.
- **MLflow** (`manifests/08/rhoai-dsc-enable-mlflow.yaml` + `manifests/08/mlflow-instance.yaml`): enabling MLflow requires **two separate steps**. First, set `mlflowoperator: Managed` in `DataScienceCluster` to deploy the operator (the `mlflow` flag in `OdhDashboardConfig` is deprecated — ignore it). Second, create the cluster-scoped `MLflow` CR (`mlflow-instance.yaml`) — the operator does not auto-create an instance. Without the `MLflow` CR, the dashboard shows "MLflow is currently unavailable" even though the operator pod is running. The instance uses SQLite + PVC for persistence (no S3 needed) and `workspaceLabelSelector: opendatahub.io/dashboard: "true"` to expose all RHOAI data science project namespaces. Verify: `oc get mlflow mlflow -o jsonpath='{.status}'`.
- **Models-as-a-Service / MaaS** (`manifests/08/maas-gateway.yaml` + `manifests/08/rhoai-dsc-enable-maas.yaml` + `manifests/08/odh-dashboard-config-enable-maas.yaml`): MaaS requires **three steps**. (1) Create `maas-default-gateway` in `openshift-ingress` — the name is hardcoded; the DSC reconcile fails with "GatewayNotReady" if it doesn't exist first. (2) Set `kserve.modelsAsService: Managed` in DSC. (3) Set `modelAsService: true` in `OdhDashboardConfig`. Kuadrant (RHCL) must be installed — it is (Step 2). Verify: `oc get DataScienceCluster default-dsc -o jsonpath='{.status.conditions[?(@.type=="ModelsAsServiceReady")]}'`.
- **MaaS infrastructure prerequisites** (`manifests/09/`): even after the three MaaS steps above, `ModelsAsServiceReady` stays `False (PrerequisitesNotMet)` until three more things exist in `redhat-ods-applications`: (1) the `maas-db-config` Secret with a valid PostgreSQL `DB_CONNECTION_URL`, (2) a running Authorino instance, and (3) optionally the `cluster-monitoring-config` ConfigMap in `openshift-monitoring`. All three are in `manifests/09/`.
- **`maas-db-config` Secret** (`manifests/09/maas-db-config.yaml`): key `DB_CONNECTION_URL`, value format `postgresql://user:pass@host:5432/db?sslmode=disable`. The companion PostgreSQL deployment (`maas-postgresql.yaml`) runs in `redhat-ods-applications` and is reachable in-cluster at `maas-postgresql.redhat-ods-applications.svc.cluster.local:5432`.
- **Authorino instance for MaaS** (`manifests/09/authorino-instance.yaml`): The Authorino Operator (installed in Step 2.3) requires an `Authorino` CR (`operator.authorino.kuadrant.io/v1beta1`) deployed in `redhat-ods-applications`. `clusterWide: true` lets it watch `AuthConfig` CRs across all model namespaces. TLS is disabled on the listener — plain HTTP is sufficient for in-cluster MaaS communication.
- **PostgreSQL image for MaaS**: `registry.redhat.io/rhel9/postgresql-15` — data path is `/var/lib/pgsql/data` (Red Hat convention). Credentials come from the `maas-postgresql-credentials` Secret; the `maas-db-config` Secret must reference the same password value.
- **Kuadrant CR** (`manifests/02/kuadrant-instance.yaml`): The RHCL (Kuadrant) operator alone is not enough — a `Kuadrant` CR must be created in `kuadrant-system` to activate the operator and wire up its components (Authorino, Limitador, DNS Operator). Without the CR, `AuthPolicy` resources are accepted but never enforced (`"kuadrant is not installed, please create resource"`), meaning Kuadrant's Authorino is never started and no `AuthConfig` resources are generated. Apply immediately after the RHCL operator is ready. Verify: `oc get kuadrant kuadrant -n kuadrant-system -o jsonpath='{.status.conditions[0].message}'` → `"Kuadrant is ready"`.
- **`opendatahub.io/connections` annotation portability**: The RHOAI dashboard injects an `opendatahub.io/connections: <secret-name>` annotation when you create a `LLMInferenceService` or `InferenceService` through the UI. The mutating webhook `connection-llmisvc.opendatahub.io` (failurePolicy: Fail) validates that the named secret exists in the target namespace on every CREATE/UPDATE — if it doesn't, the apply is denied. Dashboard-generated secret names (e.g. `secret-6a153e`) are auto-generated and namespace-specific; never commit them to manifests. Remove the annotation for images pulled via the cluster pull secret (`registry.redhat.io`, `quay.io`, etc.). Only include it when you have pre-created a named data connection secret in the target namespace and want the dashboard to display the connection.
- **`dybbol` project namespace** (`manifests/07/dybbol-project.yaml`): The model deployment namespace must carry the label `opendatahub.io/dashboard: "true"` to appear as a Data Science Project in the RHOAI dashboard. Without this label the namespace exists but is invisible to the dashboard. The label `modelmesh-enabled: "false"` tells RHOAI to use KServe/llm-d rather than ModelMesh for serving. Apply with `oc apply -f manifests/07/dybbol-project.yaml` before deploying the `LLMInferenceService` — the namespace must exist first.
- **`MaaSModelRef`** (`manifests/09/maas-model-ref.yaml`): After the MaaS control plane is ready (`ModelsAsServiceReady: True`) and a `LLMInferenceService` is running, a `MaaSModelRef` CR must be created in the **same namespace** as the inference service to register it with MaaS. Without this CR, the Subscriptions and Authorization Policies forms in the dashboard show "No models available" and API keys cannot be issued. The spec uses `spec.modelRef.kind: LLMInferenceService` and `spec.modelRef.name: <service-name>`. Verify: `oc get maasmodelref -n dybbol -o jsonpath='{.items[0].status.phase}'` → `Ready`. Once Ready, the model appears in the Subscriptions dropdown and the full admin flow (Subscription → Authorization Policy → API key → curl) becomes available.
- **MaaS requires `maas-default-gateway` in `LLMInferenceService`**: A `LLMInferenceService` intended for MaaS must explicitly set `spec.router.gateway.refs` to `maas-default-gateway` in `openshift-ingress`. The default (`gateway: {}`) wires the HTTPRoute to `openshift-ai-inference` (the llm-d gateway from Step 5). The `MaaSModelRef` controller validates the HTTPRoute's `parentRefs` and sets phase `Failed` with "does not reference gateway (expected: openshift-ingress/maas-default-gateway)" if the default is left in place. See `manifests/07/llm-inference-qwen25-7b-instruct.yaml` for the correct `router.gateway.refs` pattern.
- **MaaS DNS routing** (`manifests/09/maas-dns-record.yaml`): The `maas-ui` sidecar inside the dashboard auto-discovers the MaaS API URL as `maas.<apps-domain>` (derived from `GATEWAY_DOMAIN` env var). On clusters where `*.apps` is a wildcard CNAME to the OpenShift Router, this URL resolves to the Router — not the `maas-default-gateway` ELB. The Router has no backend for this host and returns 503, causing the dashboard API keys panel to fail. Fix: create a specific `DNSRecord` (ingress.operator.openshift.io/v1) for `maas.<apps-domain>` as a CNAME to the `maas-default-gateway` ELB. This record overrides the wildcard in Route53 and is managed by the same ingress operator that manages the `*.apps` wildcard. The manifest in `manifests/09/maas-dns-record.yaml` is a template with `<APPS_DOMAIN>` and `<MAAS_ELB>` placeholders — use the dynamic command in Step 9d instead of applying the template directly.

## Pending Work

- [ ] **Test MaaS end-to-end** — create a Subscription and Authorization Policy via the dashboard, generate an API key, confirm `curl https://maas.<apps-domain>/v1/chat/completions` returns a valid response. Document any additional errors encountered.
- [ ] **Declarative Subscription / Authorization Policy manifests** — the admin flow (Subscription → AuthPolicy) is currently dashboard-only. Investigate whether `MaaSSubscription` and `MaaSAuthorizationPolicy` CRDs exist (`oc get crd | grep maas`) and, if so, add manifests to `manifests/09/` so the full MaaS setup is reproducible without the UI.
- [ ] **`llm-inference.yaml` (HuggingFace / Qwen3-0.6B) namespace alignment** — this manifest still creates `my-first-model` inline. Decide whether it should also target `dybbol` (for consistency) or stay as a standalone quick-start example with its own namespace.
- [ ] **HuggingFace example MaaS-readiness** — if the HuggingFace model should also be served via MaaS, update `llm-inference.yaml` to use `maas-default-gateway` and create a second `MaaSModelRef` for it.

## Contributing

Work on the `contribution` branch (see `contribute.md`). PRs go to `IsaiahStapleton/rhoai-config-guide` upstream. The `scratch/` directory must not be committed.
