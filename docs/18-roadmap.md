# 18 — Roadmap

> Phased delivery. The architecture in this folder is large; the MVP is small.
> Each phase ends with a system that's useful on its own — no phase relies on
> the next one being shipped to be valuable.

Cross-refs: [02 — Architecture overview](02-architecture-overview.md),
[03 — Modular domain system](03-modular-domain-system.md),
[17 — Tech stack](17-tech-stack.md).

---

## Phase 0 — Foundations (weeks 1–2)

Setup so every later phase can run.

- Repo skeleton: `core/`, `domains/`, `spa/`, `infra/`.
- Local dev environment (`make dev`) with Mongo + Chroma + MinIO + Ollama.
- CI: build + lint + smoke tests; container builds.
- Better Auth wired up; basic user model.
- The 19-file `docs/` folder (this set), kept in repo.

**Exit criteria:** a developer can clone the repo, run `make dev`, sign
up a user via Better Auth, and hit a `/version` endpoint.

---

## Phase 1 — Ingest + library (weeks 3–4)

Get documents in and addressable. No LLM work yet.

- `POST /uploads` with presigned-URL flow.
- Password-protected PDF unlock flow.
- Format detection, native-text extraction, **OCR fallback** with
  Tesseract.
- Markdown tree builder; tree persisted to S3; per-page WebP rasters
  rendered once.
- Per-document Chroma collection populated.
- **Master library** populated with metadata (insurer, UIN, year, etc.)
  — initial metadata extracted via a cheap LLM call (re-used as the
  "first pass" for skill matching in Phase 3).
- Library browse UI: table + filters per the user's brief
  (Archive/Non-Archive, FY, insurer, UIN, product, approval date).

**Exit criteria:** upload a 50-page scanned health-insurance PDF, see
it ingested with OCR, see its node tree in the citation viewer (no
analysis yet), see it appear in the master library and the user's own
customer library.

---

## Phase 2 — Single-agent extraction (week 5)

A bare-minimum pipeline to prove end-to-end plumbing before adding fan-out.

- Health domain module skeleton: `manifest.yaml`, schema (the existing
  [`health_insurance_extraction.json`](health_insurance_extraction.json)).
- **One** sub-agent (`policy_identity`), with prompt file, RAG queries,
  agent.yaml.
- LLM gateway (Phase 2 minimum: Anthropic + OpenAI providers).
- Verifier with deterministic source-span check.
- `POST /analyses` returns a job, SSE progress, persists analysis +
  citations.
- Side-by-side citation viewer in the SPA — limited to the 9 fields of
  `policy_identity`.

**Exit criteria:** upload a policy, hit "analyze," watch the job
progress via SSE, see policy identity fields appear in the viewer with
working "view source" highlights.

---

## Phase 3 — Full multi-agent fan-out (weeks 6–8)

The actual product.

- All 12 health sub-agents authored and tuned.
- Orchestrator fan-out + verifier + retry layer.
- Skill matching pipeline (pre-pass for `policy_identity`, then skill
  attach to sub-agents).
- Initial skill library: ports of v1's per-insurer table hints from
  `universal_agent.py` and `structural_extractor.py` into individual
  skill YAMLs (16 insurers × 1–3 quirks each, roughly).
- Synthesizer for layman summaries per section.
- Full analysis caching keyed on `(content_hash, domain, schema_version,
  prompt_bundle_hash, skill_bundle_hash)`.

**Exit criteria:** all 113 schema fields filled with valid citations on
a corpus of ≥ 20 real policies, with verifier flag rate < 10%.

---

## Phase 4 — Q&A + feedback loop (week 9)

Close the human loop.

- Q&A agent (Flow A: guided question bank; Flow B: free-form chat).
- Question bank for health, ported and expanded from v1's `qa_agent.py`
  (the live 5 questions plus the commented-out ~25, re-enabled with
  validators).
- Per-section + per-field feedback widget in the SPA.
- Skill-draft modal: authors a YAML pre-filled from the failing
  extraction.
- Admin queue UI: review pending skills + prompt edits, run regression
  suite, approve/reject.
- Feedback router writes drafts; approval ships skills to `skills/`.

**Exit criteria:** a thumbs-down on a wrong field produces a skill
draft visible in the admin queue; approving it auto-fixes future
analyses matching the skill's selector.

---

## Phase 5 — Report generation (week 10)

The customer-facing deliverable.

- HTML/CSS report template for the health module (3 variants: customer,
  rm, compliance).
- Headless Chromium rendering pipeline.
- Report API endpoints, persistence, presigned URLs.
- Branding config plumbing (per-tenant logos / colors / contact).

**Exit criteria:** an analysis can produce a clean, cited, branded PDF
report on demand; regeneration after a template edit costs no LLM tokens.

