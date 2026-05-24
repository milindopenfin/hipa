# 01 — Legacy HIPA Observations

> Observations of the existing `hipa/` codebase as of writing (2026-05).
> The point is not to dunk on v1 — it works and validated the product hypothesis.
> The point is to record, concretely, **what to keep, what to discard, and why**, so
> v2 doesn't accidentally re-invent the same failure modes.

Cross-refs: [02 — Architecture overview](02-architecture-overview.md),
[04 — Multi-agent orchestration](04-multi-agent-orchestration.md),
[09 — Citation & grounding](09-citation-and-grounding.md).

---

## What v1 is

A Flask monolith (`hipa/app_.py`, ~5,900 lines) that:

1. Accepts a policy PDF/DOCX/XLSX/CSV via `POST /api/upload`.
2. Extracts text with `pdfplumber`, chunks it, builds an in-memory BM25 index.
3. Detects insurer via `router.py` (regex + LLM fallback).
4. Runs a multi-phase pipeline:
   - **Phase 1**: 6 async domain workers fan out via `asyncio.gather()` —
     coverage, financial, waiting-periods, benefits, premiums, plus a
     structural extractor that produces insurer-specific JSON.
   - **Verifier** (`agents/verifier.py`) deterministically checks each field's
     `source_quote` against the raw text and assigns a confidence score.
   - **Phase 2**: a single synthesis call produces the 12-section / 113-question
     narrative.
5. Persists metadata + raw text to MongoDB, raw file to S3, RAG chunks to a
   disk-backed pickle (`hipa_file_store.pkl`).
6. Drives a Q&A flow (`agents/qa_agent.py`) backed by per-session JSON files
   in `sessions/`.
7. Captures section-level thumbs-up/down feedback to MongoDB.
8. Serves a single 13k-line HTML SPA (`frontend.html` ≈ dev,
   `index.html` ≈ prod — same code, different `API_URL`).

---

## What v1 actually does well — keep these ideas

1. **`extraction_workers.py` source-quote contract.** Every extracted field is
   `{value, source_quote}`. The verifier then proves the quote exists in the
   raw document. This is the right primitive — v2 promotes it from
   "structural-only" to **every fact, including narrative**.
   → [09 — Citation & grounding](09-citation-and-grounding.md)

2. **Async fan-out for extraction.** `asyncio.gather()` over six workers
   cuts wall time to ≈ slowest worker, not sum. v2 keeps the pattern but
   widens it — there's no reason to cap parallelism at six.
   → [06 — Extraction pipeline](06-extraction-pipeline.md)

3. **Per-role model overrides** (`MODEL_ROUTER`, `MODEL_EXTRACTION`, …) in
   `agents/client.py`. The shape is right — keep it, generalize it.
   → [11 — LLM provider abstraction](11-llm-provider-abstraction.md)

4. **Hard JSON schema with descriptions** (`agents/policy_extraction_schema.json`).
   This is the same file now sitting at
   [`docs/health_insurance_extraction.json`](health_insurance_extraction.json).
   v2 treats it as the canonical contract for the *health* domain module.
   → [07 — Extraction schema spec](07-extraction-schema-spec.md)

5. **Section-level feedback in the UI** (`frontend.html` line ~11579–11717).
   Thumbs up/down + free-text per section, with a `review_snapshot`. Right
   primitive — v2 routes the output into a **skill or prompt-review queue**
   rather than just logging it.
   → [10 — Skills & HITL feedback](10-skills-and-hitl-feedback.md)

6. **Insurer alias detection** in `router.py`. Cheap, deterministic, robust
   for the 16 named insurers. v2 keeps the regex tier and falls back to LLM
   only when regex is ambiguous.

---

## What v1 gets wrong — fix in v2

### 1. Text extraction is silent on scans
`pdfplumber` returns empty text for scanned PDFs without erroring. Result:
the LLM analyzes an empty document and confidently fabricates fields.
**v2 must detect zero-text-density pages and route them to OCR.**
→ [05 — Document ingestion](05-document-ingestion.md)

