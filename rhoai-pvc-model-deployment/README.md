# Deploying AI Models from RWX PVC on MOC OpenShift

This guide covers creating a ReadWriteMany (RWX) PVC for model storage and deploying AI models from it using Red Hat OpenShift AI (RHOAI) within the MOC.

## Overview

Using a RWX PVC for model storage provides several advantages:
- **Shared access**: Multiple inference pods can mount the same PVC simultaneously
- **No container rebuilds**: Update models without rebuilding container images
- **Research workflows**: Fine-tune models and save updated weights back to PVC
- **Easier scaling**: Support horizontal pod autoscaling with shared model storage


## Step 1: Create a RWX PVC

### 1.1 Create PVC

Create a file named `model-storage-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-storage-nfs
  namespace: ai-performance-profiling  # Change to your namespace
  labels:
    app: model-storage
    purpose: ai-models
    opendatahub.io/dashboard: "true"  # Makes PVC visible in RHOAI UI
  annotations:
    openshift.io/display-name: "Model Storage (NFS RWX)"
spec:
  accessModes:
    - ReadWriteMany  # RWX - multiple pods can mount simultaneously
  resources:
    requests:
      storage: 200Gi  # Adjust based on your needs
  storageClassName: nfs-csi  # Provides RWX storage
  volumeMode: Filesystem
```

### 1.2 Apply the PVC

```bash
oc apply -f model-storage-pvc.yaml
```

### 1.3 Verify PVC is Bound

```bash
oc get pvc model-storage-nfs -n ai-performance-profiling
```

Expected output:
```
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
model-storage-nfs   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   200Gi      RWX            nfs-csi        10s
```

**Status should be `Bound`**. If stuck in `Pending`, check storage class availability:
```bash
oc get storageclass nfs-csi
```

## Step 2: Create Model Uploader Pod

To upload models to the PVC, create a helper pod with the PVC mounted.

### 2.1 Create Uploader Pod YAML

Create a file named `model-uploader-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: model-uploader-nfs
  namespace: ai-performance-profiling  # Change to your namespace
  labels:
    app: model-uploader
spec:
  containers:
  - name: uploader
    image: registry.access.redhat.com/ubi9/python-311:latest
    command: ["/bin/bash", "-c", "sleep infinity"]
    volumeMounts:
    - name: models
      mountPath: /models
    resources:
      requests:
        memory: "4Gi"
        cpu: "2"
      limits:
        memory: "8Gi"
        cpu: "4"
  volumes:
  - name: models
    persistentVolumeClaim:
      claimName: model-storage-nfs
  restartPolicy: Never
```

### 2.2 Create the Uploader Pod

```bash
oc apply -f model-uploader-pod.yaml
```

### 2.3 Wait for Pod to be Ready

```bash
oc wait --for=condition=Ready pod/model-uploader-nfs -n ai-performance-profiling --timeout=300s
```

## Step 3: Upload Models to PVC

### Option A: Upload from Local Directory

**Use case**: You have a model downloaded locally

```bash
# Create destination directory in PVC
oc exec -n ai-performance-profiling model-uploader-nfs -- mkdir -p /models/MODEL_NAME

# Upload using tar (excludes git artifacts)
tar czf - -C /path/to/local/model --exclude='.git' --exclude='.cache' --exclude='.gitattributes' . | \
  oc exec -i -n ai-performance-profiling model-uploader-nfs -- tar xzf - -C /models/MODEL_NAME

# Verify upload
oc exec -n ai-performance-profiling model-uploader-nfs -- ls -lh /models/MODEL_NAME
oc exec -n ai-performance-profiling model-uploader-nfs -- du -sh /models/MODEL_NAME
```

**Example**:
```bash
# Upload Qwen3-TTS model
tar czf - -C /home/user/ai/models/Qwen3-TTS-12Hz-1.7B-CustomVoice \
  --exclude='.git' --exclude='.cache' . | \
  oc exec -i -n ai-performance-profiling model-uploader-nfs -- \
  tar xzf - -C /models/Qwen3-TTS-12Hz-1.7B-CustomVoice
```

