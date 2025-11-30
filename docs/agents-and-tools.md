# Agents and controlled internet research

This doc is about **one thing**:

> Let the system look things up online **without ever giving the main model direct internet access**.

We do that with **agents**. If you want a fully sealed/offline setup, you can skip the internet-research agent or point it to an internal search endpoint; rely on intel-sync + your own documents for RAG.

**Offline/internal search option (example)**
- Skip DuckDuckGo and instead expose a small internal search service (for example, a local index over your docs) on the LAN and add its URL to `allowlist.txt`.
- Keep the agent code the same, but swap the search call to your internal endpoint and lock outbound to LAN-only.

---

## 1. What an "agent" is in this stack

In this runbook an *agent* is not magic or a framework. It's:

> A small, tightly-scoped service that:
> - Receives a request from the orchestrator,
> - Does some work (HTTP calls, lookups, scripts),
> - Returns a compact result,
> - Logs everything it did.

The worker LLM:

- Never makes raw HTTP requests.
- Never sees your API keys.
- Never decides where it can and can't connect.

It only ever says, in effect:

> "I need more information -- ask the `internet_research` tool for this query."

Your **orchestrator** (or UI / middleware) is the one that:

- Sees that tool request.
- Calls the appropriate agent endpoint.
- Feeds the result back into the model as extra context.
- Optionally asks the watchdog to score the whole thing.

Firewall rules from `safety-and-isolation.md` enforce that:

- The worker container has no general outbound internet.
- The agent containers are the **only** things allowed to talk to the outside world,
  and even then only to a small allowlist.

---

## 2. Architecture at a glance

Pieces involved:

- **Worker model** -- your main LLM (no direct internet).
- **Watchdog model** -- smaller model scoring / monitoring requests and responses.
- **Orchestrator** -- the glue that:
  - Receives prompts from the UI / API callers.
  - Calls the worker.
  - Triggers agents when needed.
  - Sends bundles to the watchdog.
- **Agents** -- small services that:
  - Have limited outbound HTTP.
  - Return summaries / structured results.
- **Firewall / network rules** -- enforce who can talk to what.

High-level path when the model needs fresh info:

1. User asks something that requires up-to-date information.
2. Worker decides it needs external info and emits a *tool call* like:

   > `internet_research(query="what changed in TLS 1.3 vs 1.2")`

3. Orchestrator sees that tool call and sends something like this to the agent:

   POST /agent/internet-research  
   Content-Type: application/json  

       {
         "query": "what changed in TLS 1.3 vs 1.2",
         "run_id": "2025-01-15T21-43-17Z-abc123"
       }

4. The **internet-research agent**:
   - Validates the query.
   - Hits only allowlisted docs/sites.
   - Produces a short summary + key snippets + source list.
   - Logs everything under `logs/agents/internet-research.log`.
   - Returns JSON to the orchestrator.

5. Orchestrator feeds that result back into the worker as extra context and asks for a final answer.
6. Optionally, `{user_prompt, research_result, final_answer}` are sent to the watchdog for scoring.

The worker itself **never** sees "do arbitrary HTTP" as a tool, and the container doesn't have the network rights to do it anyway.

---

## 3. Agent #1 -- On-demand internet research

This is the **essential** agent: when the main model needs to look something up on the internet, this is the only door.

### 3.1 What this agent is responsible for

- Accept a plain query string.
- Call a **small allowlist** of sites and/or search APIs.
- Strip junk (ads, nav, tracking garbage) as much as practical.
- Return:
  - A short human-readable summary.
  - A few key snippets.
  - A list of URLs / titles that were used.
- Log:
  - Timestamp.
  - Query.
  - URLs actually hit.
  - Any errors.

It is **not** responsible for:

- Talking directly to the user.
- Writing long essays.
- Deciding what actions to take next.

That's the worker's job.

### 3.2 Example API shape

This is the interface you want to expose on your LAN, behind the reverse proxy:

- Endpoint: `POST /agent/internet-research`
- Request body (JSON):

       {
         "query": "what changed in TLS 1.3 vs 1.2",
         "max_sources": 5,
         "run_id": "optional-correlation-id"
       }

