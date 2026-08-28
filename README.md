# ModelSmithAI (Standalone)

**Your Autonomous Model Factory — describe what you want to recognize in plain English, and six AI agents gather data, train a model, and deliver a security-scanned classifier with an audit report.**

This is the standalone version: pure Python agents + a web UI. No external platform or framework required.

---

## What it does

Type a request like `husky vs wolf vs malamute`. Six coordinated agents then:

1. **Planner** — parse the request into a class list, search queries, and a target accuracy (local LLM, with a deterministic fallback).
2. **Scout** — fetch open-licensed images from Openverse & Wikimedia, recording each image's source and license; tops up until enough clean images survive.
3. **Curator** — verify each image with zero-shot CLIP, deduplicate, and cache embeddings.
4. **Trainer** — train a logistic head on the cached CLIP embeddings; report accuracy, per-class recall, and confusion pairs.
5. **Diagnostician** — decide STOP (target met) or reason about which classes are failing and issue a structured re-fetch request (local LLM, with a deterministic fallback).
6. **Sentinel** — scan the final model for malicious pickle opcodes (without loading it) and convert it to the safe SafeTensors format.

Every build auto-generates a **PDF audit report** (accuracy progression, data provenance with licenses, agent decision log, security verdict).

---

## Architecture

```
WEB APP (website/index.html)  --HTTP-->  PYTHON SIDECAR (sidecar/app.py, FastAPI :8000)
                                              |  runs the six agents via orchestrator.py
                                              v
                                   SHARED STORAGE (storage/)
                          images/ · model.pt · model.safetensors · state.json · audit_report.pdf
```

The website talks only to the local Python backend over HTTP. Everything runs on your machine.

---

## Tech Stack

- **Backend:** Python 3.12, FastAPI, Uvicorn
- **Vision:** OpenCLIP `ViT-B-32` → scikit-learn `LogisticRegression`; PyTorch
- **LLM reasoning:** Ollama (local, `qwen2.5:3b`) — used by the Planner and Diagnostician, each with a deterministic fallback so it never blocks
- **Data sources:** Openverse, Wikimedia Commons
- **Security:** `pickletools` (load-free scan) + `safetensors` (safe conversion)
- **Reporting:** ReportLab
- **Frontend:** static HTML + vanilla JavaScript (no build step)

---

## Setup

**Prerequisites:** Python 3.12, and (optional but recommended) [Ollama](https://ollama.com) for LLM reasoning. A GPU is optional; CPU works.

### 1. Python backend

```bash
cd sidecar
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1        # Windows
# source .venv/bin/activate         # macOS/Linux
pip install -r requirements.txt
```

This pulls in PyTorch and OpenCLIP, so the first install is a large download. The CLIP `ViT-B-32` weights are fetched once on the first build and cached afterwards.

### 2. (Optional) Local LLM

```bash
ollama pull qwen2.5:3b
```

If Ollama is not running, the Planner and Diagnostician automatically fall back to deterministic logic — the system still works, just with simpler reasoning.

### 3. (Optional) Choose where runs are stored

Images, models, run state and audit reports are written under `~/.modelsmith/storage`. Point that anywhere writable with the `MODELSMITH_STORAGE` environment variable:

```bash
# Windows (PowerShell)
$env:MODELSMITH_STORAGE = "D:/modelsmith/storage"

# macOS / Linux
export MODELSMITH_STORAGE=~/modelsmith/storage
```

Set it in the same shell that runs the sidecar (or the CLI), so both read the same root.

---

## Running

**Option A — Web UI (recommended)**

Terminal 1 (backend):
```bash
cd sidecar
.\.venv\Scripts\Activate.ps1
uvicorn app:app --port 8000 --reload
```

Terminal 2 (website):
```bash
cd website
py -3.12 -m http.server 5500
```

Open **http://localhost:5500**, type a request (e.g. `cat vs dog`), set images/class, and click **Build model**. Watch the agents stream live and download the audit report when done.

**Option B — Command line**

```bash
cd sidecar
.\.venv\Scripts\Activate.ps1
python orchestrator.py "husky vs wolf vs malamute" --per-class 12 --target 0.85
```

---

## Configuration

| Parameter | Default | Meaning |
|---|---|---|
| `request` | — | natural-language description of the classes |
| `images_per_class` | 12 | clean images to gather per class |
| `target_accuracy` | 0.85 | loop stops when overall accuracy ≥ this |
| `max_iterations` | 3 | hard cap on the self-improving loop |
| `KERNELS_LLM` (env) | `qwen2.5:3b` | Ollama model for the Planner/Diagnostician |
| `MODELSMITH_STORAGE` (env) | `~/.modelsmith/storage` | where images, models, state and reports are written |

---

## Security highlights

- Disassembles pickle bytecode **without loading** the file; flags dangerous opcodes against a torch-only allowlist.
- Inspects every archive entry **by pickle signature, not filename** (defeats extension-hiding attacks, e.g. CVE-2025-1889).
- Converts to **SafeTensors** (loaded `weights_only=True`), a format with no code-execution path — so the delivered model is safe by construction.

---

## Known limitations

- Accuracy is bounded by open-data availability; niche classes may plateau below target (the loop stops honestly).
- The live accuracy climb isn't guaranteed — it depends on re-fetches finding better images.
- Easy tasks (cat vs dog) can saturate at 100% in one iteration.
- Classification only (no detection/video/audio).

---

## File guide

```
sidecar/
  app.py            FastAPI server: endpoints + async build jobs + CORS
  orchestrator.py   the loop; shared state; clean-image top-up
  plan.py           Planner (LLM + fallback)
  scout.py          Scout (multi-source fetch, offsets, robust re-fetch)
  curate.py         Curator (CLIP verify + dedup + embedding cache)
  train_real.py     Trainer (embeddings -> logistic head -> model.pt)
  diagnose.py       Diagnostician (LLM + fallback + STOP-at-target)
  sentinel.py       Sentinel (pickle scan + SafeTensors convert)
  report.py         audit PDF generator
  requirements.txt
website/
  index.html        single-page web UI (live agent relay + accuracy chart + model card)
```

---

*ModelSmithAI — from a sentence to a secure model.*