---

## Phase 6 — Hardening (weeks 11–12)

What it takes to put the system in front of real users.

- RBAC fully implemented: roles, permissions, API tokens, audit log.
- Rate limiting, idempotency keys, error handling polish.
- Observability: traces, structured logs, dashboards.
- Backups + DR runbook.
- Load testing.
- Eval harness (port of v1's `scripts/eval_extraction.py`) running in CI
  against a curated regression corpus.

**Exit criteria:** the system can be handed to a small pilot group of
RMs and customers without a developer on standby.

---

## Cumulative MVP (end of Phase 6)

- Health insurance fully supported as a domain module.
- OCR + password-protected docs.
- Multi-agent extraction with citation, retry, verification.
- Side-by-side citation viewer.
- Q&A with grounded answers.
- Per-section feedback → skills / prompt-review queue.
- Branded HTML→PDF reports (3 variants).
- Master library + customer library cross-reference.
- Better Auth + RBAC + API tokens.
- LLM provider abstraction (Anthropic, OpenAI default; OpenRouter,
  Ollama configurable).
- No expensive re-runs; reproducible from cache.

This is enough to replace v1 entirely for the health line and to put
the platform in front of paying users.

---

## Post-MVP — modules

These are the explicit modularity payoff. Each is roughly **3–4 weeks**
for a new domain, dominated by schema design + sub-agent prompt
authoring + report template + an initial skill seed. None require core
changes.

| Module | Notes |
|---|---|
| **Fire / property** | Similar shape; perils-based schema; different report template |
| **Motor** | Multi-vehicle handling; PUC/RC integration as tools |
| **Travel** | Short-tenure; trip details from customer profile |
| **Life** | Different beast — riders heavy; surrender values; long projections |
| **Marine** | Specialty; lowest priority unless a buyer asks |

Adding any of these = `domains/<name>/` directory + the seven
artifacts described in [03 — Modular domain system](03-modular-domain-system.md).

---

## Post-MVP — features

| Feature | Why later |
|---|---|
| Live insurer-API quote integration | Each insurer integration is a separate negotiation; the payload we produce is already quote-ready |
| Year-over-year UIN diffs | Compliance value, but requires curated label data; Phase 7+ |
| Customer memory across sessions | Privacy-cost vs. benefit unclear; revisit with usage data |
| Fine-tuned extraction model | Need labeled extraction pairs; build the dataset from feedback first |
| Dashboard for feedback metrics | Sketched in [10](10-skills-and-hitl-feedback.md); operations need it; not blocking the product |
| GraphQL / batch APIs | Only when API consumers demand it |
| Mobile native apps | PWA covers the gap |
| Multi-tenant per-customer Chroma collections | Useful for very large enterprise tenants; not needed at MVP scale |
| White-label tenant onboarding flow | Manual config is fine for the first N tenants |

---

## What we will *not* build

- An end-to-end chat agent that closes the policy sale (out of legal
  scope; we produce advice + payload, not transactions).
- A general-purpose document-Q&A platform (HIPA is insurance-specific
  by design; the modularity is across insurance lines, not domains).
- A replacement for IRDAI compliance review (we surface evidence; we
  don't replace humans on regulated decisions).
- A "smart" memory that auto-summarizes user history (until the value
  is clear, we keep state explicit and bounded).

---

## Risk register (top items)

| Risk | Mitigation |
|---|---|
| OCR quality on real Indian scans is worse than benchmarks | Phase 1 includes a small annotated corpus; if Tesseract underperforms, PaddleOCR is the next stop |
| LLM provider rate limits during a launch spike | Multiple providers wired in from Phase 2; per-role failover configurable; back-pressure visible to users via SSE |
| Schema over-fitting to current insurers | Schema is versioned; migrations are first-class; we expect at least one bump in the first 6 months |
| Skill explosion (100s of small skills) | Approval gate + regression suite is the load-bearing control; if maintenance gets painful, consolidate skills via prompt edits |
| The 12 sub-agent design is too granular | We can collapse to fewer sub-agents per section without breaking the contract; conversely, we can split a sub-agent (e.g. exclusions → permanent + temporary) without breaking it either |
| Citation viewer UX is harder than expected | We have prior art in v1's PDF download + a clear bbox storage model; biggest unknown is highlight precision on OCR'd pages |

---

## Definition of "done" for each phase

For every phase:

- Code reviewed and merged.
- Tests written and passing in CI.
- Regression suite (Phase 3 onward) passing with no new flags.
- Phase-specific exit criteria above met.
- A short demo recorded.
- Docs updated where the implementation revealed something the plan
  missed (this folder is a living document, not a tombstone).
