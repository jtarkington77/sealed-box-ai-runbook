# Stack setup -- worker, watchdog, vector store, UI

This doc is the **"meat and bones"** setup for your sealed-box stack:

- Worker model (your main LLM).
- Watchdog model (smaller, used later for safety/monitoring).
- Vector store for retrieval (RAG).
- A real UI -- we'll use **Open WebUI** as the front-end and light orchestrator.

You will end up with:

- All services running as containers on your `ai-box`.
- Everything reachable **only on localhost**:
  - Open WebUI on `http://127.0.0.1:3000`
  - Worker API on `http://127.0.0.1:8000`
  - Watchdog API on `http://127.0.0.1:8100`
  - Qdrant on `http://127.0.0.1:6333`
- No WAN exposure, no TLS, no tunnels here.  
  (Those live in `safety-and-isolation.md`.)

---

## 0. Starting point

We assume:

- You're on a **Linux host or VM** (we'll call it `ai-box`).
- You have an **NVIDIA GPU** and drivers installed.
- **Docker** is installed and running.
- **NVIDIA Container Toolkit** is installed so containers can see the GPU.
- You're comfortable editing text files and running commands.

If any of that is missing, fix it first using the official docs for your distro.

Pick a working directory for the stack. In this doc we'll use:

```bash
mkdir -p ~/sealed-box-ai/stack
cd ~/sealed-box-ai/stack
```

If you prefer a different path, adjust commands accordingly.

This doc expects you already read:

- `README.md` -- what this project is and who it's for.
- `docs/models.md` -- how to choose worker/watchdog models based on your hardware.
- `docs/safety-and-isolation.md` -- where network hardening and reverse proxy live.

Nothing in this file exposes ports to the WAN.  
We bind services to `127.0.0.1` only.

---

## 1. GPU sanity check

Before you touch models, make sure the GPU path actually works.

### 1.1 Check from the host

On `ai-box`:

```bash
nvidia-smi
```

You should see:

- Your GPU name (e.g. 4090, A6000, etc.).
- Driver version.
- Basic utilization info.

If this fails, fix your driver setup before continuing.

### 1.2 Check from a container

Now confirm Docker containers can see the GPU:

```bash
docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
```

You should see similar output to the host `nvidia-smi`, but running **inside** a container.

If this fails:

- Re-check that `nvidia-container-toolkit` is installed.
- Make sure Docker is configured to use the NVIDIA runtime.
- Reboot if you just installed drivers/toolkit.

Do **not** move on until both checks work.

---

## 2. Folder layout for the stack

Inside `~/sealed-box-ai/stack`, create a basic layout:

```bash
cd ~/sealed-box-ai/stack
mkdir -p env data/qdrant data/open-webui data/models logs
```

We'll keep things simple:

- `docker-compose.yml` -- main stack definition.
- `env/` -- env files for configuration.
- `data/qdrant/` -- persistent storage for the vector store.
- `data/open-webui/` -- Open WebUI config and history.
- `data/models/` -- optional cache location for downloaded models.
- `logs/` -- logs you might want to examine later.

---

## 3. Decide your models (high level)

This file does **not** re-teach models. That's what `docs/models.md` is for.

For this setup we assume you have already chosen:

- A **worker model** (general assistant) that fits your GPU.
- A **watchdog model** (small instruct model) for scoring/monitoring.
- An **embedding model** for RAG (used by your ingestion pipeline / tools).

You'll plug the worker and watchdog into env vars shortly.  
If you haven't decided yet, read `docs/models.md` first, then come back here.

---

## 4. Docker Compose -- first version

Create `docker-compose.yml` in `~/sealed-box-ai/stack` with the following content:

