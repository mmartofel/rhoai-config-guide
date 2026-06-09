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

## 8. Deploy a LLMInferenceService (llm-d)

***`LLMInferenceService` is the llm-d custom resource for serving large language models. It manages vLLM pods, an EPP scheduler, and HTTPRoute wiring automatically.***

### 8.1 Create the target namespace

```bash
oc new-project my-first-model
```

### 8.2 Create the HuggingFace token Secret

Required when using `hf://` model URIs. Without it, the `storage-initializer` init container stalls silently due to anonymous rate limits.

```bash
cp user.env.example user.env   # fill in your HF_TOKEN value
oc create secret generic huggingface-token \
  -n my-first-model \
  --from-env-file=user.env
```

> **Never commit `user.env`** — it is gitignored. Use `user.env.example` as the template.

### 8.3 Apply the LLMInferenceService manifest

Two examples are provided:

**Qwen3-0.6B from HuggingFace** (uses `hf://` URI, requires `huggingface-token` secret):

```bash
oc apply -f manifests/07/llm-inference.yaml
```

**Qwen2.5-7B-Instruct from Red Hat OCI registry** (uses `oci://` URI, no HF token needed):

```bash
oc apply -f manifests/07/llm-inference-qwen25-7b-instruct.yaml
```

### 8.4 Monitor startup

```bash
oc get pods -n my-first-model -w
oc get llminferenceservice -n my-first-model
```

All conditions should reach `True`. The inference endpoint URL is shown in the `URL` column once `Ready` is `True`.

## 🤝 Contributing

Feel free to submit issues, pull requests, or suggest new features.

## ⚡ License

This repository is licensed under the MIT License. See LICENSE for details.