- Response body (JSON):

       {
         "query": "what changed in TLS 1.3 vs 1.2",
         "summary": "TLS 1.3 removes older key exchange modes and requires forward secrecy...",
         "snippets": [
           "TLS 1.3 removed RSA key exchange and legacy bulk ciphers.",
           "All cipher suites in TLS 1.3 provide forward secrecy by design."
         ],
         "sources": [
           {
             "url": "https://datatracker.ietf.org/doc/html/rfc8446",
             "title": "The Transport Layer Security (TLS) Protocol Version 1.3"
           },
           {
             "url": "https://learn.microsoft.com/...",
             "title": "Configure TLS 1.3 on Windows Server"
           }
         ]
       }

Your orchestrator can treat this as a **tool result** and feed it back into the worker as extra context.

### 3.3 Allowlist and outbound control

On the firewall and in the agent code, lock this down:

- Example allowlist (adjust for your world):
  - `https://datatracker.ietf.org/`
  - `https://learn.microsoft.com/`
  - `https://docs.aws.amazon.com/`
  - `https://your-wiki.internal/`
- Reject anything that doesn't match those prefixes.
- In `AI_NET -> WAN` rules (from `safety-and-isolation.md`):
  - Only allow the research agent container to talk to those domains.
  - Block everything else.

If you later need more sites, you **add them consciously** to the allowlist; the model does not get to decide.

### 3.4 How the worker calls it (conceptually)

From the worker's point of view, this is just a tool called `internet_research`.

Very simplified example of a tool definition (pseudo-JSON):

    {
      "name": "internet_research",
      "description": "Look up technical information on a small list of allowed documentation sites.",
      "parameters": {
        "type": "object",
        "properties": {
          "query": {
            "type": "string",
            "description": "What you want to look up."
          }
        },
        "required": ["query"]
      }
    }

When the worker emits a call to this tool, your orchestrator:

1. Reads the `query`.
2. Calls `POST /agent/internet-research`.
3. Takes the JSON response and passes it back into the model as a `tool` / `function` result.
4. Asks the worker to produce the final answer using **only that result**, not by going to the internet directly.

### 3.5 Build it end-to-end (step by step)

**Where to put this code**

- Root of agents: `~/sealed-box-ai/agents/`
- Internet research agent files live in: `~/sealed-box-ai/agents/internet-research/`
- Your main stack (compose) lives in: `~/sealed-box-ai/stack/`
- When you build, run `docker compose` from `~/sealed-box-ai/stack/` after adding the service.

**Folder layout**

```
~/sealed-box-ai/agents/
  internet-research/
    Dockerfile
    requirements.txt
    app.py
    allowlist.txt
```

**1) Define the allowlist**

`allowlist.txt` (one prefix per line):
```
https://datatracker.ietf.org/
https://learn.microsoft.com/
https://docs.aws.amazon.com/
```

**2) Minimal app (FastAPI example)**

`requirements.txt`:
```
fastapi
uvicorn
httpx
readability-lxml
```

`app.py` (minimal; replace search logic with your own API/allowlisted search):
```python
import os, time, json, httpx, pathlib
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
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
    return doc.title(), text[:4000]  # cap bytes

@app.post("/agent/internet-research")
def handler(req: Req):
    t0 = time.time()
    # Simple example: treat query as a suffix on allowlist roots (replace with your search API)
    urls = []
    for root in ALLOW:
        candidate = root.rstrip("/") + "/" + req.query.replace(" ", "+")
        urls.append(candidate)
    urls = urls[: req.max_sources]
    sources = []
    for u in urls:
        if not allowed(u):
            continue
        try:
            title, body = fetch(u)
            sources.append({"url": u, "title": title, "snippet": body[:480]})
        except Exception as e:
            log({"ts": time.time(), "run_id": req.run_id, "query": req.query, "url": u, "error": str(e)})
            continue
    if not sources:
        raise HTTPException(status_code=502, detail="No sources fetched")
    summary = f"{len(sources)} sources checked; see snippets."
    out = {"query": req.query, "summary": summary, "snippets": [s["snippet"] for s in sources], "sources": sources}
    log({"ts": time.time(), "dur": round(time.time()-t0,2), "run_id": req.run_id, "query": req.query, "urls": [s['url'] for s in sources]})
    return out
```

**Important:** This fetcher is a placeholder. Replace the “build URL from query” logic with a real search/lookup against your allowlisted docs/APIs (for example, a custom search endpoint you host). Do not deploy it as-is expecting meaningful research results. The point is to show wiring and isolation; you decide how to fetch. If you ship this to others, remove the placeholder and implement a real allowlisted fetch/search before deploying.

**3) Container**

