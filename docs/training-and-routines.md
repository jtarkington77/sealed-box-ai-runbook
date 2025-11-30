# Training and routines -- feeding the system safely

This doc covers how to grow the stack's knowledge without leaking or ruining your data.

## 1. Principles

- Default to **RAG ingestion** first; it is reversible and lower risk.
- Keep **training/fine-tuning** rare, deliberate, and on scrubbed data.
- Separate **source data**, **ingested chunks**, and **model artifacts** so you can roll back any step.
- Log what you ingest and when, so you can audit later.

## 2. Data hygiene: what to include

Safer inputs (good RAG candidates):

- Your own notes, lab writeups, project docs, KB exports.
- Public documentation without PII or contractual data.
- Summaries you or the model wrote locally.

High-risk inputs (avoid or isolate heavily):

- Email archives, client/customer data, HR/financial/medical info.
- Anything that would be a reportable incident if leaked.
- Unvetted third-party dumps.

If you must handle risky data:

- Keep it on an **encrypted volume**.
- Ingest into a **separate collection** with stricter access.
- Do not echo contents into verbose logs.

## 3. RAG ingestion pipeline (baseline)

1. **Collect sources** into a staging folder (markdown, txt, PDFs after OCR, etc.).
2. **Normalize**:
   - Convert to UTF-8 text.
   - Split into reasonable chunks with overlap (tunable per document type).
   - Add metadata (title, path, timestamp, tags).
3. **Embed and upsert** into the vector store (Qdrant):
   - Use your chosen embedding model (see `docs/models.md`).
   - Keep collections by domain (notes, labs, intel, docs) to avoid cross-talk.
4. **Verify**:
   - Run a couple of test queries.
   - Spot-check stored payloads for PII and formatting.
5. **Log** what was ingested (source path, collection, timestamp, embedding model version).

Automate steps 2-5 with a small script or job runner; keep config in version control.

### 3.1 Minimal ingestion script shape (Python/httpx)

Save this as `~/sealed-box-ai/stack/ingest.py` (or another clear name) so you can run it from the stack folder:

```python
import os, pathlib, requests, json
from sentence_transformers import SentenceTransformer

QDRANT_URL = os.environ["QDRANT_URL"]          # e.g. http://127.0.0.1:6333
COLLECTION = os.environ.get("QDRANT_COLLECTION","notes")
EMBED_MODEL = os.environ.get("EMBED_MODEL","BAAI/bge-m3")
chunk_size = 800
overlap = 120

model = SentenceTransformer(EMBED_MODEL)
# This uses local CPU/GPU via sentence-transformers. If you prefer an embedding service, swap this call to hit it instead.

def chunks(text, size, overlap):
    out = []
    start = 0
    while start < len(text):
        out.append(text[start:start+size])
        start += size - overlap
    return out

def upsert(payloads):
    points = []
    for idx,p in enumerate(payloads):
        vec = model.encode(p["text"]).tolist()
        points.append({"id": p["id"], "vector": vec, "payload": p})
    requests.put(f"{QDRANT_URL}/collections/{COLLECTION}/points",
                 headers={"Content-Type":"application/json"},
                 data=json.dumps({"points": points}),
                 timeout=30).raise_for_status()

def ingest_file(path: pathlib.Path):
    text = path.read_text(encoding="utf-8", errors="ignore")
    payloads = []
    for i,chunk in enumerate(chunks(text, chunk_size, overlap)):
        payloads.append({
            "id": f"{path.name}-{i}",
            "text": chunk,
            "source": str(path),
            "title": path.stem,
        })
    upsert(payloads)

if __name__ == "__main__":
    for f in pathlib.Path("./staging").glob("*.md"):
        ingest_file(f)
```

Wrap this with logging and a `run_id` so you can audit later. Use separate collections for notes/labs/intel.

### 3.2 Quick Qdrant collection init

```bash
curl -X PUT http://127.0.0.1:6333/collections/notes \
  -H "Content-Type: application/json" \
  -d '{"vectors":{"size":1024,"distance":"Cosine"}}'
```

