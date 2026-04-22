# vLLM Model Provisioner

This Helm chart is designed to decouple the downloading of Large Language Models (LLMs) from Hugging Face from your actual vLLM serving workloads. It allows you to declaratively provision, track, and remove models on demand using PersistentVolumeClaims (PVCs) in an OpenShift/Kubernetes environment.

## Features

- **Declarative Management:** Set a model's state to `present` to download it, or `absent` to automatically delete the PVC and free up storage.
- **State Tracking:** Writes a `.download_status` file (`IN_PROGRESS`, `SUCCESS`, `FAILED`) to the PVC so consumer pods know when the model is ready.
- **Model Registry:** Automatically maintains a Kubernetes `ConfigMap` mapping model names to their underlying PVC names for easy discovery.
- **Optimized for OpenShift:** Runs as non-root, handles ephemeral storage limits using `emptyDir` caches, and natively supports RWX storage classes like OpenShift Data Foundation (ODF).

## Prerequisites

1. A Kubernetes/OpenShift cluster.
2. A StorageClass capable of dynamic provisioning (ReadWriteMany/RWX is highly recommended if scaling vLLM pods).
3. A Hugging Face account and an Access Token (for gated models like Llama-3).

## Installation

### 1. Create the Hugging Face Token Secret

Before deploying the chart, create a secret containing your Hugging Face access token:

```bash
kubectl create secret generic hf-token-secret \
  --from-literal=token="hf_YOUR_TOKEN_HERE"
```

### 2. Configure `values.yaml`

Define the models you want to download. Assign a unique name, the Hugging Face repository ID, a PVC size large enough to hold the model, and set the state to `present`.

```yaml
storage:
  storageClass: "ocs-storagecluster-cephfs" # Change to your cluster's StorageClass
  accessMode: ReadWriteMany

huggingface:
  tokenSecretName: hf-token-secret

models:
  - name: llama-3-8b
    hfRepo: "meta-llama/Meta-Llama-3-8B-Instruct"
    pvcName: "pvc-llama-3-8b"
    size: "50Gi"
    state: "present"
```

### 3. Deploy the Chart

```bash
helm install model-provisioner ./vllm-download
```

## Checking Download Status

You can monitor the provisioning process in several ways:

**Check the Kubernetes Jobs:**
```bash
kubectl get jobs -l app.kubernetes.io/name=model-provisioner
```

**View the Model Registry:**
To see exactly which PVCs hold which downloaded models:
```bash
kubectl get configmap available-models-registry -o yaml
```

## Consuming the Models in vLLM

In your separate vLLM deployment chart, mount the exact `pvcName` configured above. 

It is highly recommended to use an `initContainer` to check the `.download_status` marker file. This prevents your vLLM pod from crashing while trying to load a partially downloaded model:

```yaml
      initContainers:
        - name: check-download-status
          image: busybox
          command: ["/bin/sh", "-c"]
          args:
            - |
              STATUS=$(cat /model-data/.download_status)
              if [ "$STATUS" != "SUCCESS" ]; then
                echo "Model download is not complete. Current status: $STATUS"
                exit 1
              fi
              echo "Model download verified successful!"
          volumeMounts:
            - name: model-volume
              mountPath: /model-data
```

## Removing a Model

To delete a model and its associated PVC, simply update your `values.yaml` and change the `state` from `present` to `absent`, then upgrade the release:

```yaml
  - name: llama-3-8b
    ...
    state: "absent" 
```

```bash
helm upgrade model-provisioner ./vllm-download
```