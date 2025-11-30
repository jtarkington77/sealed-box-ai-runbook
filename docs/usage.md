# Usage -- day-to-day flows

This doc is about using the stack after it is up and sealed. No install steps here.

## 1. Assumptions

- You already stood up the stack from `docs/stack-setup.md`.
- The worker, watchdog, vector store, and Open WebUI are reachable on localhost (or through your reverse proxy/tunnel).
- You picked models and context limits from `docs/models.md`.
- Network exposure and auth are handled per `docs/safety-and-isolation.md`.

## 2. Where to talk to it

- **Web UI (Open WebUI)**: `http://127.0.0.1:3000` on the box, or the URL behind your proxy/tunnel.
- **API (OpenAI-compatible)**: `http://127.0.0.1:8000/v1` for the worker. Use the API key you set in the UI/env.
- **Watchdog**: `http://127.0.0.1:8100/v1` for ping/tests; it is wired in by your orchestrator, not for direct chat.

If you expose via a reverse proxy, keep the same paths but use HTTPS and your auth layer.

### 2.0 Initial UI setup (Open WebUI)

1. Go to `http://127.0.0.1:3000`.
2. Create the admin user.
3. Settings -> Connections:
   - Provider: OpenAI-compatible.
   - Base URL: `http://worker-llm:8000/v1` (use the service name from docker-compose; if testing outside compose, use `http://127.0.0.1:8000/v1`).
   - API key: `local-sealed-box` (or the key you set in `stack.env`).
4. Add model entries:
   - Worker: `Qwen/Qwen2.5-32B-Instruct`.
   - Code: `Qwen/Qwen2.5-Coder-14B-Instruct`.
   - Math: `Qwen/Qwen2.5-Math-7B-Instruct` (optional).
5. Save and open a chat; send “hello” to confirm responses.

If the model list is empty, add the names manually as above; the UI does not auto-discover them from vLLM.

Quick rule:
- **Inside compose (Open WebUI container):** use `http://worker-llm:8000/v1`.
- **From host via reverse proxy:** use your proxy URL (for example `https://ai-box.local/v1`) and ensure auth is enforced.

## 2.1 API keys and app hooks

- **UI-side key**: In Open WebUI settings, set the OpenAI base to `http://worker-llm:8000/v1` and an API key (for example `local-sealed-box`). This is used by the UI itself.
- **Service keys**: Create separate API keys for scripts/apps. Store them in env files, not in code:
  ```bash
  export SBX_API_BASE="https://ai-box.yourdomain.com/v1"   # or http://127.0.0.1:8000/v1 locally
  export SBX_API_KEY="your-strong-key"
  ```
- **cURL sanity test**:
  ```bash
  curl $SBX_API_BASE/chat/completions \
    -H "Authorization: Bearer $SBX_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{"model":"Qwen/Qwen2.5-32B-Instruct","messages":[{"role":"user","content":"Hello from API"}]}'
  ```
- **Python example** (requests):
  ```python
  import os, requests
  base = os.environ["SBX_API_BASE"]
  key = os.environ["SBX_API_KEY"]
  payload = {
      "model": "Qwen/Qwen2.5-32B-Instruct",
      "messages": [
          {"role": "system", "content": "You are my sealed-box assistant."},
          {"role": "user", "content": "Summarize the last meeting notes in 3 bullets."},
      ],
  }
  r = requests.post(f"{base}/chat/completions",
                    headers={"Authorization": f"Bearer {key}"},
                    json=payload,
                    timeout=60)
  print(r.json())
  ```
- Point external tools (Obsidian, VS Code extensions, CLI scripts) at the same base/key. Do **not** reuse your UI admin account for automation; give each tool its own key so you can revoke it.

## 3. Running chats and RAG

- Start with a **system prompt** that reflects your rules (for example, "You are my sealed-box assistant. Never assume internet access. Cite sources when present.").
- For RAG-backed questions, make sure the documents are already in the vector store (ingested by your jobs or notes workflow).
- Ask for **short answers plus bullets with sources** to keep context lean.
- When pasting logs or code, prefer smaller batches and ask the model to summarize first, then follow with specific asks.

