# 15 — Persistence & Re-run Guard

> Analyses are expensive (real money per run). The architecture is designed so
> that **the same input never re-runs**, and so that downstream artifacts
> (reports, Q&A, summaries) are cheap to regenerate from stored intermediate
> state.

Cross-refs: [02 — Architecture overview](02-architecture-overview.md),
[06 — Extraction pipeline](06-extraction-pipeline.md),
[08 — RAG & document store](08-rag-document-store.md),
[09 — Citation & grounding](09-citation-and-grounding.md),
[14 — Report generation](14-report-generation.md).

---

## The cache key

```
analysis_cache_key = sha256(
  content_hash       ||   # the original upload's bytes hash
  domain             ||   # "health"
  schema_version     ||   # integer
  prompt_bundle_hash ||   # hash over all sub-agent prompts
  skill_bundle_hash      # hash over (skill names + versions) that matched this doc
)
```

If a key already exists in Mongo, the orchestrator returns the stored
analysis without calling the LLM.

Cache invalidates **automatically** when:

- the document is re-uploaded with different bytes (different content_hash)
- the schema is bumped (different schema_version)
- a prompt is edited (different prompt_bundle_hash)
- a skill that matches this document is added/updated (different skill_bundle_hash)

Nothing more, nothing less. Editing an unrelated sub-agent's prompt does
not invalidate this analysis — because the bundle hash is over all
prompts but the resolved bundle changes only when the relevant prompts do.

---

## MongoDB collections

```
analyses {
  _id: "<analysis_id>",
  cache_key: "sha256:...",                # unique index
  content_hash: "sha256:...",
  domain: "health",
  schema_version: 1,
  prompt_bundle_hash: "sha256:...",
  skill_bundle_hash: "sha256:...",
  skills_applied: ["niva_bupa__health_companion__2024@v2", ...],

  result: { ...the filled schema... },    # large; consider GridFS for >16MB
  metadata: { sub_agent_run_summary, costs, latencies, ... },

  status: "succeeded" | "needs_review" | "failed",
  flags: [{ field_path, code, message }, ...],

  user_id, customer_id,
  created_at, updated_at,
  superseded_by: <analysis_id> | null     # if recomputed under new versions
}

citations {
  _id, analysis_id, field_path, index_in_field,
  node_id, page, char_start, char_end, quote, content_hash
}   // separate from analyses to keep the parent doc small

master_documents {
  _id: "<content_hash>",                  # one per unique upload
  domain, archive_status, financial_year,
  insurer_name, uin, product_name, approval_date, doc_type,
  s3_key, size_bytes, page_count,
  uploaded_by, uploaded_at,
  superseded_by: <content_hash> | null    # year-over-year linkage
}

customer_libraries {
  _id, customer_id, content_hash, label, added_at
}   // many-to-many: same doc can appear in N customer libraries

jobs {
  _id: "<job_id>",
  analysis_id: <set when running>,
  content_hash, domain,
  state: "queued" | "running" | "verifying" | "retrying" | "succeeded" | "needs_review" | "failed",
  error?, retry_count,
  created_at, started_at, finished_at,
  worker_id, heartbeat_at,                # for sweeper
  ...progress: counts, completed sub-agent list
}

reports {
  _id, analysis_id, content_hash, domain, variant,
  template_version, generated_at,
  s3_key, bytes,
  generated_by, generated_for
}

qa_sessions { ... }              # shape in [13]
feedback { ... }                 # shape in [10]
skill_drafts { ... }
prompt_reviews { ... }

users + auth { ... }             # owned by Better Auth — see [16]
roles + permissions { ... }
```

Each collection has obvious indexes. Worth calling out:

- `analyses.cache_key` — unique. Read-most-of-the-time.
- `analyses.content_hash` — for finding all analyses of one doc across
  schema/prompt/skill versions (audit).
- `master_documents.uin` — for cross-policy lookup.
- `master_documents.{insurer_name, financial_year}` — for the browse UI.
- `citations.{analysis_id, field_path}` — primary access pattern.
- `jobs.heartbeat_at` — sweeper finds stalled workers.

---

## S3 layout

```
s3://<bucket>/
├── docs/<content_hash>/
│   ├── original.<ext>
│   ├── tree.json
│   ├── full_markdown.md
│   ├── ocr_pages.json
│   ├── pages/page-0001.webp ... page-NNNN.webp
│   └── reports/<template_version>/<variant>.pdf
├── prompts/<prompt_bundle_hash>/        # cold-storage prompt bodies (audit)
│   ├── health/policy_identity.md
│   ├── health/waiting_periods.md
│   └── ...
└── branding/<tenant>/logo.svg
```

Everything addressed by hash; never by mutable path. The same document
re-uploaded by a different customer reuses the existing `docs/<hash>/`
artifacts.

---

## Re-run guarantees

