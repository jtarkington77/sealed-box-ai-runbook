# Build Sheet -- first full setup (step by step)

This is a linear checklist you can follow from a clean host. Assumes basic Linux skills, Docker, NVIDIA drivers, and enough VRAM for your chosen models. Commands are Ubuntu-flavored.

## 0) Paths and prerequisites (one clear path)

- Use `~/sealed-box-ai/` as the base (adjust if you prefer).
- Install Docker on Ubuntu:
  ```bash
  curl -fsSL https://get.docker.com | sudo sh
  sudo usermod -aG docker $USER
  newgrp docker
  ```
  For issues, see https://docs.docker.com/engine/install/ubuntu/ and https://docs.docker.com/engine/install/linux-postinstall/
- Install NVIDIA drivers + toolkit (Ubuntu):
  ```bash
  sudo apt update && sudo apt install -y ubuntu-drivers-common
  sudo ubuntu-drivers autoinstall
  sudo reboot   # then rerun below
  sudo apt install -y nvidia-container-toolkit
  sudo nvidia-ctk runtime configure
  sudo systemctl restart docker
  ```
  Verify: `nvidia-smi` and `docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi`

## 1) Create folders

```bash
mkdir -p ~/sealed-box-ai/stack/{env,data,logs} ~/sealed-box-ai/agents
```

## 2) Drop in docker-compose.yml

Copy the compose from `docs/stack-setup.md` (with pinned images) and save it as `~/sealed-box-ai/stack/docker-compose.yml`. If you’re unsure, open that doc, copy the YAML block, and paste into the file:
```bash
cat > ~/sealed-box-ai/stack/docker-compose.yml <<'EOF'
# (paste the YAML from docs/stack-setup.md here)
EOF
```

## 3) Create env/stack.env

```bash
cat > ~/sealed-box-ai/stack/env/stack.env <<'EOF'
WORKER_MODEL=Qwen/Qwen2.5-32B-Instruct
WORKER_MAX_CONTEXT=8192
WATCHDOG_MODEL=microsoft/Phi-3.5-mini-instruct
WATCHDOG_MAX_CONTEXT=4096
EOF
```

Adjust model IDs to your hardware (see `docs/models.md`). For 12–16 GB VRAM, use the smaller options there. If offline or using gated models (Meta), pre-download models/embeddings into `data/models` first and set up a Hugging Face token/mirror per `docs/models.md`.

## 4) Start core stack

```bash
cd ~/sealed-box-ai/stack
docker compose --env-file ./env/stack.env up -d
docker compose ps
```

## 5) Verify services

```bash
curl http://127.0.0.1:8000/v1/models      # worker
curl http://127.0.0.1:8100/v1/models      # watchdog
curl http://127.0.0.1:6333/collections    # qdrant
```

## 6) Set up Open WebUI

- Browse to `http://127.0.0.1:3000`.
- Create admin user.
- Settings -> Connections:
  - Provider: OpenAI-compatible.
  - Base URL: `http://worker-llm:8000/v1` (inside compose). From host via proxy: use your proxy URL.
  - API key: `local-sealed-box` (or the one you set).
- Add models: `Qwen/Qwen2.5-32B-Instruct`, `Qwen/Qwen2.5-Coder-14B-Instruct`, `Qwen/Qwen2.5-Math-7B-Instruct` (optional).
- Send a “hello” message to confirm response.

If you set the base URL from inside the UI container, use the service name (`http://worker-llm:8000/v1`). From the host (through your reverse proxy), use your proxy URL instead; do not point the host directly at `worker-llm`.

## 7) Build the internet-research agent (real fetch) — optional

Skip this if you want a fully sealed/offline setup, or swap the search backend to an internal search endpoint instead of DuckDuckGo.

Structure:
```
~/sealed-box-ai/agents/internet-research/
  Dockerfile
  requirements.txt
  app.py
  allowlist.txt
```

`allowlist.txt` (prefixes you trust):
```
https://datatracker.ietf.org/
https://learn.microsoft.com/
https://docs.aws.amazon.com/
```

`requirements.txt`:
```
fastapi
uvicorn
httpx
duckduckgo_search==4.2.2
readability-lxml
```

`app.py` (query-aware search + allowlist filter):
```python
import os, time, json, pathlib
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from duckduckgo_search import DDGS
import httpx
from readability import Document

ALLOW = [l.strip() for l in open("allowlist.txt") if l.strip()]
LOG_PATH = pathlib.Path(os.getenv("LOG_PATH", "/logs/agents/internet-research.log"))

class Req(BaseModel):
    query: str
    max_sources: int = 5
    run_id: str | None = None

app = FastAPI()

def log(line: dict):
    LOG_PATH.parent.mkdir(parents=True, exist_ok=True)
    with LOG_PATH.open("a", encoding="utf-8") as f:
        f.write(json.dumps(line) + "\n")

def allowed(url: str) -> bool:
    return any(url.startswith(p) for p in ALLOW)

def fetch(url: str):
    with httpx.Client(timeout=8) as client:
        r = client.get(url, headers={"User-Agent": "sbx-agent/1.0"})
        r.raise_for_status()
    doc = Document(r.text)
    text = doc.summary()
    return doc.title(), doc.summary()[:4000]

@app.post("/agent/internet-research")
def handler(req: Req):
    t0 = time.time()
    results = []
    with DDGS() as ddgs:
        for r in ddgs.text(req.query, max_results=req.max_sources):
            if not r.get("href"):
                continue
            url = r["href"]
            if not allowed(url):
                continue
            results.append(url)
            if len(results) >= req.max_sources:
                break
    sources = []
    for u in results:
        try:
            title, body = fetch(u)
            sources.append({"url": u, "title": title, "snippet": body[:480]})
        except Exception as e:
            log({"ts": time.time(), "run_id": req.run_id, "query": req.query, "url": u, "error": str(e)})
            continue
    if not sources:
        raise HTTPException(status_code=502, detail="No sources fetched")
    summary = f"{len(sources)} sources used."
    out = {"query": req.query, "summary": summary, "snippets": [s["snippet"] for s in sources], "sources": sources}
    log({"ts": time.time(), "dur": round(time.time()-t0,2), "run_id": req.run_id, "query": req.query, "urls": [s['url'] for s in sources]})
    return out
```

