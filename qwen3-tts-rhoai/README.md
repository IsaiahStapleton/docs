# Deploying Qwen3-TTS-12Hz-1.7B-CustomVoice on RHOAI

## Prerequisites

- `oc` CLI logged into the cluster
- `huggingface-cli` installed (`pip install huggingface-hub`)

## Background

Researchers in the MOC needed to deploy the Qwen3-TTS model, which required a vLLM-omni ServingRuntime in RHOAI. An [existing ServingRuntime by @vraiti](https://github.com/vraiti/omni-kserve/blob/e8a5dd6dd9216977f2cc122e9c334ec605186174/01-servingruntime.yaml) assumed models would be pulled directly from HuggingFace. However, we typically deploy models from S3 storage because researchers fine-tune model weights, save changes back to S3, and redeploy. So the ServingRuntime needed modifications to support S3 based model deployment. Another option is to [containerize the model and deploy it via modelcar](https://github.com/cbtham/rhoai-genai-workshop?tab=readme-ov-file#11-option-1-using-a-pre-built-llm-container-faster), but that requires rebuilding and pushing a new image after every fine-tuning iteration, which doesn't fit the researcher workflow.


### Why this ServingRuntime is Qwen3-TTS-specific

At a high level, the ServingRuntime serves whatever model is in `/mnt/models` using a single image (`vllm/vllm-omni`) and a single command pattern (`vllm serve <path> --omni`). However, you cannot simply run `vllm serve /mnt/models --omni` for Qwen3-TTS because vLLM-omni's [Qwen3-TTS-specific code](https://github.com/vllm-project/vllm-omni/blob/main/vllm_omni/model_executor/models/qwen3_tts/qwen3_tts.py) infers a "task type" from the model path. For Qwen3-TTS there are three task types that map to three different generation methods:

| Task type | Generation method | Use case |
|---|---|---|
| **CustomVoice** | `generate_custom_voice()` | Predefined speakers (e.g. "vivian") |
| **VoiceDesign** | `generate_voice_design()` | Design a voice from instructions |
| **Base** | `generate_voice_clone()` | Clone from reference audio |

So for a path like `.../Qwen3-TTS-12Hz-1.7B-CustomVoice`, it grabs `CustomVoice`. If you pass `/mnt/models`, it produces `mnt` or `models` and fails with `Invalid task type: mnt/models`.

The workaround is a symlink: create `/tmp/Qwen/${HF_MODEL_NAME}` pointing to `/mnt/models`, then run `vllm serve /tmp/Qwen/${HF_MODEL_NAME}`. That way vLLM-omni gets the model name string (to infer task type) while the OS follows the symlink to read the actual weights from `/mnt/models`.

### Toward a universal vLLM-omni ServingRuntime

In an ideal world, we would have a single "vLLM-omni" ServingRuntime that supports deployments from S3, HuggingFace, or sidecar for any multi-modal model, similar to the default vLLM ServingRuntime for KServe that ships with RHOAI.

This is not straightforward today because within vLLM-omni, each supported model has its own specific code (under `vllm_omni/model_executor/models/`) and each model's code may make different assumptions about path structure, config format, or expected arguments. For example:

- **Qwen3-TTS** infers task type from the model path and expects a HuggingFace-style path name.
- **GLM-TTS** does not even have a `config.json` at the root level of the model.
- Other omni models (CosyVoice3, FLUX, Qwen3-Omni, etc.) have their own pipelines and assumptions.

We plan to submit an RFC to upstream [vllm-project/vllm-omni](https://github.com/vllm-project/vllm-omni) to standardize the expected repository structure.

## Current Limitations

This ServingRuntime can only be used to deploy Qwen3-TTS variants (CustomVoice, VoiceDesign, Base). To deploy a different variant, change the `HF_MODEL_NAME` env var in the ServingRuntime as outlined in section 3. Other vLLM-omni models may work without the symlink hack but have not been tested.

## 1. Deploying Minio S3 Storage

Script to deploy S3 Storage: https://github.com/IsaiahStapleton/rhoai-model-deployment-guide/blob/main/minio-setup.yaml

Change lines 21 and 22 to change default user and pass.

Deploy Minio in your Project namespace:

```bash
oc apply -f minio-setup.yaml -n <your-namespace>
```

Verify the pod is running:

```bash
oc get pods -n <your-namespace> | grep minio
```

Get the Minio route URLs:

```bash
oc get routes -n <your-namespace> | grep minio
```

Two routes will be created:
- **minio-api** — programmatic S3 access
- **minio-ui** — browser-based UI

### 1.1 Create a Bucket

1. Open the **minio-ui** route URL in your browser
2. Log in with the credentials from the Minio deployment (default: `minio` / `minio123`)
3. Go to **Buckets** and select **Create Bucket**
4. Name it `models` and click **Create Bucket**

### 1.2 Create a Data Connection in RHOAI

1. In the RHOAI Dashboard, navigate to your Data Science Project
2. Go to **Connections** and select **Create connection**
3. Fill in the following:
   - **Connection name**: e.g., `minio-models`
   - **Access key**: `minio` (or whatever you set minio_root_user in `minio-setup.yaml`)
   - **Secret key**: `minio123` (or whatever you set minio_root_password in `minio-setup.yaml`)
   - **Endpoint**: the **minio-api** route URL, NOT the minio-ui route
   - **Bucket**: `models`

## 2. Downloading the Model and Uploading to S3

### 2.1 Download the Model

```bash
huggingface-cli download Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice \
    --local-dir ./Qwen3-TTS-12Hz-1.7B-CustomVoice
```

This downloads approximately 3.5 GB.

### 2.2 Upload to S3

1. Open the Minio UI route in your browser
2. Navigate to the `models` bucket
3. Select **Upload > Upload Folder** and choose the downloaded model directory
4. Wait for the upload to complete

## 3. ServingRuntime

The vLLM-Omni ServingRuntime (`vllm-qwen3-tts-omni-runtime`) is already applied to the cluster. It uses the `HF_MODEL_NAME` environment variable to determine which Qwen3-TTS variant to load.

If you want to deploy a different variant, create a new ServingRuntime with a different `HF_MODEL_NAME` value:

```bash
oc get servingruntime vllm-qwen3-tts-omni-runtime -n <your-namespace> -o yaml > my-servingruntime.yaml
```

The ServingRuntime definition is also here:

```
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  name: vllm-qwen3-tts-omni-runtime
  annotations:
    openshift.io/display-name: vLLM-Omni Qwen3-TTS-12Hz-1.7B-CustomVoice
    opendatahub.io/recommended-accelerators: '["nvidia.com/gpu"]'
    opendatahub.io/apiProtocol: REST
spec:
  annotations:
    prometheus.io/path: /metrics
    prometheus.io/port: "8000"
  supportedModelFormats:
    - name: vllm-omni
      autoSelect: true
  containers:
    - name: kserve-container
      image: vllm/vllm-omni:v0.14.0
      command:
        - /bin/sh
        - -c
      args:
        - >
          mkdir -p /tmp/Qwen && ln -sf /mnt/models /tmp/Qwen/${HF_MODEL_NAME} &&
          exec vllm serve /tmp/Qwen/${HF_MODEL_NAME} --omni --host 0.0.0.0
          --port 8000 --served-model-name={{.Name}} --trust-remote-code
          --enforce-eager
      env:
        - name: HF_MODEL_NAME
          value: Qwen3-TTS-12Hz-1.7B-CustomVoice
        - name: HOME
          value: /tmp
        - name: HF_HOME
          value: /tmp/hf_home
        - name: NUMBA_CACHE_DIR
          value: /tmp/numba_cache
        - name: MPLCONFIGDIR
          value: /tmp/matplotlib
      ports:
        - name: http
          containerPort: 8000
          protocol: TCP
      resources:
        requests:
          memory: 16Gi
          cpu: "4"
          nvidia.com/gpu: "1"
        limits:
          memory: 32Gi
          cpu: "8"
          nvidia.com/gpu: "1"
      volumeMounts:
        - name: shm
          mountPath: /dev/shm
  volumes:
    - name: shm
      emptyDir:
        medium: Memory
        sizeLimit: 8Gi
  multiModel: false

```

Edit the `HF_MODEL_NAME` env var to match the variant you uploaded to S3:

```yaml
- name: HF_MODEL_NAME
  value: Qwen3-TTS-12Hz-1.7B-VoiceDesign
```

Then apply the new ServingRuntime:

```bash
oc apply -f my-servingruntime.yaml -n <your-namespace>
```

## 4. Deploying the Model on RHOAI

1. In the RHOAI Dashboard, go to **Models** in your Data Science Project
2. Select **Single-model serving**, then **Deploy model**
3. Fill in the following:
   - **Model deployment name**: e.g., `qwen3-tts`
   - **Serving runtime**: **vLLM-Omni Runtime**
   - **Model server size**: Small is fine for 1.7B
   - **Accelerator**: NVIDIA GPU
   - **Model route**: Check **Make deployed models available through an external route**
   - **Token authentication**: Check **Require token authentication**
   - **Source model location**: Select the Minio Data Connection, then set the path to `Qwen3-TTS-12Hz-1.7B-CustomVoice`
4. Click **Deploy**

The model will take a few minutes to load. Once the pod status shows **Ready**, proceed to testing.

## 5. Testing the Deployment

Get the **Inference endpoint** URL and **Authorization token** from the RHOAI Dashboard under **Models**.

```bash
export INFERENCE_URL="https://<your-model-endpoint>"
export TOKEN="<your-bearer-token>"
```

### List Available Voices

```bash
curl -k "${INFERENCE_URL}/v1/audio/voices" \
  -H "Authorization: Bearer ${TOKEN}"
```

### Generate Speech

```bash
curl -k -X POST "${INFERENCE_URL}/v1/audio/speech" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${TOKEN}" \
  -d '{
    "input": "Hello from OpenShift AI!",
    "voice": "vivian",
    "response_format": "wav"
  }' \
  --output output.wav
```

Play the resulting audio (linux):

```bash
aplay output.wav
```

## Troubleshooting

| Issue | Fix |
|---|---|
| `Invalid task type: mnt/models` | The `HF_MODEL_NAME` env var in the ServingRuntime doesn't match the model uploaded to S3. |
| Model pod OOMKilled | Increase memory limits. The 1.7B model needs ~8 GB RAM + GPU VRAM. |
| Pod terminates shortly after starting | Check logs with `oc logs <pod-name> -n <namespace>`. Verify the S3 path matches the folder name in your bucket. |
| 502 / timeout on first request | The model is still loading. First inference can take 30-60 seconds. |
| `numba` caching errors | The `NUMBA_CACHE_DIR` env var is missing from the ServingRuntime. |
| `PermissionError: '/.config'` | The `HOME` env var must be set to `/tmp` in the ServingRuntime. |

## References

- [Qwen3-TTS on HuggingFace](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice)
- [vLLM-Omni Qwen3-TTS Documentation](https://docs.vllm.ai/projects/vllm-omni/en/latest/user_guide/examples/online_serving/qwen3_tts/)
- [vLLM-Omni Supported Models](https://docs.vllm.ai/projects/vllm-omni/en/latest/models/supported_models/)
- [vLLM-Omni GitHub](https://github.com/vllm-project/vllm-omni)
- [vLLM-Omni Qwen3-TTS source (task type logic)](https://github.com/vllm-project/vllm-omni/blob/main/vllm_omni/model_executor/models/qwen3_tts/qwen3_tts.py)
- [@vraiti's omni-kserve ServingRuntime](https://github.com/vraiti/omni-kserve/blob/e8a5dd6dd9216977f2cc122e9c334ec605186174/01-servingruntime.yaml)
- [Guide to Deploying AI Models on RHOAI](https://github.com/IsaiahStapleton/rhoai-model-deployment-guide)