```yaml
version: "3.9"

services:
  # Main worker LLM -- your "assistant" model
  worker-llm:
    image: vllm/vllm-openai:0.4.2  # pin a known-good version
    container_name: sealed-box-worker
    restart: unless-stopped
    env_file:
      - ./env/stack.env
    command: >
      --model ${WORKER_MODEL}
      --host 0.0.0.0
      --port 8000
      --max-model-len ${WORKER_MAX_CONTEXT}
      --gpu-memory-utilization 0.9
      --enforce-eager
    ports:
      - "127.0.0.1:8000:8000"          # OpenAI-compatible API, local only
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: ["gpu"]
    volumes:
      - ./data/models:/root/.cache/huggingface

  # Watchdog LLM -- small model we'll wire up later for safety/monitoring
  watchdog-llm:
    image: vllm/vllm-openai:0.4.2  # same version as worker
    container_name: sealed-box-watchdog
    restart: unless-stopped
    env_file:
      - ./env/stack.env
    command: >
      --model ${WATCHDOG_MODEL}
      --host 0.0.0.0
      --port 8100
      --max-model-len ${WATCHDOG_MAX_CONTEXT}
      --gpu-memory-utilization 0.3
      --enforce-eager
    ports:
      - "127.0.0.1:8100:8100"          # local only; not exposed to LAN or WAN
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: ["gpu"]
    volumes:
      - ./data/models:/root/.cache/huggingface
    depends_on:
      - worker-llm

  # Vector store -- Qdrant for RAG
  qdrant:
    image: qdrant/qdrant:1.13.3  # pin a stable release
    container_name: sealed-box-qdrant
    restart: unless-stopped
    environment:
      QDRANT__SERVICE__GRPC_PORT: 6334
      QDRANT__SERVICE__HTTP_PORT: 6333
    ports:
      - "127.0.0.1:6333:6333"
      - "127.0.0.1:6334:6334"
    volumes:
      - ./data/qdrant:/qdrant/storage

  # Open WebUI -- UI + light orchestrator
  open-webui:
    image: ghcr.io/open-webui/open-webui:v0.3.18  # pin a known-good release
    container_name: sealed-box-ui
    restart: unless-stopped
    environment:
      # Tell Open WebUI to talk to the worker using OpenAI-compatible API
      OPENAI_API_BASE_URL: http://worker-llm:8000/v1
      OPENAI_API_KEY: local-sealed-box
      # You can tune more env vars later (storage, auth, etc.)
    ports:
      - "127.0.0.1:3000:8080"          # Web UI, local only
    depends_on:
      - worker-llm
      - qdrant
    volumes:
      - ./data/open-webui:/app/backend/data
```

Notes:

- **Worker / watchdog** use `vllm/vllm-openai` as an example engine.  
  If you prefer another runtime (Ollama, llama.cpp, etc.), swap the image/command accordingly.
- All ports are bound to `127.0.0.1`:
  - Usable from the `ai-box` host.
  - Not directly visible on the LAN/WAN.
- Open WebUI uses the worker's **OpenAI-compatible API** at `http://worker-llm:8000/v1`.

We haven't wired the watchdog into your pipelines yet.  
That's done in `docs/watchdog-monitoring.md`. For now it just needs to run and respond.

---

## 5. Environment variables

Create `env/stack.env` to hold model choices and basic settings:

```bash
cd ~/sealed-box-ai/stack
cat > env/stack.env << 'EOF'
# Worker model (general assistant)
WORKER_MODEL=meta-llama/Meta-Llama-3-70B-Instruct
WORKER_MAX_CONTEXT=8192

# Watchdog model (small, fast)
WATCHDOG_MODEL=microsoft/Phi-3.5-mini-instruct
WATCHDOG_MAX_CONTEXT=4096
EOF
```

Adjust:

- `WORKER_MODEL` and `WATCHDOG_MODEL` to match what you picked from `docs/models.md`.
- `WORKER_MAX_CONTEXT` and `WATCHDOG_MAX_CONTEXT` to realistic values for your GPU and engine.
- If you're offline/air-gapped, ensure these models (and embeddings) are already cached in `data/models` or your runtime's cache path before you start compose.

If you later add more env vars (for example embedding model IDs, or routing hints), keep them in this file so you only restart containers, not edit compose YAML constantly.

---

