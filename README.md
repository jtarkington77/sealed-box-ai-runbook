# Sealed Box AI Runbook

I write a lot about the risks and abuse potential of AI systems -- especially large hosted models and so-called "private" AI that still runs on someone else's infrastructure. Even when vendors say "we don't train on your data," you're still depending on their logging, telemetry, and internal controls.

Day to day, my own AI runs locally: A RTX A6000 GPU, 128 GB RAM, 1I9 13900K CPU, and several terabytes of storage, fully under my control. This repo documents how I structure that environment and how you can build a similar sealed setup for yourself at whatever scale your hardware allows.

The goal is simple:  
**show you how to move from "trust the cloud" to "I own the stack"** -- models, workflows, guardrails, and all.

### Why I ended up building this

I didn’t start with a monster GPU box. My first “local AI” was a single CPU-only model running on whatever hardware I had. From there I moved to a dedicated AI server with a 2080, kept pushing it until it broke, and used every failure to harden the stack, tighten the network, and clean up the workflows.

Over time that grew into a full AI server: multiple local models, better GPUs, and a stack of agents that handle real work. Today I run roughly two dozen agents tied into tools like n8n, AnythingLLM, and a few utilities I’ve built myself. They help with everything from lab work and writing to personal finance workflows.

The rule hasn’t changed: the AI can be powerful, but the data stays mine. Prompts, documents, logs, and even banking workflows live on hardware I control, with tightly-scoped access to the outside world instead of shipping everything to a hosted “private” model.


---

This repo is a set of guides for building your own **sealed local AI stack**:

- One or more **worker models** that do the actual work.
- A smaller **watchdog model** that keeps an eye on what's happening.
- Local **retrieval** over your own data.
- **Agents** and workflows running on top of that.
- Network and host **safety** so it stays on your side of the wire.

All the details live in `docs/`.  
This file just tells you what you're looking at, what kind of hardware it assumes,  
what we expect you to know, and where to go next.

---

## How this setup flows (my workflow)

- You talk to **Open WebUI** (or API clients).
- Open WebUI calls the **worker model** for answers.
- When the worker needs fresh info, it calls a **tool**; your orchestrator routes that to a **locked-down agent** (internet research or intel sync). The worker never has outbound internet.
- The **watchdog** gets a summary of each turn (prompt, tools used, answer) and scores/flags it. Logs capture run_id, URLs hit, verdicts.
- **RAG** pulls from your local vector store (Qdrant) that you fill with notes/docs and the intel agent. Answers stay local.
- Exposure and auth sit at the **reverse proxy**; everything else is bound to localhost/internal nets.

---

## Recommended folder layout

Keep this repo wherever you like. For the running stack and agents, use a predictable layout:

```
~/sealed-box-ai/
  stack/            # docker-compose.yml, env/, data/, logs/ for core services
    env/
    data/
    logs/
  agents/           # agent code/images
    internet-research/
    intel-sync/
```

All file paths in the docs assume this layout; adjust if you change it.

> If you are air-gapped: pre-download models (worker, watchdog, embeddings) and Python wheels to `data/models` / a local wheelhouse before running compose. The defaults (Qwen, Phi, BGE) will otherwise attempt to pull from the internet.
> If you use gated models (e.g., Meta), set up Hugging Face auth or mirror them internally; see notes in `docs/models.md`.

---

## What's in this repo

This repo gives you:

- Conceptual layouts and **markdown runbooks**.
- Example patterns for:
  - Wiring worker + watchdog models.
  - Adding retrieval over your own files.
  - Building and running agents.
  - Locking the whole thing down and still reaching it remotely.
- Reference configs you can adapt to your own stack and tools.

It does **not** promise:

- A one-click install.
- That it'll feel good on tiny hardware.
- That you never have to tweak anything.

You're expected to read, adapt, and build.

---

## Hardware snapshot

This project targets a **real local AI setup**, not tiny toy models.

Your **GPU VRAM** is the main limit on what you can realistically run.  
Detailed model/hardware discussion lives in [`docs/models.md`](docs/models.md).  
Here's the quick version.

### Bare minimum - 12-16 GB VRAM

You can stand the design up and prove it out, but you'll feel the constraints.

- **CPU:** 8 modern cores  
- **RAM:** 64 GB  
- **Storage:** 2 TB NVMe SSD  
- **GPU:** NVIDIA CUDA, **12-16 GB VRAM**

What this means in practice:

- Smaller or heavily-compressed worker models.
- A simple watchdog (likely on CPU or sharing tight VRAM).
- Agents and RAG are possible, but you'll be trading speed, context size, and quality constantly.

Good for learning the pattern and light use.

### Recommended - 16-24 GB VRAM

This is the range most people should aim at if they actually want to use this daily.

- **CPU:** 12-16 cores  
- **RAM:** 96-128 GB  
- **Storage:** 2-4 TB NVMe SSD  
- **GPU:** NVIDIA CUDA, **16-24 GB VRAM**

What you get:

- A local worker model that doesn't feel like a toy.
- A watchdog model that can stay resident.
- Agents, retrieval, and scheduled jobs that are usable without every run feeling painful.

### "Comfort build" - 24+ GB VRAM

If you want headroom for bigger models and heavier workflows:

- **CPU:** 16+ cores  
- **RAM:** 128 GB+  
- **Storage:** 4 TB+ NVMe SSD  
- **GPU:** NVIDIA CUDA, **24 GB+ VRAM**  
  - e.g. 4090 / 5090 / A6000 / RTX 6000-class / newer 6000-series pro cards

