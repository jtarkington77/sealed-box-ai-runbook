# Models & hardware – picking your stack

This doc is where we stop saying “big model, small model” and actually talk about:

- What different model **roles** are in this stack.
- How parameter count, context, and training data change behavior.
- Concrete model families that are good at:
  - Everyday assistant work.
  - Coding.
  - Math / deep reasoning.
  - Guarding / watchdog.
  - Embeddings for RAG.
- What is realistic on different GPU tiers.

By the end you should be able to:

- Pick a **worker** model that fits your GPU and your workflow.
- Pick a **watchdog** that can keep up.
- Add optional **code** and **math/research** specialists.
- Choose an **embedding** model that doesn’t suck.

---

## 1. Quick mental model: how these things actually work

You don’t need transformer math, but you do need a few concepts straight:

### 1.1 Tokens & context window

Models read text as **tokens** (chunks of words / punctuation). The **context window** is how many tokens they can “see” at once:

- 4k–8k tokens → fine for chat, short notes.
- ~32k+ → you can start throwing real docs and logs at it.
- 100k+ → long reports, multi-doc workflows, “read this whole folder” type stuff.

If your context window is tiny, RAG still helps, but you’ll constantly fight truncation.

### 1.2 Parameters (B = billion)

Parameter count is a rough capacity indicator:

- 3B–8B: “small” – fine for simple tasks, starts to fall apart on complex chains of thought.
- 13B–16B: **floor** for a general assistant that doesn’t feel like a toy.
- 30B–70B: where open-weight models start feeling like real workers for serious workflows.
- MoE (Mixture-of-Experts) like Mixtral:
  - 8×7B behaves more like a much larger dense model in many tasks.

More params → more VRAM, slower, but usually better reasoning and robustness.

### 1.3 Base, instruct, code, math, embeddings, watchdogs

You’ll see model labels like:

- **Base** – raw pretrain, not aligned for instructions. Good starting point for fine-tuning, bad as a “just talk to it” assistant.
- **Instruct / Chat** – tuned to follow instructions. This is what you want for:
  - Worker.
  - Watchdog (small instruct).
- **Code** – trained heavily on code + comments. Better at:
  - Filling in functions.
  - Fixing scripts.
  - Understanding stack traces.
- **Math / reasoning** – tuned on math proofs, competitions, theorems.
  - Overkill for daily chat.
  - Excellent for heavy problem solving if you have the hardware.
- **Embedding models** – not chat at all. Text → vector. Used for RAG.
- **Guard / safety / watchdog** – small models used to:
  - Classify content.
  - Score for risk.
  - Enforce simple policies.

### 1.4 Quantization: how we cram big models into smaller GPUs

You will almost certainly not run everything in FP16.

- **FP16 / BF16** – high quality, heavy VRAM.
- **8-bit / 4-bit quantization (GGUF, EXL2, etc.)**:
  - Compress weights to fit models into smaller cards.
  - Slight quality hit, sometimes noticeable, sometimes not.
  - Lets you run “too big” models on 24–48 GB cards by trading speed and purity.

For this stack, assume:

- Workers are usually **quantized**.
- Watchdog and embeddings can often be smaller / lighter.

---

## 2. My sealed-box reference stack (example)

This whole repo is written from the perspective of a box that looks roughly like this:

- **GPU:** RTX A6000-class (48 GB VRAM).
- **CPU:** i9-13xxx-class.
- **RAM:** 128 GB.
- **Storage:** multiple TB of NVMe.

On that kind of hardware, a **realistic sealed-box stack** looks like:

- **Worker (general assistant):**
  - A ~30B-class *instruct* model from one of the modern families  
    (Llama 3.x / Qwen 2.5 / Mixtral-class).
- **Code specialist:**
  - A ~14B *code-tuned* model (Qwen2.5-Coder / DeepSeek-Coder-class).
- **Math / deep-reasoning specialist (optional):**
  - A math-tuned model (Qwen2.5-Math / DeepSeek-Math / Llemma-class).
- **Watchdog:**
  - A 3–4B “small but smart” instruct model (Phi-style SLM) for scoring and tagging.
- **Embeddings:**
  - A modern embedding model (BGE-M3 or nomic-embed-text-v1).

I’m not listing the exact checkpoints I run because I swap them fairly often, but everything below is chosen to be **real, reproducible equivalents** that will behave similarly on hardware in this ballpark.

If your GPU is smaller, you’ll use **smaller / more aggressively quantized variants** of the same families and accept some trade-offs.

---

## 3. Roles in this repo’s model lineup

For this runbook, think in terms of **jobs**, not just “a model.”

### 3.1 Worker (main assistant)