## 6. Bring the stack up

From `~/sealed-box-ai/stack`:

```bash
docker compose --env-file ./env/stack.env up -d
```

Then check:

```bash
docker ps
```

You should see at least:

- `sealed-box-worker`
- `sealed-box-watchdog`
- `sealed-box-qdrant`
- `sealed-box-ui`

If something is `Restarting` or `Exited`:

```bash
docker logs sealed-box-worker
docker logs sealed-box-watchdog
docker logs sealed-box-qdrant
docker logs sealed-box-ui
```

Fix obvious issues (bad model ID, no GPU, missing env vars) before moving on.

---

## 6.1 Extending compose to include agents

When you build the agents from `docs/agents-and-tools.md`, add them under the same `services:` block in `docker-compose.yml` and keep them on the `sealed_ai_net` network. Example:

```yaml
  internet-research:
    build: ../agents/internet-research
    environment:
      LOG_PATH: /logs/agents/internet-research.log
    volumes:
      - ./logs/agents:/logs/agents
    networks:
      - sealed_ai_net
    restart: unless-stopped

  intel-sync:
    build: ../agents/intel-sync
    environment:
      QDRANT_URL: http://qdrant:6333
      QDRANT_COLLECTION: intel
      EMBED_MODEL: BAAI/bge-m3
    volumes:
      - ./logs/agents:/logs/agents
      - ./data/ingest/intel:/data/ingest/intel
    networks:
      - sealed_ai_net
    restart: unless-stopped
    entrypoint: ["sh","-c","while true; do python /app/sync.py || true; sleep 3600; done"]
```

Run `docker compose build` then `docker compose up -d` from `~/sealed-box-ai/stack` after adding these.

---

## 7. Quick local tests

### 7.1 Open WebUI

On `ai-box` (or via SSH with port forwarding), open:

```text
http://127.0.0.1:3000
```

You should see Open WebUI's setup flow.

- Create an admin user.
- In settings, confirm it's using:
  - Backend: **OpenAI compatible**
  - Base URL: `http://worker-llm:8000/v1`
  - API key: `local-sealed-box` (or whatever you set in env).

Then:

- Open a chat.
- Ask something simple that doesn't require your own data yet.
- Confirm latency/quality feel roughly like what you expect for your worker model.

### 7.2 Worker API directly

From `ai-box`:

```bash
curl http://127.0.0.1:8000/v1/models
```

You should get back a JSON listing the model(s) that vLLM is serving.

Try a simple completion (replace the model ID with your worker):

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer local-sealed-box" \
  -d '{
    "model": "meta-llama/Meta-Llama-3-70B-Instruct",
    "messages": [
      {"role": "system", "content": "You are my local sealed-box assistant."},
      {"role": "user", "content": "In one sentence, explain what this sealed box is."}
    ]
  }'
```

You should get an answer back.  
If you don't, check `docker logs sealed-box-worker`.

### 7.3 Watchdog ping

We're not using the watchdog in the pipeline yet, but verify it's alive:

```bash
curl http://127.0.0.1:8100/v1/models
```

If that returns a model list, the watchdog container is running correctly.

---

## 8. Where this connects to the other docs

At this point you have:

- A local worker LLM.
- A local watchdog LLM.
- A vector store.
- A UI that talks to the worker.

From here:

- **Models & hardware details** live in:  
  `docs/models.md`
- **Network isolation, firewall, Caddy, tunnels, Cloudflare, etc.** live in:  
  `docs/safety-and-isolation.md`
- **Agents and example workflows** (including internet research via a separate agent, not by giving the worker raw internet access) live in:  
  `docs/agents-and-tools.md`
- **Watchdog wiring and monitoring** live in:  
  `docs/watchdog-monitoring.md`
- **Using the stack day-to-day** lives in:  
  `docs/usage.md`
- **Training/routines/RAG** live in:  
  `docs/training-and-routines.md`
- **Operations and ongoing maintenance** live in:  
  `docs/operations.md`

This file's only job was to get the core stack running locally.