`Dockerfile`:
```
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py allowlist.txt .
CMD ["uvicorn","app:app","--host","0.0.0.0","--port","8080"]
```

**4) Compose add-on**

Add to your main `docker-compose.yml` (in `~/sealed-box-ai/stack/`):
```yaml
  internet-research:
    build: ./agents/internet-research
    environment:
      LOG_PATH: /logs/agents/internet-research.log
    volumes:
      - ./logs/agents:/logs/agents
    networks:
      - sealed_ai_net
    restart: unless-stopped
```

**5) Wire the tool**

- Tool name: `internet_research`
- Orchestrator target: `http://internet-research:8080/agent/internet-research`
- Pass `{query, run_id}`; consume `{summary, snippets[], sources[]}`.

**6) Network rules**

- Firewall: only this container can egress to the domains in `allowlist.txt`; block everything else.
- Worker stays outbound-denied.
- If you change the search provider (here DuckDuckGo), update allowlist and firewall to match what that provider needs.

---

## 4. Agent #2 -- Scheduled "intel & docs" ingestor

The first agent is interactive ("I need to look this up now").  
The second agent handles **scheduled research**:

> Pull fresh information from a few trusted feeds and write it into your local knowledge base (RAG) so the model can use it later without going online at all.

### 4.1 What this agent does

Example responsibilities:

- On a schedule (cron, systemd timer, job runner, etc.):
  - Fetch new items from a small list of feeds / APIs, such as:
    - Vendor security advisories.
    - NVD / CVE feeds.
    - Your company blog / KB.
  - Normalize them into a simple internal format.
  - Write them into:
    - A flat file / markdown folder, and/or
    - Your vector store (Qdrant) via its API.
- Optionally call the worker locally to create short summaries per item before storing.

The worker still never goes to the internet; the agent does that work and leaves behind **local artifacts** the worker can read later.

### 4.2 Example API / job shape

You can implement this agent either:

- As a scheduled job with no external API, or
- As a small service with an admin endpoint for manual runs.

Example service interface:

- Endpoint: `POST /agent/intel-sync/run`
- Request body (JSON):

       {
         "feeds": ["all"],  // or ["msrc", "nvd"]
         "run_id": "optional-correlation-id"
       }

- Response body (JSON):

       {
         "run_id": "2025-01-15T21-50-02Z-xyz789",
         "fetched": 12,
         "stored": 10,
         "skipped": 2,
         "errors": []
       }

Internally it might:

1. Pull the latest JSON from:
   - `https://api.msrc.microsoft.com/...`
   - `https://services.nvd.nist.gov/...`
   - `https://your-blog-or-kb-feed/`
2. For each new item:
   - Build a short text blob like:

         [MSRC] CVE-2025-12345 -- Remote code execution in XYZ. Affects ...

   - Optionally ask the worker (offline) to summarize or tag severity.
   - Call Qdrant's HTTP API to upsert the new text + metadata.
3. Log everything under `logs/agents/intel-sync.log`.

Now, when you or the worker ask questions like:

> "Anything I should know about recent RDP vulnerabilities from Microsoft--

the answer comes from **your local RAG**, not from the internet in real time.  
The internet work already happened earlier via this agent.

### 4.3 How the worker uses this data

From the worker's perspective, this intel agent doesn't look like a tool. It just shows up as **extra knowledge in RAG**.

Typical flow:

1. Your RAG layer points at:
   - Your own notes / write-ups.
   - The intel summaries written by the sync agent.
2. When you ask a question, RAG pulls both.
3. The worker sees “recent MSRC advisories” as normal context text, not as live HTTP calls.

### 4.4 How to build it (minimal blueprint)

**Folder layout**

```
~/sealed-box-ai/agents/
  intel-sync/
    Dockerfile
    requirements.txt
    sync.py
    feeds.yml
```

**1) Feeds config (`feeds.yml`)**
```yaml
feeds:
  - name: msrc
    url: https://api.msrc.microsoft.com/some/feed
  - name: nvd
    url: https://services.nvd.nist.gov/...
```

**2) App (`requirements.txt`)**
```
python-dateutil
httpx
pyyaml
sentence-transformers
```