`Dockerfile`:
```
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py allowlist.txt .
CMD ["uvicorn","app:app","--host","0.0.0.0","--port","8080"]
```

Add to `~/sealed-box-ai/stack/docker-compose.yml`:
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
```

Rebuild/up:
```bash
cd ~/sealed-box-ai/stack
docker compose build internet-research
docker compose up -d
```

Note: this agent requires outbound HTTP to your allowlist and to DuckDuckGo (or whatever search API you choose). If you are air-gapped, swap in your own internal search endpoint instead.

## 8) Build intel-sync agent — optional

Skip or point feeds to internal sources if you want a fully sealed/offline setup.

Follow `docs/agents-and-tools.md` (intel-sync section). Place it in `~/sealed-box-ai/agents/intel-sync/`, add the service to compose, and rebuild/up:

```bash
cd ~/sealed-box-ai/stack
docker compose build intel-sync
docker compose up -d
```

## 9) Ingest a sample note

Save `ingest.py` from `docs/training-and-routines.md` into `~/sealed-box-ai/stack/ingest.py`, then:
```bash
cd ~/sealed-box-ai/stack
python -m venv .venv && source .venv/bin/activate
pip install sentence-transformers requests
mkdir -p staging
echo "Test note content" > staging/test.md
export QDRANT_URL="http://127.0.0.1:6333"
export QDRANT_COLLECTION="notes"
export EMBED_MODEL="BAAI/bge-m3"
python ingest.py
curl -s http://127.0.0.1:6333/collections/notes/points/count
```

If the count increases, ingestion worked. Move your real markdown/txt/PDF-to-text files into `staging/` and re-run. If offline, pre-download the needed wheels (torch, sentence-transformers, deps) into a local wheelhouse and `pip install --no-index --find-links /path/to/wheels ...`.

## 10) Wire watchdog

- Ensure watchdog is running (already in compose).
- If using the bridge from `docs/watchdog-monitoring.md`, add its service to compose and point Open WebUI/API to the bridge URL (internal service name). Keep it behind your proxy/auth.

## 11) Lock down access

- Apply firewall + reverse proxy from `docs/safety-and-isolation.md`.
- Only port 443 (proxy) should be exposed on LAN/WAN; no direct LLM/agent ports.
- At minimum on Ubuntu with UFW:
  ```bash
  sudo ufw default deny incoming
  sudo ufw default allow outgoing
  sudo ufw allow from 192.168.0.0/16 to any port 22 proto tcp
  sudo ufw allow from 192.168.0.0/16 to any port 443 proto tcp
  sudo ufw enable
  ```
  Adjust the subnet to your LAN. Do not open 8000/8100/8080 externally; reach them via the reverse proxy or SSH/VPN/tunnel only.

## 12) Run health checks

```bash
curl -sf http://127.0.0.1:8000/v1/models
curl -sf http://127.0.0.1:8100/v1/models
curl -sf http://127.0.0.1:6333/collections
```

Open WebUI: send a test prompt. Research agent: trigger via tool or call its endpoint with a simple query. Intel-sync: confirm logs and Qdrant entries.

If something fails, jump to `docs/troubleshooting.md` for common fixes.

## 13) Backups (quick)

```bash
cd ~/sealed-box-ai/stack
docker compose stop qdrant
tar czf /backups/sbx-configs.tgz docker-compose.yml env
tar czf /backups/sbx-qdrant.tgz data/qdrant
tar czf /backups/sbx-openwebui.tgz data/open-webui
tar czf /backups/sbx-logs.tgz logs
docker compose start qdrant
```

Include agent configs (allowlist, feeds.yml) and models cache if you want a full offline restore.

## 14) Troubleshooting quick hits

- **nvidia-smi fails or no GPU in containers:** reinstall/update drivers and NVIDIA container toolkit; reboot; rerun `nvidia-ctk runtime configure`.
- **curl to worker/watchdog fails:** check `docker compose ps`; inspect logs (`docker logs sealed-box-worker`); confirm `docker compose --env-file ./env/stack.env up -d` ran from the right folder.
- **UI can’t reach worker:** make sure UI base URL uses `http://worker-llm:8000/v1` inside compose, or your proxy URL from the host. Do not point the host directly at `worker-llm`.
- **Model download/auth errors:** set HF token (if gated) or pre-download into `data/models` and retry.
- **Research agent empty:** ensure outbound allowed to your allowlist + search API; if sealed/offline, disable the agent or point it at an internal search.
- **OOMs / crashes:** lower `--max-model-len`, use smaller models (see `docs/models.md`), and avoid running multiple big models together on small VRAM.
