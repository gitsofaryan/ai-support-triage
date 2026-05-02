# Support Triage Agent

Terminal-based support ticket triage for HackerRank, Claude, and Visa.
The agent uses local retrieval over the provided `data/` corpus, deterministic
safety rules, and grounded response generation via OpenRouter or Claude.

## Architecture

```text
INPUT: support_tickets.csv (Issue, Subject, Company)
  |
  1. Safety gate
     - Escalates fraud, refunds, score disputes, account restore requests,
       destructive requests, prompt injection, legal/privacy authority issues,
       and broad outages.
  |
  2. Company and request classification
     - Uses the CSV company hint when present.
     - Infers HackerRank, Claude, Visa, or unknown for blank company fields.
  |
  3. Local retrieval
     - Loads markdown files from ../data.
     - Chunks by markdown headings.
     - Uses BM25-style scoring.
     - Searches within the detected company corpus when possible.
  |
  4. Grounded response
     - Tries OpenRouter (gpt-oss-120b:free) first if OPENROUTER_API_KEY is set.
     - Falls back to Claude if ANTHROPIC_API_KEY is set.
     - Uses extractive local-doc fallback if no LLM is available.
     - All responses are grounded in retrieved chunks only.
  |
OUTPUT: output.csv
```

## Modules

- `main.py` - CLI entry point and orchestration.
- `corpus.py` - Markdown loading, heading chunking, and BM25 scorer.
- `retriever.py` - Domain-filtered BM25 retrieval.
- `safety.py` - Deterministic escalation rules.
- `classifier.py` - Company, product area, and request type classification.
- `reasoning.py` - LLM response generation (OpenRouter + Claude fallback) plus extractive fallback.
- `output.py` - Writes the prediction CSV.

## Installation

Install dependencies:

```bash
pip install -r requirements.txt
```

Or manually:

```bash
pip install anthropic openai
```

## Setup

1. Copy `.env.example` to `.env`
2. Add your API keys (or set environment variables):

**Option A: Using .env file**
```
OPENROUTER_API_KEY=sk-or-v1-YOUR_KEY_HERE
ANTHROPIC_API_KEY=sk-ant-YOUR_KEY_HERE
```

**Option B: Environment variables**
```powershell
$env:OPENROUTER_API_KEY="sk-or-v1-..."
$env:ANTHROPIC_API_KEY="sk-ant-..."
```

```bash
export OPENROUTER_API_KEY="sk-or-v1-..."
export ANTHROPIC_API_KEY="sk-ant-..."
```

No keys are required for the extractive fallback mode.

## Quick Run

**Windows:**
```bash
.\run.bat
```

**Unix/Linux/macOS:**
```bash
bash run.sh
```

## Usage

From the repo root:

```bash
python code/main.py --file support_tickets/support_tickets.csv --output support_tickets/output.csv
```

Single ticket:

```bash
python code/main.py --ticket "How do I create a test?" --company HackerRank
```

Interactive mode:

```bash
python code/main.py
```

## Output Format

The writer preserves the existing starter output shape:

```csv
issue,subject,company,response,product_area,status,request_type,justification
```

Allowed values:

- `status`: `replied` or `escalated`
- `request_type`: `product_issue`, `feature_request`, `bug`, or `invalid`

## Design Decisions

### Why OpenRouter (gpt-oss-120b:free)?

**Advantages:**
- Fast inference on short support prompts
- Free tier allows unlimited testing before submission
- OpenAI SDK is familiar and widely supported
- Reliable for extracting structured JSON responses

**Rationale:** The agent receives BM25-retrieved chunks (context capped at 5 chunks), 
so reasoning time is brief. GPT-oss-120b is sufficient for "read context, extract answer" 
tasks and avoids hallucination because responses must extract/summarize chunks only.

### Fallback to Claude?

If OpenRouter is unavailable, the agent tries Claude (via Anthropic SDK). Claude is 
slightly more robust at following strict JSON formatting rules. Both providers use the 
same grounding strategy: pass only retrieved corpus chunks in the prompt.

### Why BM25 rather than a vector DB?

The corpus is small enough to keep in memory, and many support questions
contain exact terms such as product names, settings, certificate, dispute,
Bedrock, LTI, subscription, or minimum spend. BM25 is deterministic,
dependency-light, and easy to explain in the judge interview.

### Why safety rules first?

Some tickets should not be answered even if a nearby document exists. Refunds,
score changes, account restoration, identity theft, prompt injection, and
destructive system requests need conservative routing.

### Why an extractive fallback?

The submission should still run if both API keys are missing. The fallback
quotes/summarizes retrieved local documentation instead of using model memory.

## Testing

Compile all modules:

```powershell
Get-ChildItem code -Filter *.py | ForEach-Object { python -m py_compile $_.FullName }
```

Run the sample set:

```bash
python code/main.py --file support_tickets/sample_support_tickets.csv --output support_tickets/sample_output_review.csv --quiet
```

Run the target set:

```bash
python code/main.py --file support_tickets/support_tickets.csv --output support_tickets/output.csv --quiet
```

## Known Limitations

- The agent cannot modify external systems or issue refunds.
- The agent cannot access private user/account data.
- Very vague tickets are intentionally escalated.
- The extractive fallback is safer than a hallucinated answer, but less polished
  than an LLM-generated response.

## Judge Interview Talking Points

- I chose local RAG because the task is grounded support triage over a fixed
  corpus, not model training.
- I used deterministic safety rules before retrieval to prevent unsafe answers.
- I scoped retrieval by company to reduce cross-domain false positives.
- I added a no-key fallback so the CLI remains reproducible.
- I tuned the architecture against the sample CSV and documented target-ticket
  expectations in `KNOWLEDGE_BASE.md`.
