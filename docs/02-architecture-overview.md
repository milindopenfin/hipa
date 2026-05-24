# 02 — Architecture Overview

> The 10,000-ft view of HIPA v2. Read this first; each layer is then detailed
> in its own doc.

Cross-refs: [03 — Modular domain system](03-modular-domain-system.md),
[04 — Multi-agent orchestration](04-multi-agent-orchestration.md),
[17 — Tech stack](17-tech-stack.md).

---

## Layered model

```
 ┌────────────────────────────────────────────────────────────────────┐
 │  L1 — Edge                                                         │
 │  React SPA (TypeScript) · presigned uploads · live SSE for jobs    │
 └────────────────────────────────────────────────────────────────────┘
                              │
 ┌────────────────────────────────────────────────────────────────────┐
 │  L2 — API gateway (Go)                                             │
 │  Better Auth · RBAC · rate limit · request routing · idempotency   │
 └────────────────────────────────────────────────────────────────────┘
                              │
 ┌────────────────────────────────────────────────────────────────────┐
 │  L3 — Orchestrator                                                 │
 │  Domain dispatch · job lifecycle · cache lookup · retry policy     │
 └────────────────────────────────────────────────────────────────────┘
                              │
 ┌─────────────┬────────────────┬─────────────────┬─────────────────┐
 │  L4 — Domain modules (plug-and-play)                              │
 │  health/    │  fire/  (future)│  motor/  (future)│  life/ (future) │
 │  schema     │  schema         │  schema          │  schema         │
 │  sub-agents │  sub-agents     │  sub-agents      │  sub-agents     │
 │  skills     │  skills         │  skills          │  skills         │
 │  report tpl │  report tpl     │  report tpl      │  report tpl     │
 └─────────────┴────────────────┴─────────────────┴─────────────────┘
                              │
 ┌────────────────────────────────────────────────────────────────────┐
 │  L5 — Shared platform services                                     │
 │  Ingest (PDF → md tree, OCR, unlock)                               │
 │  RAG (ChromaDB) + master library                                   │
 │  LLM gateway (multi-provider, per-role models)                     │
 │  Tools (web search, time, calculator, doc lookup)                  │
 │  Citation index (source spans)                                     │
 │  Report renderer (HTML → PDF via headless Chromium)                │
 │  Feedback router (→ skills, → prompt-review queue)                 │
 └────────────────────────────────────────────────────────────────────┘
                              │
 ┌────────────────────────────────────────────────────────────────────┐
 │  L6 — Persistence                                                  │
 │  MongoDB (analyses, jobs, sessions, skills, prompts, feedback)     │
 │  S3 (raw docs, generated reports, intermediate artifacts)          │
 │  ChromaDB (vector index)                                           │
 └────────────────────────────────────────────────────────────────────┘
```

Everything **above** L4 is domain-agnostic — it doesn't know what a "sum
insured" or a "premium" is. Everything **inside** L4 is domain-specific. Adding
a new line of insurance means dropping a new directory into L4 and registering
it with the orchestrator. See [03](03-modular-domain-system.md).

---

## End-to-end flow

```
upload ──► ingest ──► dedupe ──► (cache hit?) ──yes──► return stored analysis
                                       │ no
                                       ▼
                          orchestrator picks domain
                                       │
                                       ▼
                     domain spawns N parallel sub-agents,
                     each with section-scoped RAG + skills
                                       │
                                       ▼
                 verifier rejects any field missing source span
                                       │
                                       ▼
                 retry layer re-runs failures with broader RAG
                                       │
                                       ▼
                 synthesizer writes layman summary (also cited)
                                       │
                                       ▼
                 persist: analysis JSON · source spans · used prompts
                                       │
                ┌──────────────────────┴──────────────────────┐
                ▼                                             ▼
        report renderer                                 Q&A agent
        HTML → PDF                                      (consumes cache)
                                                              │
                                                              ▼
                                                       feedback widget
                                                       (per section)
                                                              │
                                                              ▼
                                                  feedback router
                                                  → prompt-review queue
                                                  → new/updated skill
```

---

## Components in one paragraph each

