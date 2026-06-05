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
```

The `scratch/` directory is gitignored and used for generated/temporary cluster artifacts.

## Manifest Structure

| Directory | Purpose |
|-----------|---------|
| `manifests/01/` | NFD Operator, NVIDIA GPU Operator, optional GPU sample workload |
| `manifests/02/` | KServe prerequisite operators: Service Mesh 3, Serverless, Authorino, Cert-Manager, JobSet, Red Hat Connectivity Link, LeaderWorkerSet |
| `manifests/03/` | RHOAI Operator (`rhods-operator`) and `DataScienceCluster` CRD |
| `manifests/04/` | MySQL deployment for Model Registry backend (Kustomize, targets `rhoai-model-registries` namespace) |

## Key Configuration Details

- **RHOAI channel**: `stable-3.x` — must be set explicitly; the default `stable` channel still pins to v2.x.
- **Service Mesh**: uses `servicemeshoperator3` (not the legacy v2 operator name).
- **DataScienceCluster (`manifests/03/rhoai-operator-dsc.yaml`)**: controls which RHOAI components are `Managed` vs `Removed`. KServe raw deployment is set to `Headless`; Kueue and MLflow are `Removed` by default.
- **MySQL image**: `registry.redhat.io/rhel10/mysql-84` — data path is `/var/lib/mysql/data` (Red Hat image convention, not upstream `/var/lib/mysql`). All credentials come from the `mysql-credentials` Secret.
- **Model Registry namespace**: `rhoai-model-registries` (matches `registriesNamespace` in the DSC).

## Contributing

Work on the `contribution` branch (see `contribute.md`). PRs go to `IsaiahStapleton/rhoai-config-guide` upstream. The `scratch/` directory must not be committed.