This is the model that:

- Answers your questions.
- Drives your agents.
- Writes drafts, summaries, notes, tickets, study guides.

You care about:

- Reasoning quality.
- Ability to follow instructions cleanly.
- Not freaking out when you give it messy input (logs, notes, mixed formats).

### 3.2 Watchdog (oversight / guardrail)

Separate, smaller model that:

- Gets a view of:
  - The **prompt** (possibly redacted).
  - The **retrieved context**.
  - The **worker’s answer** (or a summary of it).
- Tags interactions for:
  - Potential data exfiltration (huge dumps of sensitive data).
  - Dangerous commands / scripts / configs.
  - Probing / jailbreak / “escape” behavior.
- Writes **structured log records** so you can actually query:
  - “Show me everything tagged high-risk this week.”
  - “Show me all commands that looked destructive.”

You care about:

- Speed.
- Stability.
- Good classification / scoring, not fancy writing.

### 3.3 Code specialist (optional)

Used when you’re:

- Writing / reading PowerShell, Bash, Python, C#, etc.
- Debugging scripts.
- Building automation around this stack.

You care about:

- Syntax correctness.
- Understanding of libraries / APIs.
- Ability to refactor / explain code.

### 3.4 Math / deep reasoning specialist (optional)

Used for:

- Serious math problems.
- Proof-like reasoning.
- Very complex chains of thought where general models start to hallucinate or hand-wave.

You care about:

- Actual reasoning, not vibe-based answers.
- Tool-integrated reasoning (CoT + calculators, search, etc.) if you wire it.

### 3.5 Embedding model

Used only for:

- Turning text into vectors for RAG.
- Powering “find related docs / notes / logs” queries.

You care about:

- Retrieval quality.
- Multi-lingual support if you need it.
- Context window for ingesting larger documents.

---

## 4. Model families by job (and what they’re actually good at)

Below are **families** you can build on. You don’t need all of them – pick the ones that match how you actually work.

---

### 4.1 General assistant / worker (13B+ recommended)

These are your “talk to me, help me work” models.

#### Llama 3.x family

- Example checkpoints:
  - `meta-llama/Meta-Llama-3-8B-Instruct`
  - `meta-llama/Llama-3.1-8B-Instruct`
  - `meta-llama/Llama-3.1-70B-Instruct`
- Good for:
  - Everyday assistant work (notes, writing, summarizing).
  - General reasoning.
  - Multi-language workflows (3.1 improves this).
- Watch for:
  - 8B is usable but still small; treat it as an **entry tier**, not an end-game worker.
  - 70B-class needs serious VRAM and/or heavy quantization.

Use this family if you want something mainstream, well-supported, and balanced.

#### Qwen 2.5 family

- Example checkpoints:
  - `Qwen/Qwen2.5-14B-Instruct`
  - `Qwen/Qwen2.5-32B-Instruct`
  - Long-context variants for big documents and logs.
- Good for:
  - General assistant work.
  - Mixed English + other languages.
  - Strong reasoning for the size.
- Watch for:
  - 7B variants are okay, but again, think of **13B+ as the floor** for your main worker if you can.

Use this if you want a **strong open generalist** with good future expansion (math, code, etc. in the same family).

#### Mixtral family (Mixture-of-Experts)

- Example checkpoint:
  - `mistralai/Mixtral-8x7B-Instruct-v0.1`
- Good for:
  - Heavier reasoning workloads where a simple 7B/13B struggles.
  - Punches above its parameter count because of MoE routing.
- Watch for:
  - VRAM is real: at 4-bit you’re still in the ~22–24 GB VRAM ballpark just for the worker, plus overhead.
  - Needs a decent GPU tier to feel good as a daily driver.

Use this if you want a **serious worker on a 24+ GB card** and are okay with more setup work.

---

### 4.2 Code models

These are for **actual coding**, not just explaining error messages.

#### Qwen2.5-Coder series

- Example checkpoints:
  - `Qwen/Qwen2.5-Coder-7B-Instruct`
  - `Qwen/Qwen2.5-Coder-14B-Instruct`
  - `Qwen/Qwen2.5-Coder-32B-Instruct`