**Edge SPA.** TypeScript + React. Uploads via presigned URLs straight to
S3 (the API never proxies bytes). Streams job progress via SSE. Renders
the side-by-side citation view (analysis on the left, PDF on the right
with highlights). Has its own feedback widget per analysis section.

**API gateway.** Go. Single binary. Owns auth (Better Auth), RBAC, request
validation, idempotency keys, rate limiting. Stateless — horizontal
scaling is just more pods.

**Orchestrator.** Go process that consumes the job queue. On a new analysis
it: (1) confirms cache miss for `(content_hash, domain, schema_version,
prompt_version)`, (2) loads the domain module, (3) spawns sub-agents, (4)
runs verifier + retry, (5) persists. See
[04 — Multi-agent orchestration](04-multi-agent-orchestration.md).

**Domain module.** A directory under `domains/<name>/` holding schema,
sub-agent definitions, prompt files, skills, report template, optional
domain-specific tools. The orchestrator treats every domain identically —
no `if domain == "health"` branches anywhere in core. See
[03 — Modular domain system](03-modular-domain-system.md).

**Ingest service.** Takes a raw upload, detects format, unlocks if
password-protected, extracts text. Routes scan pages to OCR. Builds a
hierarchical markdown tree with stable node IDs and char offsets.
See [05 — Document ingestion](05-document-ingestion.md).

**RAG & master library.** ChromaDB instance. Two scopes per query:
*per-document* (analysis-time retrieval over only this policy) and
*cross-document* (find similar wordings from the master library —
every policy ever analyzed, keyed by UIN, insurer, year). See
[08 — RAG & document store](08-rag-document-store.md).

**LLM gateway.** A thin Go service that abstracts providers (Anthropic,
OpenAI, OpenRouter, Ollama). Per-role model config: cheap for routers,
strong for synthesis. Logs every prompt + response with its hash for
auditability. See [11 — LLM provider abstraction](11-llm-provider-abstraction.md).

**Tools.** A registry of side-effectful capabilities sub-agents can
invoke: `system_time`, `web_search`, `calc`, `master_library_lookup`,
`pincode_lookup`. Each sub-agent declares which tools it's allowed to
use. See [12 — Agent tools](12-agent-tools.md).

**Citation index.** A Mongo collection mapping `(analysis_id, field_path)`
→ list of source spans. The UI hits this to render highlights. See
[09 — Citation & grounding](09-citation-and-grounding.md).

**Report renderer.** Domain module ships an HTML/CSS template (Tailwind +
print stylesheets). Renderer interpolates the analysis JSON, runs headless
Chromium, returns a PDF stored in S3. See
[14 — Report generation](14-report-generation.md).

**Feedback router.** Receives per-section thumbs / comments. If the
reviewer marks it as a prompt issue → routes to a prompt-review queue
where a designer can edit and re-version the prompt. If marked as a
policy-specific issue → opens a skill draft prefilled with the failing
extraction, ready for approval and attachment. See
[10 — Skills & HITL feedback](10-skills-and-hitl-feedback.md).

---

## Job lifecycle (states)

```
queued ──► running ──► verifying ──► retrying* ──► succeeded
                  │                            ╲──► needs_review
                  └──► failed (terminal)
```

`*retrying` may re-enter `running` for the specific failed sub-agent.

States persist on the job document in Mongo so an SPA reconnecting after a
crash can pick up exactly where it left off.

See [15 — Persistence & re-run guard](15-data-persistence-and-rerun.md) for the
job document shape.

---

## What's in scope for the MVP

- Health insurance domain module (one schema, ~12 sections).
- Ingest with OCR fallback + password unlock.
- ChromaDB-backed RAG (per-document and master library).
- Multi-agent extraction with citation enforcement.
- Q&A over completed analysis.
- HTML→PDF report generation.
- Feedback widget routed into prompt-review queue + skill drafts.
- Better Auth + RBAC.

## What's out of scope for the MVP

- Additional domains (fire / motor / life) — the platform supports them but
  the modules aren't built yet.
- Live insurer-API quote integration (the system produces a quote-ready
  payload; pushing to insurer APIs is a later phase).
- Mobile native apps.

See [18 — Roadmap](18-roadmap.md) for the phased rollout.
