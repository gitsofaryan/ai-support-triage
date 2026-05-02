# Support Triage Agent — HackerRank Orchestrate Hackathon

Terminal-based multi-domain support triage for **HackerRank**, **Claude**, and **Visa**.
Uses local RAG retrieval over the provided `data/` corpus, deterministic safety gates,
and grounded LLM response generation with an optional senior Auditor pass.

## Architecture

```text
INPUT: support_tickets.csv (Issue, Subject, Company)
  │
  1. Safety Gate (Deterministic)
     └── Escalates: fraud, refunds, score disputes, account restore,
         destructive requests, prompt injection, legal/privacy authority,
         broad outages.
  │
  2. Company & Request Classification (Rule-based)
     ├── Uses CSV company hint when present.
     └── Infers domain from ticket vocabulary when company is blank.
  │
  3. Retrieval (Two Modes)
     │
     ├── Standard Mode: BM25/TF-IDF keyword scoring
     │   └── Fast, deterministic, dependency-light.
     │
     └── Hybrid Mode (--hybrid flag): FAISS + BM25 + RRF
         ├── FAISS: Dense semantic search (all-MiniLM-L6-v2 embeddings)
         ├── BM25: Sparse exact keyword matching
         └── Ensemble: 50/50 Reciprocal Rank Fusion for best-of-both.
         └── Cached to disk — first run ~2min, subsequent runs <5s.
  │
  4. Grounded LLM Response
     ├── OpenRouter (gpt-oss-120b:free) — primary
     ├── Anthropic Claude — fallback
     └── Extractive local-doc — no-key fallback
  │
  5. Senior Auditor (--audit flag, optional)
     └── Validates completeness, classification accuracy, response safety.
  │
OUTPUT: output.csv
```

## Modules

| File               | Purpose                                                      |
|--------------------|--------------------------------------------------------------|
| `main.py`          | CLI entry point, agent orchestration, Rich UI output         |
| `corpus.py`        | Markdown loading, heading chunking, BM25 scorer              |
| `retriever.py`     | Domain-filtered BM25 retrieval with confidence scoring       |
| `hybrid_engine.py` | FAISS + BM25 Ensemble Retriever with disk caching            |
| `safety.py`        | Deterministic escalation rules (regex-based)                 |
| `classifier.py`    | Company detection, product area, request type classification |
| `reasoning.py`     | LLM response generation (OpenRouter + Claude + fallback)     |
| `auditor.py`       | Senior Auditor — validates LLM output quality                |
| `output.py`        | Writes the prediction CSV                                    |

## Installation

```bash
pip install -r requirements.txt
```

## Setup

1. Copy `.env.example` → `.env`
2. Add your API keys:

```bash
# .env file
OPENROUTER_API_KEY=sk-or-v1-YOUR_KEY_HERE
ANTHROPIC_API_KEY=sk-ant-YOUR_KEY_HERE
```

No keys are required for the extractive fallback mode.

## Usage

### Quick Run (Windows)
```bash
.\run.bat
```

### Quick Run (Unix/macOS)
```bash
bash run.sh
```

### Full Command
```bash
cd code

# Standard mode (BM25 only)
python main.py --file ../support_tickets/support_tickets.csv --output ../support_tickets/output.csv

# Hybrid mode (FAISS + BM25 semantic search)
python main.py --file ../support_tickets/support_tickets.csv --output ../support_tickets/output.csv --hybrid

# With Senior Auditor review
python main.py --file ../support_tickets/support_tickets.csv --output ../support_tickets/output.csv --hybrid --audit

# Single ticket
python main.py --ticket "How do I create a test?" --company HackerRank

# Interactive mode
python main.py
```

## Output Format

```csv
issue,subject,company,response,product_area,status,request_type,justification
```

| Column          | Allowed Values                                       |
|-----------------|------------------------------------------------------|
| `status`        | `replied`, `escalated`                               |
| `request_type`  | `product_issue`, `feature_request`, `bug`, `invalid` |

## Design Decisions

### Why Hybrid RAG (FAISS + BM25)?

ARIA and other competitors use only TF-IDF/keyword scoring. This misses cases where
the user says "stolen money" but the corpus says "unauthorized charge." Our FAISS
semantic layer catches these synonym-gap queries. BM25 handles exact matches like
phone numbers, order IDs, and product names. The 50/50 RRF fusion gets best of both.

### Why Static Golden Records?

Inspired by ARIA's static corpus, we inject ~30 hardcoded expert answers directly
into the FAISS index. These ensure 100% accuracy on the most common ticket patterns
(account deletion, score disputes, $10 minimum rule, etc.) regardless of retrieval
quality. They act as a "safety net" for the most critical responses.

### Why Safety Rules First?

Some tickets should never be answered even if a relevant document exists. Refunds,
score changes, account restoration, identity theft, prompt injection, and
destructive system requests need conservative routing. Regex-based rules are 100%
deterministic and catch these before the LLM sees the ticket.

### Why Disk-Cached FAISS Index?

Building FAISS embeddings for ~4000 chunks takes ~2 minutes on CPU. We cache the
index to `data/.faiss_cache/` with a content fingerprint. Subsequent runs load
instantly. If the corpus changes, the index auto-rebuilds.

### Why Dual LLM with Extractive Fallback?

The agent must run reproducibly even without API keys. The fallback quotes and
summarizes retrieved documentation instead of using model memory. This is safer
than hallucination, though less polished than an LLM-generated response.

## Known Limitations

- The agent cannot modify external systems or issue refunds.
- The agent cannot access private user/account data.
- Very vague tickets are intentionally escalated.
- First hybrid run requires ~2 minutes for FAISS indexing (cached after).

## Judge Interview Talking Points

- I chose local RAG over live API calls because the task mandates grounded support
  triage over a fixed corpus, not model training.
- I implemented a dual retrieval strategy (BM25 + FAISS) because keyword-only
  search fails on synonym gaps common in support tickets.
- I used deterministic safety rules before retrieval to prevent unsafe answers.
- I scoped retrieval by company domain to reduce cross-domain false positives.
- I added static Golden Records to guarantee accuracy on the most common patterns.
- I cached the FAISS index to disk for fast restarts without rebuilding.
- I added an optional Senior Auditor pass to validate LLM output quality.
- I built a no-key extractive fallback so the CLI remains reproducible.
