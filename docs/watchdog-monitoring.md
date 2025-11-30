# Watchdog monitoring -- how to wire and read it

This doc explains how the watchdog model observes the system, what to log, and how to review it.

## 1. Purpose and scope

- The watchdog is a **small, fast instruct model** that scores interactions.
- It is not a user-facing chat model; it tags risk, policy breaks, and odd behavior.
- It should see enough context to judge, but not the entire raw payload if that would leak secrets.

## 2. What the watchdog sees

Feed it structured bundles such as:

- User prompt (redacted if needed).
- Retrieved context snippets (titles/ids, not entire documents unless safe).
- Tool calls used (names + arguments).
- Agent results (summary + URLs hit).
- Final answer returned to the user.

Pass a `run_id` or similar correlation id so you can tie model logs, agent logs, and watchdog verdicts together.

## 3. Scoring and tags (example scheme)

Return a small JSON payload like:

```json
{
  "run_id": "2025-02-01T12-30-00Z-abcd",
  "risk_level": "low|medium|high",
  "reasons": ["possible_data_exfil", "destructive_command", "off_policy"],
  "notes": "Short free-text note",
  "sources_seen": ["docs/notes/week12.md", "agent:internet-research"]
}
```

Suggested tags to start with:

- `possible_data_exfil` -- large outbound-sounding replies, dumps of sensitive info.
- `destructive_command` -- rm -rf, firewall flushes, mass deletes.
- `jailbreak_probe` -- prompt injection attempts, requests to disable safeties.
- `out_of_policy` -- user asks for things outside intended scope.
- `agent_anomaly` -- agent results look empty, irrelevant, or mismatched.

Adjust to your threat model; keep the list small so it stays actionable.

## 4. Where to run it

- Run the watchdog as a **separate container/endpoint** (for example `http://127.0.0.1:8100/v1`).
- Your orchestrator should call it **after** the worker produces an answer (or after an agent call) to avoid blocking user latency too much.
- Keep it on GPU if you can spare a few GB; CPU is acceptable if the model is tiny and latency is tolerable.

### 4.1 Container/compose sketch

```yaml
watchdog-llm:
  image: vllm/vllm-openai:latest
  command: >
    --model microsoft/Phi-3.5-mini-instruct
    --host 0.0.0.0
    --port 8100
    --max-model-len 4096
    --gpu-memory-utilization 0.3
  ports:
    - "127.0.0.1:8100:8100"
  volumes:
    - ./data/models:/root/.cache/huggingface
  restart: unless-stopped
```

### 4.2 Orchestrator call pattern

After the worker responds (or after an agent returns), send a compact bundle. Wire this into your orchestrator (UI middleware, custom router, or Open WebUI webhooks) so every turn flows through the watchdog.

```bash
curl http://127.0.0.1:8100/v1/chat/completions \
  -H "Authorization: Bearer local-watchdog" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "microsoft/Phi-3.5-mini-instruct",
    "messages": [
      {"role": "system", "content": "You score risk. Respond with JSON only."},
      {"role": "user", "content": "RUN_ID=2025-02-01T12-30Z-abcd\nPROMPT_SUMMARY=..."},
      {"role": "assistant", "content": "ANSWER_SUMMARY=..."},
      {"role": "user", "content": "TOOLS_USED=internet_research,agent:intel-sync"}
    ],
    "response_format": {"type": "json_object"}
  }'
```

Parse the JSON and log it with the run_id. Store failures/timeouts as separate log records so you know when the watchdog missed a turn.

### 4.3 Simple middleware bridge (Python example)

If you do not have a full orchestrator, a small bridge can sit in front of the worker. Place this in `~/sealed-box-ai/stack/watchdog-bridge/` as `app.py` and add it to docker-compose. **Keep the bridge internal** (compose network only) and front it with the same reverse proxy/auth you use for the UI/API; do not expose it directly to the internet.