| Scenario | Behavior |
|---|---|
| Same file re-uploaded by the same user | `master_documents` and `customer_libraries` upserts; if an analysis exists for `cache_key`, return it. **No LLM call.** |
| Same file uploaded by a different customer | Same hash → reuse. Analysis is per-`(content_hash, ...)`, not per-user — so the LLM doesn't run again. Citation viewer & reports also reuse. |
| Schema bumped | Old analyses retain. New requests run under new schema. We optionally backfill old analyses lazily — see "Migration" in [07](07-extraction-schema-spec.md). |
| Prompt edited | Bundle hash changes. Old analyses retain. New requests recompute. |
| Skill added / edited that matches this doc | New `skill_bundle_hash`. Old analyses retain; new request recomputes. |
| Report template edited | Report cache invalidates for that template_version. **Analysis cache untouched.** Re-render is < $0.001. |
| User runs the same Q&A query | Q&A is not cached at MVP — each turn is a fresh agent call. Cheap relative to extraction. Possible later optimization: cache per `(analysis_id, normalized_question)`. |

---

## Why split jobs, analyses, and citations

Three reasons:

1. **Lifecycle.** A job lives seconds-to-minutes; an analysis lives
   indefinitely; citations are append-only. Mixing them couples access
   patterns awkwardly.
2. **Mongo doc size.** A 16MB cap exists for normal docs (GridFS lifts
   it but adds complexity). Citations alone can be hundreds of items.
   Keeping them in a separate collection keeps `analyses` small and fast.
3. **Read paths.** The UI loads the analysis once; loads citations
   on-demand as the user clicks "view source." Separation reduces the
   parent-doc load.

---

## Job lifecycle (replay-safe)

The orchestrator writes a `jobs` document the moment a job is queued.
Every state transition updates `jobs.state` with an `updated_at`. Workers
update `jobs.heartbeat_at` every ~10s while running. A sweeper job
(separate cron / pod) every 60s:

- finds jobs in `running` with `heartbeat_at < now - 3 min` → re-queues
  them.
- finds jobs in `queued` older than N minutes → bumps priority or alerts.

If a worker dies mid-job, the next worker picks up exactly where the
previous left off **provided sub-agent completions are checkpointed**. We
checkpoint per-sub-agent: as each sub-agent completes, its output is
written to a `job_sub_agent_outputs` sub-document on the job. A resuming
worker reads the checkpointed completions and runs only the missing ones.
Then it re-runs the verifier + retry on the union.

This means even mid-job crashes don't burn the already-spent dollars.

---

## "No expensive re-runs" — concretely

The user's brief: *"These analyses are costly to run so we need to be
doing entries in database to not allow re-runs."* Concretely:

- The orchestrator's first action on a job is a cache lookup. A hit is
  served from Mongo in < 50ms.
- The cache key is content-aware (hash) and version-aware. Renaming a
  file doesn't break the cache; bumping a schema does.
- Even on a cache miss, sub-agent checkpointing means a partial failure
  doesn't restart from zero on retry.
- Reports, Q&A, and side-by-side viewers all read the cached analysis;
  they never trigger re-extraction.

The cost of an analysis is therefore paid **once per (content × schema ×
prompts × skills)** combination, ever. The cost of consuming that analysis
in different ways (report variants, Q&A turns, comparisons) is bounded
by Chromium CPU and one cheap LLM call per turn.

---

## Customer policy storage

The user's brief calls out: *"store all customer existing policies etc in
s3 to cross reference whenever."* This is the master library + customer
library combination:

- Every successful ingest writes the doc to `master_documents` and to S3.
- The customer's library is the join `customer_libraries.customer_id ↔
  master_documents._id`.
- Cross-reference at Q&A time pulls **all** of a customer's policies from
  their library, filters by domain, and feeds them as analysis IDs into
  the Q&A agent (which then field-lookups across them — see [13](13-qa-and-explanation.md)).

So when a customer asks "is dental covered across any of my plans?", the
Q&A agent sees a list of analysis_ids and answers grounded in their
existing covers.

---

## Retention & GDPR-adjacent

- **Originals (S3 docs).** Kept until the customer deletes their account
  or 7 years, whichever sooner. Deletion is hash-aware: a doc shared
  across customer libraries is only purged when *no* library still
  references it.
- **Analyses (Mongo).** Same retention as originals.
- **Prompt bodies (S3 cold storage).** 1 year retention. Long enough for
  audit; not forever.
- **Q&A turns.** Default 90 days; configurable per tenant.
- **Feedback.** Indefinitely (anonymized after 1 year).

Account deletion is a tombstone + background purge. The master library
metadata is anonymized but retained for cross-policy retrieval (a
customer's identity is severed from a UIN even after deletion). The
underlying PDF in S3 is purged.

---

## Disaster recovery

- **Mongo:** daily snapshots; PITR within 7 days.
- **S3:** versioning + cross-region replication on
  `docs/`. Reports under `reports/` are regenerable from analyses, so
  they're not replicated cross-region.
- **Chroma:** rebuildable from `docs/<hash>/tree.json` files. We keep
  a snapshot daily; full rebuild from S3 is a multi-hour cold start.
- **Prompt and skill bodies:** versioned in git (the source of truth);
  S3 cold-storage is for at-runtime audit.

---

## Schema migrations are first-class

When `schema_version` bumps, a Go migration runs that:

1. Generates a derived view of all old analyses under the new schema
   (where fields can be mapped without re-extraction, e.g. rename, copy,
   default).
2. Tags analyses where re-extraction is *needed* with
   `needs_reextraction_for_schema_version: N`.
3. On next user request for that analysis, the orchestrator can either
   return the derived view + a "this was migrated, want to re-extract
   for higher accuracy?" prompt, or auto-re-extract per tenant policy.

This keeps the cache useful through schema evolution without forcing a
massive re-billing event.