### 2. No document structure
The chunker treats the PDF as a flat text blob. Tables, headings, table-of-
contents, schedules and annexures are all flattened. The LLM has no way to
ask "show me only Section 4.2 — Permanent Exclusions."
**v2 produces a hierarchical markdown tree** (heading → sub-heading → tables
→ paragraphs) so sub-agents can scope retrieval to the relevant subtree.
→ [05 — Document ingestion](05-document-ingestion.md), [08 — RAG store](08-rag-document-store.md)

### 3. Citations exist for structured fields only
`universal_agent.py` produces a long narrative with no source quotes. Half
the output is grounded; half isn't. Users have no way to verify the narrative.
**v2: every fact, narrative or structured, must carry a source span.**
→ [09 — Citation & grounding](09-citation-and-grounding.md)

### 4. The "universal agent" is a 4k-token monolith
`universal_agent.py` builds a ~370-line prompt covering all 12 sections,
113 questions. That's expensive (token count) and inaccurate (the model
loses focus). The "split async" variant (A/B/C, 3 calls) is a workaround,
not a fix.
**v2 uses one sub-agent per schema section**, each with a tight,
section-scoped prompt and section-scoped RAG.
→ [04 — Multi-agent orchestration](04-multi-agent-orchestration.md)

### 5. Insurer logic is bolted in
Insurer aliases live in `universal_agent.py`. Insurer table hints live
inline in the same file. Insurer-specific schemas live in
`structural_extractor.py` (954 lines). Insurer post-processors live in
`provider_normalizers.py`. Adding "MY INSURER" touches four files.
**v2 puts each insurer's quirks in a single declarative *skill* file**
loaded by the domain module.
→ [10 — Skills & HITL feedback](10-skills-and-hitl-feedback.md), [03 — Modular domain](03-modular-domain-system.md)

### 6. Cache is per-process and per-query-string
`file_info['_analysis_cache'][query]` is in-memory only and keyed on exact
query string. Restart wipes it; a typo re-bills the user.
**v2 caches by `(content_hash, schema_version, prompt_version)`** in Mongo,
and treats Q&A as a separate cheap layer over the cached extraction.
→ [15 — Persistence & re-run guard](15-data-persistence-and-rerun.md)

### 7. RAG is BM25 in-memory, lost on restart
`PolicyChunker` + `RAGRetriever` build a BM25 index per file, kept in a
disk pickle. No cross-document retrieval; no semantic search; restart
re-builds everything.
**v2 uses ChromaDB**, indexes every document ever seen, and supports
both per-document scopes (for analysis) and cross-document scopes (for
"find similar Star Health Family Health Optima 2022 wordings").
→ [08 — RAG & document store](08-rag-document-store.md)

### 8. Verifier flags failures but never recovers them
`verifier.py` emits `needs_review=true` for low-confidence fields. Nothing
re-tries them. They ship to the UI as-is.
**v2 retries failed fields** with a different sub-agent / different retrieval
slice before flagging for human review.
→ [04 — Multi-agent orchestration](04-multi-agent-orchestration.md)

### 9. Prompts are inline Python string literals
Every prompt lives inside an agent file. Changing one means editing code;
A/B testing requires a new constant. No versioning, no rollback.
**v2 stores prompts as versioned files** (per domain, per sub-agent), with
the prompt hash logged into every analysis for reproducibility.
→ [10 — Skills & HITL feedback](10-skills-and-hitl-feedback.md), [11 — LLM abstraction](11-llm-provider-abstraction.md)

### 10. Auth is a UUID in plaintext
`app_.py` issues UUID tokens kept in an in-memory `_TOKEN_CACHE` and a
`hipa_sessions` Mongo collection. No expiry, no refresh, no scopes; tokens
are accepted from header **or query string**.
**v2 uses Better Auth** with session JWTs, scopes per route, and CSRF
protection.
→ [16 — API & auth](16-api-and-auth.md)