```python
import os, requests, json
from fastapi import FastAPI

WORKER = "http://worker-llm:8000/v1/chat/completions"
WATCHDOG = "http://watchdog-llm:8100/v1/chat/completions"
API_KEY = "local-sealed-box"

app = FastAPI()

@app.post("/chat")
def chat(body: dict):
    # 1) Call worker
    w = requests.post(WORKER,
        headers={"Authorization": f"Bearer {API_KEY}"},
        json=body, timeout=60)
    worker_resp = w.json()
    # 2) Send summary to watchdog
    run_id = body.get("run_id","unknown")
    watchdog_body = {
      "model": "microsoft/Phi-3.5-mini-instruct",
      "messages": [
        {"role":"system","content":"Score risk. JSON only."},
        {"role":"user","content": f"RUN_ID={run_id}\nPROMPT_SUMMARY={body['messages'][-1]['content'][:400]}"},
        {"role":"assistant","content": worker_resp['choices'][0]['message']['content'][:400]},
      ],
      "response_format":{"type":"json_object"}
    }
    try:
        wd = requests.post(WATCHDOG, json=watchdog_body, timeout=10)
        verdict = wd.json()
    except Exception as e:
        verdict = {"error": str(e)}
    # 3) Log (append to file)
    with open("/logs/watchdog/bridge.log","a",encoding="utf-8") as f:
        f.write(json.dumps({"run_id":run_id,"verdict":verdict})+"\n")
    return worker_resp
```

Containerize this bridge, point your UI at `/chat`, and keep worker/watchdog behind it. This gives you end-to-end watchdog coverage without rewriting the UI.

### 4.4 Compose entry for the bridge

```yaml
  watchdog-bridge:
    build: ./watchdog-bridge
    environment:
      API_KEY: local-sealed-box
    volumes:
      - ./logs/watchdog:/logs/watchdog
    networks:
      - sealed_ai_net
    restart: unless-stopped
    ports:
      - "127.0.0.1:9000:8000"
```

Set your UI/API base URL to `http://watchdog-bridge:8000/chat` from inside compose. If you need host access, front `127.0.0.1:9000/chat` with your existing reverse proxy + auth (see safety-and-isolation.md) instead of exposing it directly.

## 5. Logging and storage

- Store watchdog outputs under a dedicated path (for example `logs/watchdog/` with one file per day).
- Include timestamp, run_id, model name, and the JSON verdict.
- Do not store full prompts if they contain sensitive data; store hashes/ids instead.
- Rotate logs and include them in your backup plan (`docs/operations.md`).

### 5.1 Minimal log record (NDJSON)

```json
{"ts":"2025-02-01T12:30:00Z","run_id":"2025-02-01T12-30Z-abcd","model":"microsoft/Phi-3.5-mini-instruct","risk_level":"low","reasons":[],"notes":"Normal answer","sources_seen":["notes/week12.md"]}
```

## 6. Review and response

- Build a simple query workflow (grep/jq or a small dashboard) to answer:
  - Show me everything tagged `high` this week.
  - Show me all `destructive_command` tags.
  - Show me all runs that involved the internet agent.
- When you see a suspicious run:
  - Pull the corresponding agent and application logs using the same `run_id`.
  - Decide whether to tighten allowlists, adjust prompts, or revoke API keys.
- Keep a short **response playbook** so you are not improvising during an incident.

### 6.1 Example jq queries

- High risk this week:
  ```bash
  jq 'select(.risk_level=="high")' logs/watchdog/watchdog-$(date +%Y-%m-%d).log
  ```
- Anything involving agents:
  ```bash
  jq 'select(.sources_seen[]? | contains("agent"))' logs/watchdog/watchdog-*.log
  ```

## 7. Performance tips

- Keep prompts compact: summaries, IDs, and tool names beat full transcripts.
- Batch watchdog calls if your orchestrator supports it (for example, evaluate multiple runs in one request) to save overhead.
- If latency matters, set a **hard timeout** for watchdog calls and log timeouts separately.
- Periodically test the watchdog with known-bad prompts to ensure it still flags them.
