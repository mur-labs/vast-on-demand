# Rental LLM's — Vast.ai Lab

On-demand `llama.cpp` inference container and zero-dependency Python Orchestrator for running open-weight LLMs on Vast.ai spot/on-demand GPUs with automatic scale-to-zero.

## Architecture

```
Client / LiteLLM ──▶ Python Orchestrator (:8088) ──(dynamic port / SSH tunnel)──▶ Vast.ai GPU Instance
```

When a request hits `:8088`, the orchestrator rents the cheapest qualifying GPU on Vast.ai, injects an ephemeral `API_TOKEN`, waits for the model to load, proxies the OpenAI-compatible traffic, and destroys the instance after 10 minutes of idle time.

---

## Quick Start

1. **Configure your Vast.ai API key** (no Python dependencies required—uses stdlib only):
   ```bash
   cp .env.example .env
   # Edit .env and set VAST_API_KEY=...
   ```

2. **Start the orchestrator**:
   ```bash
   ./start-orchestrator.sh
   ```
   Useful flags:
   - `--mode on-demand` (default is `spot`)
   - `--idle-timeout 600` (seconds before destroying an idle GPU)
   - `--model-url <url>` (override GGUF download URL)
   - `--image <image>` (override container image)

3. **Send a request** (the first request provisions the GPU and may take a few minutes to download weights):
   ```bash
   curl http://127.0.0.1:8088/v1/chat/completions \
     -H "Content-Type: application/json" \
     -d '{"model": "qwen2.5-14b-instruct", "messages": [{"role": "user", "content": "Hello!"}]}'
   ```
   Check current state anytime via `GET http://127.0.0.1:8088/status`.

4. **Optional: Forking & Building Your Own Image**
   The CLI defaults to `ghcr.io/mur-labs/rental-llm:latest`, which is built transparently from [`Dockerfile`](Dockerfile) and [`entrypoint.sh`](entrypoint.sh) with no baked-in secrets or model weights (`MODEL_URL`, `API_TOKEN`, and `HF_TOKEN` are injected at runtime). If you fork this repo, [`.github/workflows/docker-build-push.yml`](.github/workflows/docker-build-push.yml) builds and pushes `ghcr.io/<your-user>/rental-llm:latest` automatically on push to `main`—no local Docker required. Pass `--image ghcr.io/<your-user>/rental-llm:latest` or set `RENTAL_DOCKER_IMAGE` in `.env`.

---

## Components

### 1. Custom Inference Container
* [`Dockerfile`](Dockerfile): Base image `ghcr.io/ggml-org/llama.cpp:server-cuda` with `aria2`, `curl`, and `ca-certificates`.
* [`entrypoint.sh`](entrypoint.sh):
  * Enforces `MODEL_URL` and `API_TOKEN` environment variables.
  * Downloads model weights on boot into `/tmp` (ephemeral disk) using `aria2c` (16 concurrent connections).
  * Launches `llama-server` binding to `0.0.0.0:8000` with Bearer token authentication via `--api-key`.

### 2. Python Orchestrator (`orchestrator/`)
* [`orchestrator/config.py`](orchestrator/config.py): Defines hardware criteria ($\ge 20$ GB VRAM, $\ge 40$ GB disk, $\ge 500$ Mbps download, $\ge 97\%$ reliability), default model (`Qwen2.5-14B-Instruct` Q4_K_M), and 10-minute idle destruction timeout.
* [`orchestrator/vast_client.py`](orchestrator/vast_client.py): REST API client for Vast.ai API v0/v1. Filters offers by lowest total task cost, rents instances with ephemeral credentials, queries status, resolves dynamic host ports, and destroys instances.
* [`orchestrator/instance_manager.py`](orchestrator/instance_manager.py): Thread-safe state machine (`STOPPED`, `PROVISIONING`, `READY`, `DESTROYING`). Deduplicates concurrent requests, generates zero-trust ephemeral tokens, runs health checks, and operates the background idle reaper.
* [`orchestrator/proxy_server.py`](orchestrator/proxy_server.py): OpenAI-compatible reverse proxy that lazily triggers GPU provisioning and streams responses from the remote endpoint.
* [`orchestrator/main.py`](orchestrator/main.py): CLI entrypoint.

---

## Test Suite

All components are tested locally using Python's standard library without requiring a local Docker daemon or active Vast.ai billing:

```bash
python3 -m unittest discover -s tests -v
```

### Test Coverage
* [`tests/test_entrypoint.py`](tests/test_entrypoint.py): Subprocess testing of environment validation, download handling, Hugging Face token passing, and `llama-server` CLI forwarding.
* [`tests/test_vast_client.py`](tests/test_vast_client.py): API request structure, minimum hardware filtering, cost sorting, dynamic port resolution, and destruction calls.
* [`tests/test_instance_manager.py`](tests/test_instance_manager.py): Concurrent request deduplication, reuse, preemption handling, failure recovery, and idle auto-destruction.
* [`tests/test_proxy_server.py`](tests/test_proxy_server.py): Lazy provisioning, proxy routing, Authorization token injection, error handling, and `/status` monitoring.
