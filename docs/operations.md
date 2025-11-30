# Operations and maintenance

Day-2 guidance for keeping the sealed box healthy, backed up, and recoverable.
All paths assume `~/sealed-box-ai/stack`; adjust if you chose a different base directory.

## 1. What "healthy" looks like

- Containers are up (`docker ps`) and responding on localhost (worker 8000, watchdog 8100, UI 3000, vector store 6333/6334).
- GPU/CPU are not pegged unexpectedly (`nvidia-smi`, `docker stats`).
- Logs rotate and do not fill disks.
- Watchdog and agent logs show expected traffic and no repeated high-risk tags.

## 2. Backups

- **Configs and env**: version control your compose file and env files; keep a secure copy off-box.
- **Models**: cache/download directories (for example `data/models/`) plus checksums; mirror to secondary storage if possible.
- **Vector store**: snapshot `data/qdrant/` (or your chosen path) regularly. Stop the container or use built-in export tools before copying for consistency.
- **Logs**: watchdog, agent, UI/app logs. Rotate and back them up if they are part of your audit trail.
- **Notes/ingestion sources**: back up the raw source folder so you can rebuild the vector store if needed.
- **Agent configs**: allowlists, feeds.yml, and any tokens/keys the agents rely on.

Keep a short **restore checklist**: bring up Docker, restore env/configs, restore vector snapshot, pull models, start compose.

### 2.1 Quick restore drill

1. New host: install Docker + NVIDIA toolkit (see `docs/stack-setup.md` GPU sanity steps).
2. Clone configs/env + `docker-compose.yml` to the same path.
3. Restore `data/qdrant/`, `data/models/`, `data/open-webui/`, `logs/` from backup.
4. `docker compose up -d`.
5. Run health checks (section 4).

### 2.2 Backup commands (local shell)

```bash
cd ~/sealed-box-ai/stack
# Stop only qdrant during snapshot to avoid write races (optional)
docker compose stop qdrant
tar czf /backups/sbx-configs.tgz docker-compose.yml env
tar czf /backups/sbx-qdrant.tgz data/qdrant
tar czf /backups/sbx-openwebui.tgz data/open-webui
tar czf /backups/sbx-logs.tgz logs
docker compose start qdrant
```
Swap the stop/start for a full `docker compose stop` if you prefer a quiet snapshot.

### 2.3 Log rotation (example)

Create `/etc/logrotate.d/sealed-box`:
```
/home/you/sealed-box-ai/stack/logs/**/*.log {
  daily
  rotate 7
  compress
  missingok
  copytruncate
}
```
Adjust paths to your install location.

## 3. Upgrades

- **Models**: change one at a time; record model id, quantization, and role. Roll back if quality regresses.
- **Containers**: pull new images (`docker compose pull`) during a maintenance window; restart services; smoke test UI and API.
- **OS and drivers**: upgrade GPU drivers and `nvidia-container-toolkit` cautiously; re-run GPU sanity checks from `docs/stack-setup.md`.
- **Reverse proxy/tunnel**: update certificates and auth plugins as needed; re-check that only the proxy is listening on 443.

### 3.1 Quick upgrade loop

```bash
cd ~/sealed-box-ai/stack
docker compose pull
docker compose down        # optional if you want a clean restart
docker compose up -d
docker compose ps
```
Then run the health checks in section 4.

### 3.2 Rollback hint

- Keep the previous image tags noted (or explicitly pin them). If an upgrade fails, revert the tag in `docker-compose.yml` to the prior version and `docker compose up -d`.
- For model changes, keep the prior model ID/quant in a note; switch back in `env/stack.env` if behavior regresses.

## 4. Health checks

- **Worker API**: `curl http://127.0.0.1:8000/v1/models` and a short chat completion.
- **Watchdog API**: `curl http://127.0.0.1:8100/v1/models`.
- **Vector store**: hit the HTTP port and list collections.
- **UI**: load the login page and confirm it talks to the worker.
- Automate these with a small script on a timer; alert if any step fails.

Example script stub (bash):
```bash
#!/usr/bin/env bash
set -e
curl -sf http://127.0.0.1:8000/v1/models >/dev/null
curl -sf http://127.0.0.1:8100/v1/models >/dev/null
curl -sf http://127.0.0.1:6333/collections >/dev/null
```

Add a cron (as your user) to run this every 5 minutes and email or log failures.

## 5. Monitoring and logs

- Collect key logs in `logs/` with rotation:
  - Reverse proxy access/error.
  - Worker, watchdog, vector store.
  - Agents (internet research, intel sync).
  - Watchdog verdicts (see `docs/watchdog-monitoring.md`).
- Keep agent configs (allowlists, feeds.yml) in backup scope so you can rebuild after restore.
- Consider light metrics (container CPU/mem, GPU utilization, request latency) via `docker stats` or a small Prometheus/exporter setup if you already run one.

## 6. Incident response (quick playbook)

If something looks wrong (unexpected outputs, suspected leak, or compromised agent):

1. **Freeze exposure**: disable tunnels/port forwards; restrict proxy auth; optionally stop agent containers.
2. **Preserve logs**: copy relevant logs (proxy, agent, worker, watchdog) to a safe path.
3. **Check blast radius**:
   - Which API keys were used?
   - Which agents ran and which URLs they hit?
   - What data collections were touched?
4. **Contain**:
   - Revoke affected API keys.
   - Tighten firewall rules/allowlists.
   - Patch or rebuild compromised containers from clean images.
5. **Recover**:
   - Restore from backup if needed.
   - Re-run health checks and a small test suite of prompts/queries.
6. **Document** what happened and what was changed.

Keep a one-page checklist in your ops folder so you can run this under pressure.

## 7. Change management

- Keep a changelog for:
  - Model swaps and quantization changes.
  - Config changes to agents/allowlists.
  - Network/proxy/auth changes.
- Test changes on a small set of representative prompts before making them default.
- Prefer **feature flags** (env vars, config toggles) where possible so you can roll back quickly.

## 8. Capacity planning

- Watch GPU VRAM headroom when loading multiple models. Avoid loading everything at once on small cards.
- Use batch sizes and context windows that fit your hardware; lower `--max-model-len` if you see OOMs.
- For heavy workloads, separate roles across hardware if available (worker on the big GPU, watchdog/embeddings on CPU or a smaller GPU).

### 8.1 Quick VRAM sanity

```bash
nvidia-smi
docker stats
```
If VRAM is tight, drop secondary models or reduce context (`--max-model-len`) before retrying.