- Good for:
  - Code generation.
  - Code reasoning and fixing.
  - Multi-language support (Python, JS, C#, etc.).
- Watch for:
  - Larger variants want more VRAM; 14B is a good sweet spot if you can afford it.

Good fit if you want a **single modern, actively updated code family**.

#### DeepSeek-Coder-class models

- Example: `deepseek-ai/DeepSeek-Coder-V2` variants.
- Good for:
  - High-end code performance, competitive with big closed models.
- Watch for:
  - Heavier VRAM and infra requirements.
  - Licensing – check if your use is allowed.

Use this if code is a **primary use case** and you have hardware to back it.

---

### 4.3 Math / heavy reasoning models

These are overkill for daily chat, but useful when you’re doing serious math or hardcore reasoning.

#### Qwen2.5-Math series

- Example checkpoints:
  - `Qwen/Qwen2.5-Math-7B-Instruct`
  - `Qwen/Qwen2.5-Math-72B-Instruct`
- Good for:
  - English + Chinese math problems.
  - Chain-of-thought and tool-integrated reasoning.
- Watch for:
  - Meant for math. Don’t use these as your default chit-chat worker.

#### DeepSeek-Math series

- Example checkpoints:
  - `deepseek-ai/DeepSeek-Math` (7B and newer V2 variants).
- Good for:
  - Competition-level math.
  - Theorem-style reasoning.
- Watch for:
  - Research-leaning licenses.
  - Very heavy high-end variants (hundreds of billions of params) that are not realistic on a single homelab GPU.

#### Llemma

- Example checkpoints:
  - `EleutherAI/llemma_7b`
  - `EleutherAI/llemma_34b`
- Good for:
  - Mathematical reasoning, proofs, formal theorem-style work.
- Watch for:
  - Strong math brain, but not a general conversationalist.

Use math models when you’re doing **actual math**, and keep a generalist worker for everything else.

---

### 4.4 Watchdog / guard models

You want something **small but not stupid**.

#### Phi-3 / Phi-3.5 family

- Example checkpoints:
  - `microsoft/Phi-3-mini-4k-instruct`
  - `microsoft/Phi-3.5-mini-instruct`
- Good for:
  - Running on 3–4B params with surprisingly strong reasoning for the size.
  - Classification, scoring, “is this risky?” tasks.
- Watch for:
  - Small context windows on some variants; but that’s usually fine for short summaries of interaction.

You can also use:

- Smaller Llama 3 / Qwen 2.5 instruct variants as watchdogs if you prefer family alignment.

The key idea: **watchdog doesn’t have to be huge**, it just has to be fast and consistent.

---

### 4.5 Embedding models for RAG

These are non-chat models for your vector store.

#### BGE-M3

- Checkpoint: `BAAI/bge-m3`
- Good for:
  - Dense retrieval.
  - Lexical matching.
  - Multi-vector interaction.
  - Multi-lingual retrieval (100+ languages).
- Notes:
  - Strong default if you want to support more than English.

#### nomic-embed-text-v1

- Checkpoint: `nomic-ai/nomic-embed-text-v1`
- Good for:
  - 8k context text encoder.
  - Strong retrieval quality vs common baselines.
- Notes:
  - Simple, solid default for English-heavy workloads.

Both are light enough to run on CPU if your GPU is busy.

---

## 5. Hardware tiers and realistic model stacks

Map these back to what the README calls out for VRAM.

### 5.1 12–16 GB VRAM – “proving ground”

Example cards: 3060 12GB, 4070 12GB, 4060Ti 16GB.

Reality:

- This is where you **prove the architecture**, not conquer the world.

Suggested stack:

- **Worker:**  
  - Llama 3.x 8B Instruct **or** Qwen2.5-7B-Instruct (quantized).
- **Watchdog:**  
  - Phi-3/3.5-Mini-Instruct (3–4B).
- **Code (optional):**  
  - Qwen2.5-Coder-7B-Instruct (loaded as needed).
- **Math (optional / ad-hoc):**  
  - Use a hosted model or load a small Qwen2.5-Math when needed.
- **Embeddings:**  
  - BGE-M3 or nomic-embed-text on CPU.

What this is good for:

- Learning the sealed-box pattern.
- Personal notes, articles, and moderate RAG.
- Light agents and scheduled jobs.

Where it *will* hurt:

- Deep reasoning over huge, messy inputs.
- Heavy DFIR / log workflows.
- Massive context windows.

Treat 7B/8B here as “entry-tier worker” until you can move up.

---

### 5.2 16–24 GB VRAM – “daily driver”

Example cards: 4080 Super, 3090, older 24GB prosumer cards.

Here we can treat **13B+ as the floor** for the main worker.

Suggested stack:

- **Worker:**  
  - Qwen2.5-14B-Instruct **or** Llama-3.x 13B-class instruct (quantized).
- **Watchdog:**  
  - Phi-3.5-Mini-Instruct (always on).
- **Code:**  
  - Qwen2.5-Coder-7B or 14B-Instruct (depending on what fits).
- **Math (optional):**  
  - Qwen2.5-Math-7B-Instruct, loaded on demand for math-heavy sessions.
- **Embeddings:**  
  - BGE-M3 or nomic-embed-text on GPU for faster indexing.

What this is good for:

- Real daily use:
  - Writing, reports, notes, incident documentation.
  - RAG over a meaningful personal knowledge base.
  - Agents for “what changed this week?”, “summarize my lab notes”, etc.

Limits:

- You still have to be careful with:
  - Long-context variants.
  - Very big models (Mixtral, 32B Qwen2.5) – they may run but won’t feel snappy.

---

### 5.3 24+ GB VRAM – “comfort / high-end” (A6000-class)

Example: 4090, 5090, A6000, RTX 6000-class.

Here you can do what this runbook is actually written around.

Suggested stack on an A6000-class rig:

- **Worker (general):**  
  - Qwen2.5-32B-Instruct **or** Mixtral-8x7B-Instruct (4-bit) **or** a larger Llama-3.x instruct variant, depending on where you want to sit on the speed/quality line.
- **Watchdog:**  
  - Phi-3.5-Mini-Instruct (3–4B, always-on).
- **Code:**  
  - Qwen2.5-Coder-14B-Instruct.
- **Math / deep reasoning:**  
  - Qwen2.5-Math-7B+ or DeepSeek-Math-class model, loaded for math sessions.
- **Embeddings:**  
  - BGE-M3 or nomic-embed-text on GPU with room for aggressive batching.

What this unlocks:

- Bigger context windows without everything crawling.
- Multiple agents in parallel:
  - Notes + logs + tickets at the same time.
- “AI as infrastructure”:
  - Daily digests.
  - Weekly lab summaries.
  - RAG across a genuinely large collection of documents.

You still need to watch:

- VRAM when stacking multiple big models at once.
- Latency vs quality – a 32B or MoE worker is powerful, but every agent call has a cost.

---

## 6. Picking models by use case

Use this as a quick cheat sheet.

### 6.1 Everyday assistant / workflow

You mostly:

- Write.
- Summarize.
- Plan projects / labs.
- Use RAG on your own notes.

Good worker options:

- Qwen2.5-14B or 32B-Instruct.
- Llama 3.x 13B or 70B-class (if hardware allows).
- Mixtral-8x7B if you’re on a beefy card.

Pair with:

- Phi-3.5-Mini as watchdog.
- BGE-M3 or nomic-embed for embeddings.

### 6.2 Coding & automation

You mostly:

- Write PowerShell/Bash/Python.
- Build tools around your stack.

Keep your **general worker**, then add:

- Qwen2.5-Coder-7B/14B-Instruct **or** DeepSeek-Coder-class.
- Let:
  - Worker handle natural language.
  - Code model handle heavy code generation/repair.

### 6.3 Math / research / “hard thinking” tasks

You mostly:

- Solve math problems.
- Do deep research or formal reasoning tasks.

Keep your general worker, then add:

- Qwen2.5-Math-7B/72B for math-heavy sessions.
- DeepSeek-Math-class or Llemma if you’re comfortable with their licenses and VRAM cost.

Use them through:

- Agents that explicitly route “math-tagged” tasks to the math specialist.
- Separate endpoints so you know what model is answering what.

### 6.4 Watchdog / safety

No matter what else you choose:

- Keep watchdog small and fast (Phi-family or small Qwen/Llama).
- Run it on:
  - Every chat turn.
  - Every agent call.
  - Every RAG query that touches sensitive data.

Log:

- Risk scores.
- Keywords / tags.
- Any flagged commands or suspicious outputs.

---

## 7. How to actually choose and iterate

When you sit down to pick models:

1. **Start with your VRAM reality**  
   Know exactly what your card can handle with some headroom.

2. **Pick one worker, one watchdog, one embedding model**  
   Don’t overcomplicate your first pass. Get:
   - Worker answering.
   - RAG returning sensible docs.
   - Watchdog tagging interactions.

3. **Add specialists only when needed**  
   - Code model when you actually start doing code work.
   - Math model when you actually need serious math.

4. **Record what’s running**  
   - Write down model names, quantization, and roles.
   - Put that in your own ops doc so you can remember what you deployed.

5. **Change one thing at a time**  
   - Swap worker models and compare behavior on **your** real tasks.
   - Don’t benchmark on synthetic leaderboards alone.

Once you’re comfortable with your stack choice, the rest of the runbook (agents, safety, watchdog wiring) assumes:

- You have at least:
  - One worker.
  - One watchdog.
  - One embedding model.
- You know roughly what each one is good at.

From there, the rest of the docs show you how to wire them into a sealed, observable AI box that actually fits the way you work.