**3) `sync.py` (skeleton)**
```python
import os, json, time, pathlib, httpx, yaml
from sentence_transformers import SentenceTransformer

OUT = pathlib.Path("/data/ingest/intel")
SEEN = OUT / "seen.json"
EMBED_MODEL = os.environ.get("EMBED_MODEL","BAAI/bge-m3")
QDRANT = os.environ.get("QDRANT_URL","http://qdrant:6333")
COLLECTION = os.environ.get("QDRANT_COLLECTION","intel")
model = SentenceTransformer(EMBED_MODEL)

def load_seen():
    if SEEN.exists():
        return json.loads(SEEN.read_text())
    return {}

def save_seen(s): SEEN.write_text(json.dumps(s, indent=2))

def upsert(points):
    import requests, json as js
    requests.put(f"{QDRANT}/collections/{COLLECTION}/points",
                 headers={"Content-Type":"application/json"},
                 data=js.dumps({"points": points}),
                 timeout=30).raise_for_status()

def main():
    feeds = yaml.safe_load(open("feeds.yml"))["feeds"]
    seen = load_seen()
    for feed in feeds:
        r = httpx.get(feed["url"], timeout=15)
        r.raise_for_status()
        items = r.json().get("items", [])
        for it in items:
            uid = it.get("id") or it.get("url")
            if uid in seen:
                continue
            text = f"[{feed['name']}] {it.get('title','')}\n{it.get('description','')}\n{it.get('url','')}"
            vec = model.encode(text).tolist()
            points = [{"id": uid, "vector": vec, "payload": {"feed": feed["name"], "url": it.get("url"), "title": it.get("title"), "text": text}}]
            upsert(points)
            (OUT / f"{uid}.md").write_text(text, encoding="utf-8")
            seen[uid] = time.time()
    save_seen(seen)

if __name__ == "__main__":
    OUT.mkdir(parents=True, exist_ok=True)
    main()
```

**4) Dockerfile**
```
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY sync.py feeds.yml .
CMD ["python","/app/sync.py"]
```

**5) Compose add-on**
  ```yaml
  intel-sync:
    build: ./agents/intel-sync
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

**6) Schedule and wiring**

- The entrypoint above runs hourly; adjust sleep or swap to cron/systemd if you prefer.
- No tool call is needed; the data lands in Qdrant and your RAG layer sees it.
- Optionally add a small `/agent/intel-sync/run` endpoint if you want manual triggers.

**7) Network rules**

- Outbound only to the feed hosts listed in `feeds.yml`.
- Worker still has no outbound rights.

**Notes**

- This container runs embeddings inside itself (CPU by default). If you have a separate embedding service, point `EMBED_MODEL` and calls accordingly.

---

## 5. Adding new agents safely

- **Scope narrowly**: one agent per job (tickets, docs-only, calendar, etc.).
- **Allowlist everything**: URLs, recipients, API methods. No arbitrary destinations.
- **Tool schema first**: define the tool name, parameters, and expected JSON before writing code.
- **Logging**: timestamp, run_id, inputs, outputs, URLs hit, errors. Feed high-risk runs to the watchdog.
- **Network rules**: each agent gets its own outbound rules; the worker never does.
- **Review cadence**: keep a short checklist for new agents (allowlist reviewed, logs in place, watchdog hooked, API key scoped).

---

## 5. Logging and watchdog hooks

Both agents should feed into the same safety story:

- Every agent has its own log file, for example:
  - `logs/agents/internet-research.log`
  - `logs/agents/intel-sync.log`
  with lines like:
  - Timestamp.
  - Agent name.
  - Run ID.
  - Inputs (query, feeds).
  - URLs hit.
  - High-level outcome.
- For higher-risk actions (for example, anything that might be used for automation decisions), you can also:
  - Bundle `{user_prompt, agent_inputs, agent_result, final_answer}`.
  - Send that bundle to the watchdog model to score and tag.

The important point: agents are visible. If something weird happens, you can go back and see:

- What the model asked for.
- Which agent ran.
- Where it went on the internet.
- What came back.

---

## 6. Summary

In this runbook, agents are your internet arms:

- The worker stays on a short leash: no raw HTTP, no outbound rights.
- The internet-research agent handles on-demand questions with a tight allowlist and full logging.
- The intel-sync agent keeps a small set of feeds up-to-date inside your local RAG, so many "internet questions" become "local knowledge" over time.

From here you can:

- Add more agents (for example, a "docs-only" agent that only hits your own wiki, or a "ticketing" agent that writes to your helpdesk).
- Tighten or relax allowlists based on your threat model.
- Plug watchdog scoring into the flows that matter most.

But the rule never changes:

> The main model never talks to the internet directly.  
> It only talks to your agents, and your agents only do what you explicitly allow.