### 11. AWS keys are hardcoded in `db.py`
Lines 198–202. Self-explanatory. v2 takes secrets from env / secret
manager only.

### 12. Two near-identical 13k-line HTML files
`frontend.html` and `index.html` differ in one line (the API base URL).
This is a config problem, not a duplication problem.
**v2 is a real SPA** (React) with a build-time environment switch.
→ [17 — Tech stack](17-tech-stack.md)

### 13. Q&A questions are commented out
`qa_agent.py` ships with 5 of ~30 questions enabled. The rest were
disabled, presumably because validation broke. The flow is brittle:
adding a question means editing Python.
**v2 makes the question bank data-driven** (JSON per domain) and lets the
Q&A agent dynamically inject follow-ups based on the analyzed policy.
→ [13 — Q&A & explanation](13-qa-and-explanation.md)

### 14. No report generation
There's a "download" button (`frontend.html` ~11568) but it just opens a
presigned S3 URL to the *uploaded* PDF, not a generated report. Nothing
synthesizes a customer-facing PDF report.
**v2 generates HTML/CSS reports** server-side and prints them to PDF with
a headless browser.
→ [14 — Report generation](14-report-generation.md)

### 15. Feedback collected, never used
Thumbs up/down with comments go to a Mongo collection (`/api/feedback`)
and stop. No dashboard, no review queue, no link back to prompts or
skills.
**v2 routes feedback into a review queue** with a triage UI: every
thumbs-down is either (a) a prompt-template issue → reviewer edits the
versioned prompt; or (b) a policy-specific issue → reviewer authors a
skill file.
→ [10 — Skills & HITL feedback](10-skills-and-hitl-feedback.md)

### 16. No system clock, no web search
`universal_agent.py` cannot answer "is this policy currently in force?"
or "what is the latest IRDAI claim-settlement-ratio?" because it has no
tools. It infers everything from the document and its training data.
**v2 gives agents tools** (system time, web search, master-library lookup,
calculator) with per-role permission scopes.
→ [12 — Agent tools](12-agent-tools.md)

---

## Numbers worth remembering

| Metric | v1 |
|---|---|
| App size | ~5,900 lines in one Flask file |
| Universal-agent prompt | ~370 lines, ~4k tokens |
| Max parallel LLM calls | 6 (Phase 1 workers) |
| LLM calls per analysis | ~8 (1 router + 6 workers + 1 synthesis) |
| Insurer schemas | 9 explicit + 1 generic |
| Q&A questions live | 5 of ~30 |
| Cache scope | per-process, per-exact-query-string |
| Citation coverage | structured fields only; narrative is freeform |
| OCR coverage | none |
| Persistent vector store | none (BM25 in pickle) |

---

## What v2 inherits vs. what v2 replaces

| Concern | v1 | v2 |
|---|---|---|
| Source-quote contract | structural fields | **all facts** |
| Schema-driven extraction | yes (1 monolith) | yes (per sub-agent, schema-scoped) |
| Parallel extraction | 6 workers | unbounded fan-out, retried on failure |
| RAG | BM25, in-memory | ChromaDB, persisted, cross-document |
| OCR | none | tesseract / paddleocr fallback |
| Document tree | flat | markdown hierarchy with addressable subtrees |
| LLM provider | OpenRouter only | Anthropic / OpenAI / OpenRouter / Ollama |
| Prompts | inline | versioned files |
| Auth | UUID tokens | Better Auth |
| Cache key | exact query string | content hash + schema + prompt version |
| Citation UI | none | side-by-side highlight |
| Reports | upload presigned URL | HTML→PDF generation |
| Feedback | logged | routed to skills / prompt-review queue |
| Domain coupling | health hardcoded everywhere | health is one domain module of many |