What this unlocks:

- Room for larger local models plus a watchdog.
- Bigger context windows and heavier RAG over real data.
- Multiple agents and workflows running at the same time without constantly choking.

For **actual model choices, pros/cons, and what's realistic per tier**, see:  
[`docs/models.md`](docs/models.md).

> Note: Examples assume a CUDA-capable NVIDIA GPU. Other GPUs may work, but are not the focus here.

---

## What we assume you already know

This repo assumes the reader can:

- Install and update a **Linux** server or VM.
- Use a **terminal** without hand-holding.
- Work with a container runtime (**Docker / Podman** or similar).
- Edit configuration files (YAML, env files, etc.) and restart services.
- Read basic logs and error messages.
- Understand, at a high level:
  - IPs and ports.
  - What a reverse proxy is (Nginx, Caddy, Traefik, etc.).
  - That "exposing this to the internet" is a bad default.

You don't have to be a full-time infra engineer,  
but you do need to be willing to troubleshoot.

---

## Docs overview

Everything else lives under `docs/`. Use this as a map:

- **Models & hardware choices**  
  [`docs/models.md`](docs/models.md)  
  How to pick worker + watchdog models for each hardware tier, and what you give up or gain.

- **Agents & workflows**  
  [`docs/agents-and-tools.md`](docs/agents-and-tools.md)  
  Example agents, how they're wired, and how to adapt them to your own use cases.

- **Using the stack day-to-day**  
  [`docs/usage.md`](docs/usage.md)  
  How to actually use chat, agents, and the API for real work (notes, logs, lab work, writing, etc.).

- **Training, routines, and "learning" over time**  
  [`docs/training-and-routines.md`](docs/training-and-routines.md)  
  How to feed it your data safely and build routines without wrecking your privacy.

- **Safety, isolation, and remote access**  
  [`docs/safety-and-isolation.md`](docs/safety-and-isolation.md)  
  Network and host patterns for keeping it sealed off but still reachable by you.

- **Watchdog & monitoring**  
  [`docs/watchdog-monitoring.md`](docs/watchdog-monitoring.md)  
  How the watchdog model is wired in, what it sees, and how to read its output.

- **Operations & maintenance**  
  [`docs/operations.md`](docs/operations.md)  
  Backups, upgrades, health checks, and what to do when something looks wrong.

- **Troubleshooting**  
  [`docs/troubleshooting.md`](docs/troubleshooting.md)  
  Common errors (Docker/GPU, YAML, model downloads, agents, OOMs) and quick fixes.

- **Build sheet (single-path commands)**  
  [`docs/build-sheet.md`](docs/build-sheet.md)  
  Copy/paste commands for a first full setup (folders, compose/env, optional agents, ingestion, health checks, backups).

---

## How to use this runbook (order)

1. **Pick models** that fit your GPU: read `docs/models.md` (see “What I'm actually running”).
   - If you're on smaller hardware (12–16 GB VRAM), use the smaller model suggestions in `docs/models.md`.
2. **Stand up the core stack** on localhost: follow `docs/stack-setup.md` in `~/sealed-box-ai/stack/`; verify worker, watchdog, Qdrant, Open WebUI.
3. **Wire the UI/API**: use `docs/usage.md` to point Open WebUI at the worker, add model entries, and run the API curl test.
4. **Choose your mode for fresh info**:
   - **Sealed / offline:** skip the internet-research agent; point intel-sync to internal feeds only; rely on your own documents/RAG.
   - **Online (tightly scoped):** build the internet-research agent (allowlist + search API) and intel-sync; add them to compose; set firewall allowlists. The worker still has no outbound.
5. **Ingest your data**: run the ingestion steps in `docs/training-and-routines.md` (embedding model, Qdrant collection, sample note) before adding real docs.
6. **Enable watchdog scoring**: wire every turn through the watchdog per `docs/watchdog-monitoring.md` (compose + middleware/bridge).
7. **Lock down access**: apply firewall, reverse proxy, and tunnel patterns from `docs/safety-and-isolation.md`.
8. **Operate**: use `docs/operations.md` for backups, upgrades, health checks, and incident response.

If you get stuck, use the debug checklists in `docs/usage.md` and `docs/operations.md`.

### First-hour commands (copy/paste starter)

```bash
# 1) Create folders
mkdir -p ~/sealed-box-ai/stack/{env,data,logs} ~/sealed-box-ai/agents

# 2) Place docker-compose.yml from docs/stack-setup.md into ~/sealed-box-ai/stack/
# 3) Create env/stack.env with your model IDs from docs/models.md

# Example env:
cat > ~/sealed-box-ai/stack/env/stack.env <<'EOF'
WORKER_MODEL=Qwen/Qwen2.5-32B-Instruct
WORKER_MAX_CONTEXT=8192
WATCHDOG_MODEL=microsoft/Phi-3.5-mini-instruct
WATCHDOG_MAX_CONTEXT=4096
EOF

# 4) Bring up core stack (from ~/sealed-box-ai/stack)
cd ~/sealed-box-ai/stack
docker compose --env-file ./env/stack.env up -d

# 5) Verify services
curl http://127.0.0.1:8000/v1/models
curl http://127.0.0.1:8100/v1/models
curl http://127.0.0.1:6333/collections

# 6) Open WebUI
echo "Browse to http://127.0.0.1:3000 and follow docs/usage.md setup steps"
```

Then proceed to build agents under `~/sealed-box-ai/agents/` using `docs/agents-and-tools.md`.