### Option B: Download from HuggingFace Directly into PVC


```bash
# Get shell in uploader pod
oc exec -it -n ai-performance-profiling model-uploader-nfs -- /bin/bash

# Inside the pod:
pip install huggingface-hub

# Download model (optional: set HF_TOKEN for private models)
python3 -c "
from huggingface_hub import snapshot_download
snapshot_download(
    repo_id='Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice',
    local_dir='/models/Qwen3-TTS-12Hz-1.7B-CustomVoice',
    resume_download=True
)
"

# Verify
ls -lh /models/Qwen3-TTS-12Hz-1.7B-CustomVoice
du -sh /models/Qwen3-TTS-12Hz-1.7B-CustomVoice

# Exit pod
exit
```

## Step 4: Deploy Model Using RHOAI Dashboard


### 4.1 Navigate to your RHOAI project

Within projects in RHOAI navigate to your project, select deployments, select deploy a model

### 4.2 Model Deployment Location

Under model details (first page), select "Cluster Storage", select your created PVC (Model Storage (NFS RWX)), then give the path to the model you downloaded to the pvc.

### 4.3 Verify Deployment

Wait until RHOAI shows the model as successfully deployed


## Troubleshooting

### PVC Stuck in Pending

**Check storage class exists**:
```bash
oc get storageclass nfs-csi
```

**Check PVC events**:
```bash
oc describe pvc model-storage-nfs -n ai-performance-profiling
```

**Common causes**:
- Storage class doesn't exist
- Insufficient storage quota

### PVC Not Visible in RHOAI Dashboard

**Ensure required label exists**:
```bash
oc get pvc model-storage-nfs -n ai-performance-profiling --show-labels
```

Should include: `opendatahub.io/dashboard=true`

**Add label if missing**:
```bash
oc label pvc model-storage-nfs -n ai-performance-profiling opendatahub.io/dashboard=true
```

### Model Upload Fails

**Check uploader pod status**:
```bash
oc get pod model-uploader-nfs -n ai-performance-profiling
oc logs model-uploader-nfs -n ai-performance-profiling
```

**Check PVC mount**:
```bash
oc exec -n ai-performance-profiling model-uploader-nfs -- df -h /models
```

**For large models**, uploads may take time. Monitor progress:
```bash
# In another terminal, watch size increase
watch -n 5 'oc exec -n ai-performance-profiling model-uploader-nfs -- du -sh /models/MODEL_NAME'
```

### Model Deployment Fails

**Check pod logs for errors**:
```bash
oc logs -n ai-performance-profiling -l serving.kserve.io/inferenceservice=MODEL_NAME --tail=200
```

**Common issues**:
1. **Wrong model path**: Ensure path in InferenceService matches actual path in PVC
2. **Missing model files**: Verify all required files uploaded (config.json, model weights, tokenizer)
3. **Resource constraints**: Check if pod has sufficient CPU/memory/GPU
4. **Architecture mismatch**: Ensure serving runtime supports the model architecture

**Verify model files in PVC**:
```bash
oc exec -n ai-performance-profiling model-uploader-nfs -- ls -lh /models/MODEL_NAME/
```

## Best Practices

1. **Use RWX storage classes** (`nfs-csi`, CephFS) for model serving to support multiple inference pods
2. **Exclude git artifacts** when uploading models (`.git`, `.cache`) to save space
3. **Label PVCs** with `opendatahub.io/dashboard: "true"` to make them visible in RHOAI UI
4. **Size PVCs appropriately** - account for multiple models and growth
5. **Keep uploader pod running** during active development for quick model updates
6. **Verify model files** are complete before deployment (config.json, weights, tokenizer files)
7. **Monitor resource usage** - adjust CPU/memory/GPU requests based on actual usage

## Additional Resources

- **RHOAI Documentation**: https://access.redhat.com/documentation/en-us/red_hat_openshift_ai
- **KServe Documentation**: https://kserve.github.io/website/
- **vLLM Documentation**: https://docs.vllm.ai/
- **MOC Cluster Config**: https://github.com/OCP-on-NERC/nerc-ocp-config
