# Troubleshooting quick guide

Common issues and fixes for the sealed-box stack.

## 1) Docker/NVIDIA problems

- `nvidia-smi` fails or GPU not visible in containers:
  - Reinstall/update drivers and NVIDIA container toolkit (see `docs/build-sheet.md` 0) Paths and prerequisites).
  - Run: `sudo nvidia-ctk runtime configure` then `sudo systemctl restart docker`.
  - Reboot after driver installs.
- `docker run --gpus all ... nvidia-smi` fails:
  - Check `/etc/docker/daemon.json` for default runtime or rerun the nvidia-ctk configure step.

## 2) Compose/YAML errors

- `docker compose` complains about YAML:
  - Ensure compose file is saved with proper spacing (two spaces). Re-copy from `docs/stack-setup.md`.
  - Validate with `docker compose config`.

## 3) Model download/auth errors

- Hugging Face gated models (Meta) fail to download:
  - Set `HUGGINGFACE_HUB_TOKEN` in the environment before starting compose.
  - Or pre-download with `huggingface-cli download ... --local-dir ./data/models` and point the runtime to that path.
- Offline/air-gapped:
  - Preload models/embeddings into `data/models`.
  - Pre-download Python wheels (torch, sentence-transformers) and install with `pip --no-index --find-links /path/to/wheels`.

## 4) Worker/Watchdog/API unreachable

- `curl http://127.0.0.1:8000/v1/models` fails:
  - `cd ~/sealed-box-ai/stack && docker compose --env-file ./env/stack.env up -d`.
  - Check logs: `docker logs sealed-box-worker`.
- UI cannot reach worker:
  - Inside compose: base URL must be `http://worker-llm:8000/v1`.
  - From host via proxy: use your proxy URL, not `worker-llm`.

## 5) Research/Intel agents

- Research agent returns nothing:
  - Ensure outbound allowed to your allowlist and the search API (DuckDuckGo or your chosen backend).
  - If sealed/offline, disable this agent or point it to an internal search endpoint.
- Intel-sync errors:
  - Check `logs/agents/intel-sync.log` for feed/timeouts.
  - Ensure `QDRANT_URL` is reachable and feeds.yml URLs are allowed outbound.

## 6) Out-of-memory (OOM) or slow performance

- Lower `--max-model-len` in compose/env.
- Use smaller models (see `docs/models.md` small-hardware section).
- Do not run multiple big models concurrently on 12–16 GB VRAM.
- Watch `nvidia-smi` and `docker stats` during load.

## 7) Port exposure/safety

- Do not expose 8000/8100/8080 externally. Only expose the reverse proxy on 443.
- Use the UFW example in `docs/build-sheet.md` or `docs/safety-and-isolation.md`.
- Keep the watchdog bridge behind the same proxy/auth.

## 8) Health checks

```bash
curl -sf http://127.0.0.1:8000/v1/models
curl -sf http://127.0.0.1:8100/v1/models
curl -sf http://127.0.0.1:6333/collections
```
Automate these and alert on failures.

## 9) Backups/restore

- Stop qdrant during snapshot: `docker compose stop qdrant` then tar `docker-compose.yml env data/qdrant data/open-webui logs`.
- Restore: place files back, `docker compose up -d`, rerun health checks.
