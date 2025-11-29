# Sealed Box AI Runbook

I write a lot about the risks and abuse potential of AI systems – especially large hosted models and so-called “private” AI that still runs on someone else’s infrastructure. Even when vendors say “we don’t train on your data,” you’re still depending on their logging, telemetry, and internal controls.

Day to day, my own AI runs locally: A RTX A6000 GPU, 128 GB RAM, 1I9 13900K CPU, and several terabytes of storage, fully under my control. This repo documents how I structure that environment and how you can build a similar sealed setup for yourself at whatever scale your hardware allows.

The goal is simple:  
**show you how to move from “trust the cloud” to “I own the stack”** – models, workflows, guardrails, and all.

---

This repo is a set of guides for building your own **sealed local AI stack**:

- One or more **worker models** that do the actual work.
- A smaller **watchdog model** that keeps an eye on what’s happening.
- Local **retrieval** over your own data.
- **Agents** and workflows running on top of that.
- Network and host **safety** so it stays on your side of the wire.

All the details live in `docs/`.  
This file just tells you what you’re looking at, what kind of hardware it assumes,  
what we expect you to know, and where to go next.

---

## What’s in this repo

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
- That it’ll feel good on tiny hardware.
- That you never have to tweak anything.

You’re expected to read, adapt, and build.

---

## Hardware snapshot

This project targets a **real local AI setup**, not tiny toy models.

Your **GPU VRAM** is the main limit on what you can realistically run.  
Detailed model/hardware discussion lives in [`docs/models.md`](docs/models.md).  
Here’s the quick version.

### Bare minimum — 12–16 GB VRAM

You can stand the design up and prove it out, but you’ll feel the constraints.

- **CPU:** 8 modern cores  
- **RAM:** 64 GB  
- **Storage:** 2 TB NVMe SSD  
- **GPU:** NVIDIA CUDA, **12–16 GB VRAM**

What this means in practice:

- Smaller or heavily-compressed worker models.
- A simple watchdog (likely on CPU or sharing tight VRAM).
- Agents and RAG are possible, but you’ll be trading speed, context size, and quality constantly.

Good for learning the pattern and light use.

### Recommended — 16–24 GB VRAM

This is the range most people should aim at if they actually want to use this daily.

- **CPU:** 12–16 cores  
- **RAM:** 96–128 GB  
- **Storage:** 2–4 TB NVMe SSD  
- **GPU:** NVIDIA CUDA, **16–24 GB VRAM**

What you get:

- A local worker model that doesn’t feel like a toy.
- A watchdog model that can stay resident.
- Agents, retrieval, and scheduled jobs that are usable without every run feeling painful.

### “Comfort build” — 24+ GB VRAM

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

For **actual model choices, pros/cons, and what’s realistic per tier**, see:  
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
  - That “exposing this to the internet” is a bad default.

You don’t have to be a full-time infra engineer,  
but you do need to be willing to troubleshoot.

---

## Docs overview

Everything else lives under `docs/`. Use this as a map:

- **Models & hardware choices**  
  [`docs/models.md`](docs/models.md)  
  How to pick worker + watchdog models for each hardware tier, and what you give up or gain.

- **Agents & workflows**  
  [`docs/agents-and-workflows.md`](docs/agents-and-workflows.md)  
  Example agents, how they’re wired, and how to adapt them to your own use cases.

- **Using the stack day-to-day**  
  [`docs/usage.md`](docs/usage.md)  
  How to actually use chat, agents, and the API for real work (notes, logs, lab work, writing, etc.).

- **Training, routines, and “learning” over time**  
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

---

## How to start

1. Check your hardware against the tiers above.  
2. Read [`docs/models.md`](docs/models.md) to decide what you can realistically run.  
3. From there, follow the docs in whatever order fits your environment and priorities.
