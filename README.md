# 🛠️ Guide to Configuring RHOAI for Model Deployment

[![OpenShift Ready](https://img.shields.io/badge/OpenShift-Ready-brightgreen)](https://www.openshift.com)
[![License](https://img.shields.io/badge/License-MIT-yellow)](LICENSE)


## Purpose

The purpose of this guide is to offer the simplest steps for configuring Red Hat OpenShift AI (RHOAI) for model deployment.
RHOAI does not come properly configured for GPU use and model deployment out of the box after installing the operator.
This guide will give you the steps needed to do so, as well as offer insight into why each step is necessary for model deployment. This guide focuses on preparing for deploying models using the vLLM ServingRuntime for KServe.

Required Operators:
- ***Node Feature Discovery (NFD) Operator:*** Detects and labels nodes based on hardware capabilities for proper AI workload scheduling.
- ***NVIDIA GPU Operator:*** Automates deployment of GPU drivers, CUDA libraries, and dependencies for AI workloads.
- ***OpenShift Service Mesh Operator:*** Provides Istio for managing secure communication between model-serving components.
- ***Red Hat OpenShift Serverless Operator:*** Provides Knative Serving for scalable and event-driven AI model deployment.
- ***Red Hat Authorino Operator:*** Provides authentication and authorization for secure access to AI model endpoints.
- ***OpenShift cert-manager Operator:*** Manages TLS certificates required by KServe and related components.
- ***JobSet Operator:*** Required by the RHOAI Trainer component for distributed training job management.
- ***Red Hat Connectivity Link Operator:*** Provides API gateway, authentication, rate limiting, and policy enforcement for LLMInferenceService.
- ***LeaderWorkerSet Operator:*** Enables wide expert parallelism for multi-node LLM inference workloads.
- ***Red Hat OpenShift AI Operator:*** Manages and deploys AI components and services within OpenShift.

Environment used for:
- Deploying the ***Granite model*** on RHOAI, using ***A100 NVIDIA GPU***, served using ***vLLM ServingRuntime for KServe***
- Gathering metrics and doing benchmarking on the deployed granite models 


## 1. Enabling GPU Support

### 1.1 Install the Node Feature Discovery (NFD) Operator 

***The NFD Operator is used to detect and label nodes based on their hardware capabilities. This is important that we can properly schedule AI workloads or deploy models on GPU nodes.***

Apply the Namespace, OperatorGroup, and Subscription object

```bash
oc apply -f manifests/01/nfd-operator.yaml
```

Verify the operator is installed and running before moving on to the next step.

```bash
oc get pods -n openshift-nfd -w
```

### 1.2 Apply the NFD Instance Object 

***After installing the NFD Operator, you must apply an NFD instance object to deploy the NFD DaemonSet, which scans nodes for hardware features and labels them accordingly.*** 

```bash
oc apply -f manifests/01/nfd-instance.yaml
```

### 1.3 Install the NVIDIA GPU Operator 

***The NVIDIA GPU Operator is essential for RHOAI since it is used to automate the deployment of GPU drivers, CUDA libraries, and other dependencies required for AI workloads and model serving.***

Apply the Namespace, OperatorGroup, and Subscription object

```bash
oc apply -f manifests/01/nvidia-gpu-operator.yaml
```

Verify the operator is installed and running before moving on to the next step.

### 1.4 Apply NVIDIA GPU ClusterPolicy 

***The NVIDIA GPU Operator itself only installs the operator framework, but doesn't automaticaly deploy the necessary components. We need to apply the ClusterPolicy as that is what enables and configures GPU support within the cluster.***

Create the ClusterPolicy. Following command lets you retrieve example ClusterPolicy from installed operator.

```bash
oc get csv \
 -n nvidia-gpu-operator \
 -l operators.coreos.com/gpu-operator-certified.nvidia-gpu-operator \
 -ojsonpath='{.items[0].metadata.annotations.alm-examples}' | \
jq '.[0]' > scratch/nvidia-gpu-clusterpolicy.json
```

Apply the ClusterPolicy

```bash
oc apply -f scratch/nvidia-gpu-clusterpolicy.json
```

### 1.5 (OPTIONAL) Running a Sample GPU Workload

In order to test if GPU support is now enabled correctly in your cluster, we can run a simple GPU workload. 

Create new namespace to run GPU workload

```bash
oc project sandbox || oc new-project sandbox
```

Create and run the GPU workload

```bash
oc apply -f manifests/01/nvidia-gpu-sample-app.yaml
```

Check the logs to see the output of the workload

```bash
oc logs cuda-vectoradd
```

If your GPU enabled nodes are configured correctly you shoud see an output as follows:

```
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```

## 2. Install RHOAI KServe Dependencies

***KServe provides scalable and efficient model serving capabilities, enabling deployment, inference, and monitoring of AI models within OpenShift.***

### 2.1 Install the OpenShift Service Mesh Operator

***The OpenShift Service Mesh Operator is required for KServe because KServe relies on Istio for managing communication between model serving components***

Apply the Subscription object to install the operator

```bash
oc apply -f manifests/02/servicemesh-subscription.yaml
```

### 2.2 Install the Red Hat OpenShift Serverless Operator

***The Red Hat OpenShift Serverless Operator is required because it provides Knative Serving, which enables serverless capabilities that assist in model deployment.***

```bash
oc apply -f manifests/02/serverless-operator.yaml
```

### 2.3 Install the Red Hat Authorino Operator

***The Red Hat Authorino Operator is required because it provides authentication for API requests, ensuring secure access to AI model endpoints.***

```bash
oc apply -f manifests/02/authorino-subscription.yaml
```

### 2.4 Install the OpenShift cert-manager Operator

***cert-manager is required by KServe and related components to manage TLS certificates automatically. Without it, LLMInferenceService cannot be used.***

Apply the Namespace, OperatorGroup, and Subscription object

```bash
oc apply -f manifests/02/cert-manager-operator.yaml
```

Verify the operator is installed and running before moving on.

```bash
oc get pods -n cert-manager-operator -w
```

### 2.5 Install the JobSet Operator

***The JobSet Operator is required by the RHOAI Trainer component for managing distributed training jobs. Without it, the Trainer component will fail to become ready.***

Apply the Namespace, OperatorGroup, and Subscription object

```bash
oc apply -f manifests/02/jobset-operator-subscription.yaml
```

Wait for the operator to be ready, then apply the JobSetOperator instance

```bash
oc apply -f manifests/02/jobset-operator.yaml
```

### 2.6 Install the Red Hat Connectivity Link Operator

***Red Hat Connectivity Link (Kuadrant) is required to enable LLMInferenceService, providing API gateway functionality, authentication, rate limiting, and traffic policy enforcement for LLM model endpoints.***

Apply the Namespace, OperatorGroup, and Subscription object

```bash
oc apply -f manifests/02/connectivity-link-operator.yaml
```

Verify the operator is installed and running before moving on.

```bash
oc get pods -n kuadrant-system -w
```

Once the operator is ready, enable the Kuadrant web console plugin so it appears in the OpenShift web console:

```bash
oc patch console.operator.openshift.io cluster --type=json \
  -p '[{"op":"add","path":"/spec/plugins/-","value":"kuadrant-console-plugin"}]'
```

### 2.7 Install the LeaderWorkerSet Operator

***The LeaderWorkerSet Operator is required for wide expert parallelism with LLMInferenceService — it enables multi-node LLM inference workloads where a model is sharded across multiple GPUs on multiple nodes.***

Apply the Namespace, OperatorGroup, and Subscription object

```bash
oc apply -f manifests/02/leader-worker-set-subscription.yaml
```

Wait for the operator to be ready, then apply the LeaderWorkerSetOperator instance

```bash
oc apply -f manifests/02/leader-worker-set-operator.yaml
```

## 3. Install the Red Hat OpenShift AI Operator

Apply the Namespace, OperatorGroup, and Subscription object

```bash
oc apply -f manifests/03/rhoai-operator.yaml
```

### 3.1 Install RHOAI Components

Wait for the RHOAI operator to be installed before proceeding with this step. (***Provide steps for checking***)

```bash
oc apply -f manifests/03/rhoai-operator-dsc.yaml
```
Once operator becomes ready you will see new option available at the upper right menu. See the screenshot below.

![alt text](./img/1.png)

After successfull login you can start working with RHOAI web interface.

![alt text](./img/2.png)

## 4. Deploy MySQL Backend for Model Registry

***The Model Registry provides a central store for tracking model versions, metadata, and deployment history. It requires a MySQL backend.***

Apply the Kustomize configuration (creates namespace, deployment, service, and credentials Secret):

```bash
oc apply -k manifests/04/
```

## 5. Enable LlamaStack Operator

***The LlamaStack Operator adds support for running Meta Llama models in the cluster via the RHOAI dashboard.***

Enable the component by patching the DataScienceCluster:

```bash
oc patch DataScienceCluster default-dsc \
  --type=merge \
  --patch-file manifests/05/rhoai-dsc-enable-llamastack.yaml
```

Verify the operator pod is running:

```bash
oc get pods -n redhat-ods-applications | grep llama
```

## 6. Create the llm-d Inference Gateway

***`LLMInferenceService` (llm-d) routes external traffic through a dedicated Gateway named `openshift-ai-inference`. This Gateway and its GatewayClass must be created before deploying any `LLMInferenceService`.***

Apply the GatewayClass and Gateway:

```bash
oc apply -f manifests/05/openshift-ai-inference-gateway.yaml
```

Verify the Gateway is programmed and gets an address:

```bash
oc get gateway openshift-ai-inference -n openshift-ingress
```

## 7. Create the GPU Hardware Profile

***The GPU HardwareProfile defines the resource limits and requests for GPU-accelerated workloads deployed through the RHOAI dashboard.***

```bash
oc apply -f manifests/06/gpu-hardware-profile.yaml
```

## 8. Enable Dashboard Features

RHOAI 3.4 has many UI features that are hidden by default. Understanding which resource to patch is non-obvious:

| Resource | Controls |
|----------|----------|
| `DataScienceCluster` | Which operator **components** are deployed (KServe, workbenches, trainer, …) |
| `OdhDashboardConfig` | Which **UI features** are visible in the dashboard navigation |

### 8.1 Enable Gen AI Studio

***Gen AI Studio is the visual AI application builder. It is a UI toggle — the underlying operator is already present, but the dashboard hides it by default.***

```bash
oc patch OdhDashboardConfig odh-dashboard-config \
  -n redhat-ods-applications \
  --type=merge \
  --patch-file manifests/08/odh-dashboard-config-enable-aistudio.yaml
```

Verify (check `OdhDashboardConfig`, **not** `DataScienceCluster`):

```bash
oc get OdhDashboardConfig odh-dashboard-config \
  -n redhat-ods-applications \
  -o jsonpath='{.spec.dashboardConfig}'
```

Expected output contains `"genAiStudio":true`. Hard-refresh the dashboard — no pod restart needed.

### 8.2 Enable Additional UI Features

***Training Jobs, LLM Gateway field, MCP Catalog, and Prompt Management are all opt-in flags in `OdhDashboardConfig`. All required backend components are already deployed.***

```bash
oc patch OdhDashboardConfig odh-dashboard-config \
  -n redhat-ods-applications \
  --type=merge \
  --patch-file manifests/08/odh-dashboard-config-enable-features.yaml
```

| Flag | What appears in the dashboard |
|------|-------------------------------|
| `trainingJobs` | Training jobs UI under "Develop & train" (Kubeflow Trainer v2 — progress, checkpoints, distributed jobs) |
| `llmGatewayField` | LLM Gateway configuration field in model serving UI |
| `mcpCatalog` | MCP (Model Context Protocol) tools/servers catalog |
| `promptManagement` | Prompt management UI for LLM prompts |

### 8.4 Enable MLflow

***MLflow requires two steps: enabling the operator component in the DataScienceCluster, then creating the MLflow server instance. The `mlflow` flag in `OdhDashboardConfig` is deprecated in RHOAI 3.4 — ignore it.***

> **Important:** Enabling the operator alone is not enough. The MLflow operator does not auto-create a server instance. Without the `MLflow` CR, the dashboard shows "MLflow is currently unavailable" even though the operator pod is running.

**Step 1 — enable the operator component** (patches `DataScienceCluster`):

```bash
oc patch DataScienceCluster default-dsc \
  --type=merge \
  --patch-file manifests/08/rhoai-dsc-enable-mlflow.yaml
```

Verify the operator is ready:

```bash
oc get DataScienceCluster default-dsc \
  -o jsonpath='{.status.conditions[?(@.type=="MLflowOperatorReady")]}'
```

Expected: `"status":"True"`.

**Step 2 — create the MLflow server instance** (cluster-scoped `MLflow` CR):

```bash
oc apply -f manifests/08/mlflow-instance.yaml
```

This deploys an MLflow server with:
- SQLite backend + 10 Gi PVC (no S3 bucket required)
- Artifact serving built into the MLflow server
- All RHOAI data science project namespaces exposed as workspaces (selected by the `opendatahub.io/dashboard: "true"` label)

Verify the instance is ready and note the URL:

```bash
oc get mlflow mlflow -o jsonpath='{.status}' | python3 -m json.tool
```

Once `"Available": true`, the "Experiments (MLflow)" section and "Launch MLflow" link in the dashboard will both work.

### 8.5 Enable Models-as-a-Service (MaaS)

***MaaS provides an AI inference gateway with token quotas, rate limiting, and self-service API keys. It is GA in RHOAI 3.4 and requires Kuadrant (already installed in Step 2.6). Full setup has two parts: gateway/DSC/UI wiring (steps 8.5.1–8.5.3) and infrastructure prerequisites (steps 8.5.4–8.5.7). Both parts must be complete for `ModelsAsServiceReady` to become True.***

#### 8.5.1 Create the MaaS Gateway

> **Important:** The MaaS DSC component requires a Gateway named `maas-default-gateway` in `openshift-ingress` to exist **before** the DSC reconcile runs. The name is hardcoded — the reconcile will fail with "GatewayNotReady" if the gateway is not created first.

Apply the GatewayClass and Gateway (same pattern as the llm-d gateway in Step 6):

```bash
oc apply -f manifests/08/maas-gateway.yaml
```

Verify the Gateway is programmed:

```bash
oc get gateway maas-default-gateway -n openshift-ingress
```

#### 8.5.2 Enable `modelsAsService` in DataScienceCluster

```bash
oc patch DataScienceCluster default-dsc \
  --type=merge \
  --patch-file manifests/08/rhoai-dsc-enable-maas.yaml
```

#### 8.5.3 Enable MaaS UI in OdhDashboardConfig

```bash
oc patch OdhDashboardConfig odh-dashboard-config \
  -n redhat-ods-applications \
  --type=merge \
  --patch-file manifests/08/odh-dashboard-config-enable-maas.yaml
```

#### 8.5.4 Deploy PostgreSQL for MaaS

MaaS requires a PostgreSQL database to store API key lifecycle data. The manifests in `manifests/09/` deploy a PostgreSQL instance inside `redhat-ods-applications`.

```bash
oc apply -f manifests/09/maas-postgresql.yaml
oc get pods -n redhat-ods-applications -l app=maas-postgresql -w   # wait until Running
```

#### 8.5.5 Create the `maas-db-config` Secret

The MaaS reconciler looks for a Secret named `maas-db-config` in `redhat-ods-applications` with a `DB_CONNECTION_URL` key. Without it, `ModelsAsServiceReady` stays `False (PrerequisitesNotMet)`.

```bash
oc apply -f manifests/09/maas-db-config.yaml
```

Verify the Secret exists:

```bash
oc get secret maas-db-config -n redhat-ods-applications
```

#### 8.5.6 Deploy Authorino Instance

MaaS uses Authorino for API key authentication and authorization. The Authorino Operator was installed in Step 2.3 but requires an explicit instance to be created.

```bash
oc apply -f manifests/09/authorino-instance.yaml
oc get pods -n redhat-ods-applications -l app.kubernetes.io/name=authorino -w
```

#### 8.5.7 (Optional) Enable User Workload Monitoring

User Workload Monitoring enables MaaS token usage metrics per model. It is optional — MaaS will become ready without it, but showback dashboards will not be available.

```bash
oc apply -f manifests/09/cluster-monitoring-config.yaml
```

#### Verify MaaS is Ready

After all prerequisites exist, the MaaS reconciler may take 2–3 minutes to re-evaluate:

```bash
oc get DataScienceCluster default-dsc \
  -o jsonpath='{.status.conditions[?(@.type=="ModelsAsServiceReady")]}'
```

Expected: `"status":"True"`. Once ready, the overall DSC returns to `Ready` phase and the "Models as a Service" section appears in the RHOAI dashboard.

## 9. Deploy a LLMInferenceService (llm-d)

***`LLMInferenceService` is the llm-d custom resource for serving large language models. It manages vLLM pods, an EPP scheduler, and HTTPRoute wiring automatically.***

### 9.1 Create the target namespace

```bash
oc new-project my-first-model
```

### 9.2 Create the HuggingFace token Secret

Required when using `hf://` model URIs. Without it, the `storage-initializer` init container stalls silently due to anonymous rate limits.

```bash
cp user.env.example user.env   # fill in your HF_TOKEN value
oc create secret generic huggingface-token \
  -n my-first-model \
  --from-env-file=user.env
```

> **Never commit `user.env`** — it is gitignored. Use `user.env.example` as the template.

### 9.3 Apply the LLMInferenceService manifest

Two examples are provided:

**Qwen3-0.6B from HuggingFace** (uses `hf://` URI, requires `huggingface-token` secret):

```bash
oc apply -f manifests/07/llm-inference.yaml
```

**Qwen2.5-7B-Instruct from Red Hat OCI registry** (uses `oci://` URI, no HF token needed):

```bash
oc apply -f manifests/07/llm-inference-qwen25-7b-instruct.yaml
```

### 9.4 Monitor startup

```bash
oc get pods -n my-first-model -w
oc get llminferenceservice -n my-first-model
```

All conditions should reach `True`. The inference endpoint URL is shown in the `URL` column once `Ready` is `True`.

## 🤝 Contributing

Feel free to submit issues, pull requests, or suggest new features.

## ⚡ License

This repository is licensed under the MIT License. See LICENSE for details.