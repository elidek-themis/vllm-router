# vllm-router

A lightweight FastAPI reverse proxy that auto-discovers running [vLLM](https://github.com/vllm-project/vllm) servers on a configurable port range and routes incoming OpenAI-compatible requests to the correct backend based on the requested model.

## Overview

`vllm-router` sits in front of one or more vLLM instances, each serving a different model on a dedicated port. It periodically scans the configured port range, builds an in-memory model registry, and transparently forwards requests to whichever backend hosts the requested model.

```
Client ──► vllm-router :8000 ──► vLLM (model-A) :8001
                             ──► vLLM (model-B) :8002
                             ──► vLLM (model-C) :8003
```

## Features

- **Auto-discovery** — polls each port in the configured range every 10 seconds and registers healthy vLLM backends automatically.
- **OpenAI-compatible routing** — forwards any `/v1/*` request to the correct backend based on the `model` field in the request body.
- **Aggregated model list** — `GET /v1/models` returns the union of all models served across all discovered backends.
- **Health endpoint** — `GET /health` triggers an immediate refresh and reports the number of active backends.
- **Manual refresh** — `POST /refresh` forces an on-demand re-discovery cycle.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/` | Status message and active server count |
| `GET` | `/health` | Refresh registry and return health status |
| `POST` | `/refresh` | Manually trigger backend re-discovery |
| `GET` | `/v1/models` | Aggregated list of all available models |
| `GET\|POST\|HEAD` | `/{path}` | Proxy to the backend serving the requested model |

## Usage

```bash
route [--host HOST] [--port PORT] [--vllm-port-start N] [--vllm-port-end N]
```

| Argument | Default | Description |
|----------|---------|-------------|
| `--host` | `0.0.0.0` | Host to bind the router to |
| `--port` | `8000` | Port for the router |
| `--vllm-port-start` | `8001` | First port in the vLLM discovery range |
| `--vllm-port-end` | `8010` | Last port in the vLLM discovery range (inclusive) |

### Example

Start two vLLM backends on ports 8001 and 8002, then launch the router:

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct --port 8001
vllm serve google/gemma-2-27b-it            --port 8002

route --port 8000 --vllm-port-start 8001 --vllm-port-end 8002
```

Route a request to a specific model:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.1-8B-Instruct",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

## Requirements

- Python ≥ 3.11
- `vllm >= 0.17.1`

Install dependencies:

```bash
uv sync
```