## 4. Switching models on demand

- Keep the **general worker** as the default in Open WebUI.
- For **code-heavy sessions**, route to your code model (for example, Qwen2.5-Coder) via:
  - A separate OpenAI base URL, or
  - A dedicated "code" model entry in the UI.
- For **math/deep reasoning**, do the same with your math specialist. Load only when needed to save VRAM.
- Note which model answered in your notes/logs so you can compare behavior later.

## 5. Using agents safely

- The worker should never have raw HTTP rights. Use the **internet research agent**:
  - Trigger via tool call (as described in `docs/agents-and-tools.md`).
  - Expect short summary + snippets + sources, not long essays.
  - Keep the allowlist tight; update it intentionally.
- The **intel/doc sync agent** should run on a schedule:
  - Feeds fresh items into RAG.
  - Lets you answer "internet" questions from local data later.
- Always log agent calls (inputs, URLs hit, outputs). Keep those logs alongside watchdog verdicts.

## 6. Daily patterns that work well

- **Notes and outlines**: Draft in the UI, then export/store the final text into your notes repo so RAG can use it later.
- **Log triage**: Paste short excerpts, ask for key findings and follow-ups. Do not dump entire gigabyte logs; pre-filter first.
- **Research**: Ask the worker to call the research agent, then ask for a concise synthesis using only returned sources.
- **Checklists**: Have the model produce short, actionable checklists; store them as markdown for reuse.
- **API-driven workflows**: wire simple scripts to call the API for:
  - Daily summaries of new notes/commits/logs.
  - Ticket templates: send inputs, get back structured YAML/JSON to paste into your system.
  - Doc updates: generate outlines then finalize by hand before ingesting back into RAG.

## 6.1 Hooking into other apps

- **Obsidian / note apps**: use a plugin or a small script to send the current note to `/chat/completions` and append the response. Keep the key in your OS keyring or env, not in the note vault.
- **Terminals / shells**: create a `sbx` shell function that posts stdin to the API with a chosen system prompt (writer, coder, summarizer). Respect context limits.
- **Browsers**: if you must use browser userscripts, hit the reverse-proxied HTTPS URL with a scoped API key; never expose the raw worker port.
- **Automations**: for scheduled jobs (digests, syncs), run them on the same host/subnet so traffic stays local, and log run_id + prompt + result to your ops logs.

## 7. Safety habits

- Treat every session as **local-only** unless you explicitly call an agent.
- Do not paste secrets you would not store on disk; remember logs exist.
- Review watchdog tags for high-risk outputs or large data dumps.
- Rotate API keys used by scripts and keep them per-tool to limit blast radius.

## 8. Quick troubleshooting

- If responses are slow:
  - Check GPU/CPU usage (`nvidia-smi`, `docker stats`).
  - Reduce context size or switch to a smaller quantization.
- If RAG answers are bad:
  - Re-run ingestion for the relevant docs.
  - Inspect vector store for duplicates or broken chunks.
  - Simplify the prompt: ask for "use only the provided sources; if none, say so."
- If agent calls fail:
  - Check agent logs for blocked domains or auth errors.
  - Verify firewall allowlist for the agent container only.
- If the UI cannot reach the worker:
  - Confirm the base URL and API key in Open WebUI.
  - Ensure ports are still bound to localhost and not blocked by host firewall.

### 8.1 Debug checklist (commands)

```bash
# Worker up?
curl -sf http://127.0.0.1:8000/v1/models

# Watchdog up?
curl -sf http://127.0.0.1:8100/v1/models

# UI to worker path ok?
curl -sf http://worker-llm:8000/v1/models  # from inside the compose network

# Agents up?
curl -sf http://internet-research:8080/agent/internet-research -X POST -d '{"query":"ping"}' -H "Content-Type: application/json"

# GPU/CPU
nvidia-smi
docker stats --no-stream
```
