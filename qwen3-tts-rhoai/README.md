# Deploying Qwen3-TTS Models on RHOAI

## Prerequisites

- `oc` CLI logged into the cluster
- `huggingface-cli` installed (`pip install huggingface-hub`)

## Background

Researchers in the MOC needed to deploy the Qwen3-TTS model, which required a vLLM-omni ServingRuntime in RHOAI. An [existing ServingRuntime by @vraiti](https://github.com/vraiti/omni-kserve/blob/e8a5dd6dd9216977f2cc122e9c334ec605186174/01-servingruntime.yaml) assumed models would be pulled directly from HuggingFace. However, we typically deploy models from S3 storage because researchers fine-tune model weights, save changes back to S3, and redeploy. So the ServingRuntime needed modifications to support S3 based model deployment. Another option is to [containerize the model and deploy it via modelcar](https://github.com/cbtham/rhoai-genai-workshop?tab=readme-ov-file#11-option-1-using-a-pre-built-llm-container-faster), but that requires rebuilding and pushing a new image after every fine-tuning iteration, which doesn't fit the researcher workflow.

### Why this ServingRuntime is Qwen3-TTS-specific

At a high level, the ServingRuntime serves whatever model is in `/mnt/models` using a single image (`vllm/vllm-omni`) and a single command pattern (`vllm serve <path> --omni`). Qwen3-TTS requires a "task type" to determine which generation method to use. There are three task types:

| Task type | Generation method | Use case |
|---|---|---|
| **CustomVoice** | `generate_custom_voice()` | Predefined speakers (e.g. "vivian") |
| **VoiceDesign** | `generate_voice_design()` | Design a voice from instructions |
| **Base** | `generate_voice_clone()` | Clone from reference audio |

The `--task-type` flag on `vllm serve` specifies which task type to use. The ServingRuntime sets this via the `TASK_TYPE` environment variable, which defaults to `CustomVoice`. To deploy a different variant, override `TASK_TYPE` in the InferenceService — no separate ServingRuntime is needed (see section 3).


## Current Limitations

This ServingRuntime supports all Qwen3-TTS variants (CustomVoice, VoiceDesign, Base) via the `TASK_TYPE` env var, which can be overridden per InferenceService deployment (see section 3). Other vLLM-omni models have not been tested with this ServingRuntime.

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

The vLLM-Omni ServingRuntime (`vllm-qwen3-tts-omni-runtime`) is already applied to the cluster. It uses the `TASK_TYPE` environment variable to pass the `--task-type` flag to `vllm serve`, which determines which Qwen3-TTS generation method to use. The default is `CustomVoice`, but this can be overridden per InferenceService deployment. One ServingRuntime supports all Qwen3-TTS task types.

The ServingRuntime definition:

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  annotations:
    opendatahub.io/apiProtocol: REST
    opendatahub.io/recommended-accelerators: '["nvidia.com/gpu"]'
    opendatahub.io/runtime-version: v0.18.0
    openshift.io/display-name: vLLM-Omni Qwen3-TTS
  labels:
    opendatahub.io/dashboard: "true"
  name: vllm-qwen3-tts-omni-runtime
spec:
  annotations:
    prometheus.io/path: /metrics
    prometheus.io/port: "8000"
  containers:
    - args:
        - |
          exec vllm serve /mnt/models --omni --task-type ${TASK_TYPE} \
            --host 0.0.0.0 --port 8000 --served-model-name={{.Name}} \
            --trust-remote-code --enforce-eager ${EXTRA_ARGS}
      command:
        - /bin/sh
        - -c
      env:
        - name: TASK_TYPE
          value: CustomVoice
        - name: EXTRA_ARGS
          value: ""
        - name: HOME
          value: /tmp
        - name: HF_HOME
          value: /tmp/hf_home
        - name: NUMBA_CACHE_DIR
          value: /tmp/numba_cache
        - name: MPLCONFIGDIR
          value: /tmp/matplotlib
      image: vllm/vllm-omni:v0.18.0
      name: kserve-container
      ports:
        - containerPort: 8000
          name: http
          protocol: TCP
      resources:
        limits:
          cpu: "8"
          memory: 32Gi
          nvidia.com/gpu: "1"
        requests:
          cpu: "4"
          memory: 16Gi
          nvidia.com/gpu: "1"
      volumeMounts:
        - mountPath: /dev/shm
          name: shm
  multiModel: false
  supportedModelFormats:
    - autoSelect: true
      name: vllm-omni
  volumes:
    - emptyDir:
        medium: Memory
        sizeLimit: 8Gi
      name: shm
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
   - **Source model location**: Select the Minio Data Connection, then set the path to your model folder (e.g. `Qwen3-TTS-12Hz-1.7B-CustomVoice` for the CustomVoice weights from [section 2.1](#21-download-the-model))
   - **Configuration parameters**: The ServingRuntime defaults to `TASK_TYPE=CustomVoice`. If you are deploying a different Qwen3-TTS variant (VoiceDesign or Base weights, or any checkpoint whose task type is not CustomVoice), add an environment variable **`TASK_TYPE`** with the matching value: `CustomVoice`, `VoiceDesign`, or `Base` (see the task type table in [Background](#background)). Values set here override the runtime default. You can also set **`EXTRA_ARGS`** here for specific vLLM flags (e.g. `--max-model-len 4096`).
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
| `Invalid task type` | The `TASK_TYPE` env var is not set to a valid value. Must be one of `CustomVoice`, `VoiceDesign`, or `Base`. Check both the ServingRuntime default and any InferenceService override. |
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