Adjust `size` to your embedding model output dimension.

### 3.3 End-to-end first ingestion (commands)

```bash
cd ~/sealed-box-ai/stack
python -m venv .venv && source .venv/bin/activate
pip install sentence-transformers requests
mkdir -p staging
echo "Test note content" > staging/test.md
export QDRANT_URL="http://127.0.0.1:6333"
export QDRANT_COLLECTION="notes"
export EMBED_MODEL="BAAI/bge-m3"
python ingest.py   # whatever you name the script above
curl -s http://127.0.0.1:6333/collections/notes/points/count
```

If the count increases, ingestion worked. Move your real markdown/txt/PDF-to-text files into `staging/` and re-run.

## 4. Daily/weekly routines

Daily:

- Add new notes or markdown exports to the staging folder.
- Run a quick ingestion job (could be a cron/systemd timer).
- Review watchdog/agent logs for anything unusual.

Weekly:

- Pull updates from your intel/doc agent into RAG.
- Re-index large docs that changed.
- Rotate API keys used by scripts.
- Snapshot the vector store (see backups below).

Monthly or after major changes:

- Evaluate whether the worker model still fits your workflows.
- Prune stale collections; archive old chunks.
- Refresh the embedding model if you change families or versions.

## 5. Optional fine-tuning/LoRA

- Only consider after you have **good RAG coverage** and clear gaps that tuning can fill.
- Use **synthetic or scrubbed** data; avoid anything sensitive.
- Keep tuning jobs **offline**; do not push to hosted endpoints.
- Track:
  - Base model version.
  - Training data source and size.
  - Hyperparameters and epochs.
  - Validation set and results.
- Store resulting adapters on an isolated, versioned path. Document how to roll back to the base model.

### 5.1 LoRA quick start outline (example with Axolotl/peft-style tools)

1. **Prep data**: JSONL with `{"instruction": "...", "input": "...", "output": "..."}`. Scrub PII.
2. **Config file** (example `train.yml`):
   ```yaml
   base_model: Qwen/Qwen2.5-32B-Instruct
   lora:
     r: 16
     alpha: 32
     dropout: 0.05
   dataset:
     path: ./data/train.jsonl
   training:
     epochs: 2
     micro_batch_size: 4
     lr: 1e-4
   output_dir: ./artifacts/lora-qwen32b-safe
   ```
3. **Run** (on the sealed box, offline):
   ```bash
   axolotl train.yml
   ```
4. **Test**: load the base model + adapter in your inference stack; run your golden prompts.
5. **Roll back**: keep a copy of the base model path in config; if quality drops, disable the adapter.

If hardware is tight, downsize to a 14B/7B base; keep adapters small and specific to one task.

If you are new: stick to RAG first. Fine-tuning is optional and higher-risk; most “make it smarter on my data” needs are covered by ingesting your own content into RAG.

### 5.2 Loading the adapter in vLLM (example)

Add to worker command:
```
--model Qwen/Qwen2.5-32B-Instruct --adapter /path/to/artifacts/lora-qwen32b-safe
```
Then restart the worker container and test with your golden prompts.

If in doubt, prefer prompt/RAG changes over tuning.

## 6. Backups and retention

- **Vector store**: snapshot `data/qdrant/` (or your chosen path) regularly; test restores.
- **Ingestion configs/scripts**: keep in git; back up alongside env files.
- **Model weights/adapters**: store checksums; mirror to offline storage if space allows.
- **Logs** (watchdog, agents, ingestion): rotate, timestamp, and back them up with the rest of the stack.

Keep a short **restore runbook**: how to bring back the stack plus the latest vector snapshot on a fresh host.

## 7. Measuring progress

- Maintain a small **golden set** of queries per domain (notes, labs, intel).
- After each ingestion or model change, run the set and record:
  - Answer quality.
  - Latency.
  - Whether sources were cited.
- If quality regresses, revert the last ingestion/model change and re-test.
