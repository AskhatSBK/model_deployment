# model_deployment

Docker Compose configs for serving open-source LLMs and STT models via [vLLM](https://github.com/vllm-project/vllm) with an OpenAI-compatible API.

## Available models

| Directory | Model | vLLM image |
|---|---|---|
| `gptoss-120b/` | GPT-OSS 120B | `vllm/vllm-openai:v0.13.0` |
| `llama4scout/` | Llama 4 Scout | `vllm/vllm-openai:v0.10.2` |
| `qwen3.5-35b/` | Qwen 3.5 35B A3B | `vllm/vllm-openai:cu130-nightly` |
| `qwen3next-80b/` | Qwen3 Next 80B | `vllm/vllm-openai:v0.16.0` |
| `qwen3rerank/` | Qwen3 0.6B Reranker | `vllm/vllm-openai:v0.14.0` |
| `whisper-v3/` | Whisper v3 (STT) | custom Dockerfile |
| `glm-ocr/` | GLM OCR | `vllm/vllm-openai:v0.16.0` |

---

## Prerequisites

- Docker + Docker Compose v2
- NVIDIA GPU with drivers installed
- [nvidia-container-toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
- A [Hugging Face](https://huggingface.co) account with an access token (for gated models)

---

## Deployment

### 1. Download the model

Use the Hugging Face CLI to pre-download the model weights to a local directory. This avoids re-downloading on every container restart.

```bash
pip install huggingface_hub

# Replace <model-id> with the HF repo, e.g. Qwen/Qwen3-80B
huggingface-cli download <model-id> \
  --token <your-hf-token> \
  --local-dir /data/models/<model-id>
```

Or set `HF_HOME` to your cache directory and let vLLM pull automatically on first start (requires `HUGGING_FACE_HUB_TOKEN` to be set in `.env`).

### 2. Create a `.env` file

Each model directory is self-contained. Copy the template below into the relevant directory as `.env` and fill in the values.

```dotenv
# Hugging Face token (required for gated/private models)
HUGGING_FACE_HUB_TOKEN=hf_...

# HF model ID as it appears on huggingface.co, e.g. Qwen/Qwen3-80B
MODEL_ID=org/model-name

# Alias exposed by the OpenAI-compatible API
MODEL_NAME=my-model

# Local path where model weights are stored (maps to /root/.cache/huggingface or /model_cache)
HF_HOME=/data/models

# Comma-separated GPU indices to use, e.g. 0 or 0,1
GPU_DEVICE_IDS=0

# Host port that the API will be available on
PORT=8000

# --- Optional tuning (defaults shown) ---
GPU_MEMORY_UTILIZATION=0.95
TENSOR_PARALLEL_SIZE=1
MAX_MODEL_LEN=8192
DTYPE=auto
```

> **gptoss-120b** also caches tiktoken data. The compose file hard-codes
> `/data/models/harmony_cache` for that — create the directory or adjust the
> volume mount in `docker-compose.yml` before starting.

### 3. Start the service

```bash
cd <model-directory>          # e.g. cd qwen3next-80b
docker compose up -d
```

The API will be available at `http://localhost:<PORT>/v1`.

### 4. Verify

```bash
curl http://localhost:<PORT>/v1/models
```

Expected response lists the `MODEL_NAME` you configured.

---

## Per-model notes

### whisper-v3 (STT)

Builds a custom image that adds `vllm[audio]` support:

```bash
cd whisper-v3
docker compose up -d --build
```

### qwen3rerank (reranker)

Uses the `pooling` runner. Set `MODEL_ID` to the reranker checkpoint, e.g.
`Qwen/Qwen3-Reranker-0.6B`. `--trust-remote-code` is already enabled in the
compose file.

### qwen3next-80b

`MAX_MODEL_LEN` is fixed at `160000` in the compose file. You can override it
by editing `docker-compose.yml` directly.

### glm-ocr

Uses MTP speculative decoding (`num_speculative_tokens: 1`). No extra
configuration needed beyond the standard `.env`.

---

## Stopping a service

```bash
cd <model-directory>
docker compose down
```

To also remove the container image:

```bash
docker compose down --rmi all
```
