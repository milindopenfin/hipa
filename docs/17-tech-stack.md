# 17 — Tech Stack

> The chosen technologies and the reasons. Where there's a genuine choice
> still to make, this doc says so explicitly — those are the open questions
> for the build phase.

Cross-refs: [02 — Architecture overview](02-architecture-overview.md),
[11 — LLM provider abstraction](11-llm-provider-abstraction.md),
[16 — API & auth](16-api-and-auth.md),
[18 — Roadmap](18-roadmap.md).

---

## Per-layer choices

| Layer | Choice | Why |
|---|---|---|
| Control-plane language | **Go** | Single static binary; goroutines fit the multi-agent fan-out exactly; strong stdlib for HTTP/queue/crypto; cheap deploy & rollback |
| AI/document-plane language | **Python** | First-class ecosystem for PDF→markdown (markitdown/marker/Unstructured), OCR (Tesseract/Paddle), cross-encoder rerankers, embeddings, Pydantic/Instructor for structured extraction |
| RPC — typed legs (SPA↔Go, Go↔Go) | **webrpc** (RIDL) | One schema → typed TS client + typed Go server; no protobuf/HTTP/2 baggage; named errors map to our flag taxonomy |
| RPC — Python boundary (Go↔ai-py) | **OpenAPI** | FastAPI emits OpenAPI for free; mature Python codegen (`openapi-python-client`, `datamodel-code-generator` for Pydantic); webrpc-py is not yet first-class |
| Live updates (SPA job progress) | **SSE** (handwritten alongside webrpc) | webrpc is request/response only; SSE handles the one-way streaming case |
| File upload | **Presigned S3 PUT** | API never proxies bytes |
| API framework (Go) | `chi` | Stdlib affinity; pairs cleanly with webrpc-generated handlers |
| API framework (Python) | `FastAPI` | Pydantic-native; emits OpenAPI as the source of truth for the Python boundary |
| Backend job runtime | In-process workers + Mongo-backed work queue | Simplest viable; revisit if hot |
| Frontend language | **TypeScript** | Stated preference; non-negotiable |
| Frontend framework | **React 19** (or Next.js if SSR matters; default to plain React + Vite) | Stated preference: "Some flavour of React"; Vite for fastest dev loop; switch to Next.js only if SSR becomes a need |
| Styling | Tailwind | Designer-friendly, plays well with the PDF report HTML/CSS |
| Forms / SPA state | TanStack Query for server cache, Zustand for local state | TanStack Query handles caching, retries, optimistic updates; Zustand is the smallest possible store |
| Auth | **Better Auth** | Stated preference; mature; Mongo adapter exists |
| Primary DB | **MongoDB** | Stated preference; great fit for the JSON-shaped analyses |
| Vector DB | **ChromaDB** | Stated preference; simple to operate; supports hybrid search |
| Blob storage | **S3** (or S3-compatible like Cloudflare R2 / MinIO) | Stated preference; standard |
| Document parsing | `pdfium` via Go binding for native text; Tesseract 5 for OCR | Best free-tier combo for Indian insurance PDFs |
| Markdown conversion | `markitdown` (Python sidecar) or a Go port; benchmark vs. `marker` | Both produce GFM with tables; we'll pick the cleaner one |
| Headless rendering (PDF) | Chromium via ChromeDP (Go) or a Playwright sidecar | ChromeDP is Go-native; Playwright is more robust for tricky CSS |
| Queue | Mongo-backed queue initially; Redis Streams or NATS if needed | Defer the heavier dependency until volume requires it |
| Observability | OpenTelemetry → Grafana / Tempo / Loki | Standard, swappable |
| CI/CD | GitHub Actions; container builds; tag-based deploys | Familiar |
| Container runtime | Docker → Kubernetes (or Fly.io / Render for early stages) | TBD per ops preference |

---

## Why this language split

The backend cleaves cleanly into two planes with different ecosystem needs:

**Control plane (Go).** Dominated by:

- HTTP fan-out to LLM providers (concurrent, latency-sensitive)
- JSON shuffling (schema, citations, jobs)
- Long-running workers with back-pressure
- Auth, RBAC, rate limiting

Go is best-in-class for all of these. Single static binary, strong stdlib,
goroutines + channels map directly onto the L0/L1 orchestration design in
[04 — Multi-agent orchestration](04-multi-agent-orchestration.md).

**AI/document plane (Python).** Dominated by:

- PDF → hierarchical markdown (markitdown / marker / Unstructured)
- OCR (Tesseract / PaddleOCR) with preprocessing
- Hybrid retrieval + cross-encoder reranking (sentence-transformers,
  FlagEmbedding)
- Structured extraction patterns (Pydantic, Instructor)

These are the parts of the system where Go would be reinventing tooling
that already exists, mature, in Python. Critically, the LLM calls
themselves live in the **Go** LLM gateway — Python's role here is local
document processing and retrieval, not model orchestration.

**TypeScript stays on the SPA only.** It would lose to Go on the control
plane (deploy ops, fan-out, queue workers) and to Python on the AI plane.
The one TS upside — shared types with the backend — is recovered by
codegen from the schema directory rather than a shared runtime.

