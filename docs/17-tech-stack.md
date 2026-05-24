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
| Backend language | **Go** | Stated preference; single static binary; great concurrency primitives for the multi-agent fan-out; strong stdlib for HTTP/queue/crypto |
| API framework | `chi` or `echo` (TBD) | Small, idiomatic Go HTTP routers. `chi` preferred for stdlib affinity |
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

## Why Go specifically

The pipeline is dominated by:

- HTTP fan-out to LLM providers (concurrent, latency-sensitive)
- JSON shuffling (schema, citations, jobs)
- File handling (uploads, S3, PDF parsing)
- Long-running workers with backpressure

Go is good at all four. Compared to Node:

- Strong types matter when you're moving schemas around as data.
- Single static binary makes deploy/rollback trivial.
- Excellent stdlib for everything we need.

Compared to Python:

- Better concurrency story for the orchestrator and queue workers.
- We still keep Python around as a *sidecar* for things Go's ecosystem
  lacks (specifically the document-parsing front end if `markitdown`
  remains the cleanest option).

---

## Service decomposition

```
┌──────────────────────────────────────────────────────────┐
│ api-gateway        (Go)   stateless · horizontally scaled│
│ orchestrator       (Go)   queue consumer · workers       │
│ llm-gateway        (Go)   provider routing · rate limits │
│ ingest-svc         (Go + Python sidecar for parsing/OCR) │
│ render-svc         (Chromium + small Go wrapper)         │
│ chroma             (off-the-shelf)                       │
│ mongo              (off-the-shelf)                       │
│ s3                 (off-the-shelf)                       │
└──────────────────────────────────────────────────────────┘
```

The api-gateway and orchestrator can be the **same binary** in MVP,
configured by env to run one or both roles. We split only when scaling
dictates.

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

1. **`markitdown` vs `marker` vs a Go-native PDF→md** — benchmark on
   ≥50 Indian insurance PDFs and pick. Criteria: table fidelity,
   heading extraction accuracy, speed, dependencies.
2. **ChromeDP vs Playwright sidecar** for PDF rendering. ChromeDP is
   simpler; Playwright is more battle-tested. Benchmark.
3. **In-process workers vs sidecar workers.** Start in-process for
   simplicity; split if cold-start or GC tuning becomes a pain.
4. **OpenAPI vs typespec for the client SDK generation.** Either
   works; team familiarity wins.
5. **Embedding model.** OpenAI `text-embedding-3-large` vs `bge-large`
   (self-hosted) vs `nomic-embed-text` (Ollama, free). Benchmark on
   retrieval precision over a held-out doc set.

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