---

## Service decomposition

```
┌──────────────────────────────────────────────────────────────────┐
│ core-go            (Go)   one binary, role-toggled by env:       │
│                            · api-gateway role                    │
│                            · orchestrator role                   │
│                            · llm-gateway (in-process MVP)        │
│ ai-py              (Python / FastAPI)   one service, two routes: │
│                            · /v1/ingest   PDF → tree, OCR        │
│                            · /v1/retrieve hybrid RAG + rerank    │
│ render-svc         (Chromium + small Go wrapper via ChromeDP)    │
│ chroma             (off-the-shelf)                               │
│ mongo              (off-the-shelf)                               │
│ s3 / MinIO         (off-the-shelf)                               │
└──────────────────────────────────────────────────────────────────┘
```

**Deployment principle:** logical services, deployable as one or many.
Boundaries are network-shaped from day one (so splits are mechanical
later), but everything that can collocate, does, until scaling dictates
otherwise.

| Phase | What splits out | What stays collocated |
|---|---|---|
| MVP | `core-go` (multi-role binary), `ai-py`, `render-svc`, data plane | `llm-gateway` lives inside `core-go`; ingest + rag live inside `ai-py` |
| Scale v1 | `llm-gateway` becomes its own Go service (rate-limit state goes global, Redis-backed) | rest unchanged |
| Scale v2 | Split `rag-svc` from `ingest-svc` inside `ai-py` (different load profiles) | — |
| Scale v3 | Multiple `render-svc` workers behind a queue | — |

The api-gateway and orchestrator are roles inside the same `core-go`
binary — `core-go --role=gateway`, `--role=orchestrator`, or both. We
split only when scaling dictates.

---

## How services communicate

One protocol per channel, chosen for the shape of the traffic:

| Channel | Protocol | Notes |
|---|---|---|
| SPA ↔ `core-go` (RPC) | **webrpc** (RIDL) | Typed TS client + Go server; named errors |
| SPA ↔ `core-go` (job progress) | **SSE** | Handwritten endpoint next to webrpc routes |
| SPA → S3 | **Presigned PUT** | API never proxies bytes |
| `core-go` internal (gateway ↔ orchestrator ↔ llm-gateway) | **In-process** in MVP | Same binary; Mongo job queue is the real boundary |
| `core-go` ↔ `ai-py` | **OpenAPI / HTTP+JSON** | FastAPI emits the spec; Go client via `oapi-codegen`, Python models via `datamodel-code-generator` |
| `core-go` ↔ `render-svc` | **webrpc** | Both Go; result lands in S3, key returned |
| Any service ↔ Mongo / S3 / Chroma | Native client libs | Shared data plane |

### Why webrpc

- Schema-first like gRPC, but the wire format is plain HTTP+JSON —
  inspectable in browser devtools, no HTTP/2 dependency, no protobuf
  toolchain.
- The TS↔Go boundary is exactly its sweet spot: one RIDL produces a typed
  TS client and a typed Go handler interface in lockstep.
- Errors are declared in the schema and surface as typed values in both
  languages — maps cleanly onto our flag taxonomy
  (`PASSWORD_REQUIRED`, `INSUFFICIENT_EVIDENCE`, `LOW_TRUST_SOURCE`, …).

### Why OpenAPI for the Python boundary

webrpc's Python generator is community-quality and trails the Go/TS
targets. The Go↔`ai-py` interface is narrow (~5 endpoints) and slow-
changing — exactly where OpenAPI's mature Python tooling shines.
FastAPI emits the spec automatically; we never write OpenAPI YAML
by hand.

This is the **hybrid** approach. Re-evaluate end-to-end webrpc if the
Python generator reaches parity by the time we need to bump versions.

### Schemas as the load-bearing artifact

```
schemas/
├── data/                  # shared data shapes (JSON Schema)
│   ├── health_insurance_extraction.json
│   ├── tree.schema.json
│   ├── citation.schema.json
│   └── job.schema.json
├── webrpc/                # RPC contracts (RIDL)
│   ├── api-gateway.ridl   # SPA ↔ Go
│   ├── llm-gateway.ridl   # Go ↔ Go (used when split)
│   └── render-svc.ridl    # Go ↔ Go
└── openapi/               # contracts to/from Python
    ├── ingest-svc.yaml    # Go ↔ Python
    └── rag-svc.yaml       # Go ↔ Python
```

Single source of truth for every type that crosses a service boundary:

- Go structs from `oapi-codegen` (OpenAPI) and webrpc-gen (RIDL).
- Pydantic models from `datamodel-code-generator` (OpenAPI) +
  `datamodel-code-generator --input-file-type jsonschema` for the
  shared data shapes.
- TS types from webrpc-gen (RIDL).

`schema_version` per
[07 — Extraction schema spec](07-extraction-schema-spec.md); bumps are
code review.

---

## Frontend

```
spa/
├── app/                   # routes
├── components/
│   ├── citation-viewer/   # side-by-side PDF + analysis (the headline feature)
│   ├── feedback-widget/
│   ├── library/
│   ├── qa-chat/
│   └── ...
├── lib/
│   ├── api/              # generated client from OpenAPI/typespec
│   ├── auth/             # Better Auth client
│   └── ...
└── styles/
```

Key sub-components:

- **Citation viewer.** PDF.js for client-side PDF rendering (or rely on
  the server-rendered WebP rasters — likely WebP for performance). Highlight
  overlays positioned from `bbox_by_page` data.
- **Q&A chat.** SSE-driven; streams tokens. Inline citation chips.
- **Feedback widget.** Per-section + per-field, with the triage chip
  selector.
- **Library browse.** Standard table + filters; the master library view.

The SPA talks only to the API. No third-party services are called
client-side except Better Auth's own (which it owns).

---

## Database modeling notes

- **Mongo schema validation** on critical collections (`analyses`,
  `master_documents`, `feedback`). Loose enough to evolve, strict
  enough to catch shape regressions.
- **GridFS** for analyses larger than 16MB (rare but real for policies
  with huge member lists).
- **Indexes**: see per-collection notes in
  [15 — Persistence](15-data-persistence-and-rerun.md).
- **Migrations**: a versioned migrations directory; one migration per
  schema bump.

---

## Local dev story

```
make dev
```

…starts:

- Mongo (Docker)
- Chroma (Docker)
- MinIO (S3-compatible, Docker)
- The Go services (live-reload via `air`)
- The SPA (Vite)
- Optionally Ollama for offline LLM (no API keys required)

We aim for a one-command bootstrap; the legacy hipa's `start.sh` /
`start.bat` are good prior art for what's needed.

---

## Observability

- **Tracing**: OpenTelemetry spans across api-gateway → orchestrator →
  llm-gateway → provider. One trace per user request.
- **Logs**: structured (JSON), tagged with `request_id`, `analysis_id`,
  `job_id` where applicable.
- **Metrics**: per-route latency histograms; per-role LLM token + cost
  rollups; per-job state durations; verifier flag rates; thumbs-down
  rate per sub-agent.

Operational dashboards target the metrics in [10 — Skills & HITL
feedback](10-skills-and-hitl-feedback.md) ("Feedback metrics dashboard")
plus the cost telemetry in [11](11-llm-provider-abstraction.md).

---

## Security highlights

- All secrets via env or secret manager. No keys in code, ever (v1's
  `db.py` did; v2 must not).
- TLS everywhere.
- Better Auth handles user crypto.
- API tokens stored as sha256, prefix shown for UX.
- File uploads scan for max size, content type sniff, basic malware (a
  ClamAV sidecar; cheap insurance even though policy PDFs aren't a
  high-malware vector).
- Per-tenant isolation: every data access is scoped by `tenant_id` /
  `customer_id` at the middleware level.

---

## Things we explicitly defer

| Deferred | Why now is fine |
|---|---|
| Redis / NATS queue | Mongo-backed queue is enough for MVP |
| Kubernetes | Fly.io / Render / single-VM Docker works for early traffic |
| GraphQL | REST is simpler; cardinality of resource types is small |
| Server-side rendering | The SPA is fine as a SPA; deep-linkable URLs cover SEO needs |
| Native mobile | Web works; PWA later if usage demands |
| Fine-tuned models | Prompt engineering covers MVP; data for fine-tuning will accumulate |
| Multi-region active-active | One region with cross-region S3 replication is enough |

---

## Open questions for the build phase

1. **`markitdown` vs `marker` vs Unstructured** — benchmark on ≥50
   Indian insurance PDFs and pick. Criteria: table fidelity, heading
   extraction accuracy, speed, dependencies. All three are Python and
   live inside `ai-py`.
2. **ChromeDP vs Playwright sidecar** for PDF rendering. ChromeDP is
   simpler and keeps `render-svc` in Go; Playwright is more battle-tested
   on tricky CSS. Lean Go unless benchmarks force the switch.
3. **In-process workers vs sidecar workers.** Start in-process for
   simplicity; split if cold-start or GC tuning becomes a pain.
4. **Embedding model.** OpenAI `text-embedding-3-large` vs `bge-large`
   (self-hosted in `ai-py`) vs `nomic-embed-text` (Ollama, free).
   Benchmark on retrieval precision over a held-out doc set.
5. **Reranker.** `bge-reranker-v2-m3` in-process in `ai-py` vs Cohere
   Rerank vs Voyage. In-process is free and controllable; hosted is one
   less model dependency. Benchmark on the same corpus as #4.

None of these are blockers for the architecture in this docs folder.
Each is a localized decision the build can make without rearranging the
plan.

---

## Versioning of *this* repo

- `domains/<name>/` directories are versioned in git; module versions
  travel as `version:` fields in their manifest.
- Sub-agent prompt versions are git-tracked.
- The `health_insurance_extraction.json` schema is git-tracked; bumping
  `schema_version` is a code review.
- Skills are also git-tracked; the approval workflow in
  [10](10-skills-and-hitl-feedback.md) lands them by PR.

Everything that affects analysis output is reproducible from a commit
SHA + a Mongo snapshot.
