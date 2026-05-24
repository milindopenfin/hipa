# 08 — RAG Pipeline (Ingest → Retrieve → Ground)

> The full RAG stack: a self-contained **ingestion sub-module** that
> normalizes, deduplicates, extracts metadata and versions every document;
> a **hybrid retrieval pipeline** (BM25 ⊕ dense, ANN candidate fetch + deep
> reranking) with per-source confidence scoring; and a **constrained
> generation** contract that forbids the model from inventing anything the
> retrieval didn't return — with a hard insufficient-evidence fallback when
> retrieval is too weak to answer.

Cross-refs: [04 — Multi-agent orchestration](04-multi-agent-orchestration.md),
[05 — Document ingestion](05-document-ingestion.md),
[06 — Extraction pipeline](06-extraction-pipeline.md),
[09 — Citation & grounding](09-citation-and-grounding.md),
[11 — LLM provider abstraction](11-llm-provider-abstraction.md),
[15 — Persistence & re-run guard](15-data-persistence-and-rerun.md).

---

## Pipeline shape

```
                   ╭───────────────────  INGESTION SUB-MODULE  ──────────────╮
                   │                                                          │
upload  ──►  format/decrypt  ──►  text + tree  ──►  NORMALIZE  ──►  DEDUPE   │
                                       │                                      │
                                       ▼                                      │
                              METADATA EXTRACT  ──►  VERSION  ──►  INDEX     │
                              (insurer / UIN /         (immutable    (BM25 + │
                              FY / doc_type)            doc record;   dense  │
                                                        supersedes   chunks  │
                                                        chain)       in      │
                                                                     Chroma) │
                   ╰──────────────────────────────────────────────────────────╯
                                       │
                                       ▼
                   ╭─────────────────  RETRIEVAL PIPELINE  ──────────────────╮
                   │                                                          │
                   │  query(s)  ──►  HYBRID FETCH         (ANN top-K1)       │
                   │                 (BM25 ∪ dense, RRF merged)              │
                   │                              │                          │
                   │                              ▼                          │
                   │                  CROSS-ENCODER RERANK (top-K2)         │
                   │                              │                          │
                   │                              ▼                          │
                   │              SECTION-SCOPED EXPANSION                  │
                   │              (walk tree to anchor subtree)             │
                   │                              │                          │
                   │                              ▼                          │
                   │              SOURCE CONFIDENCE SCORING                 │
                   │              (consistency · trust · freshness)         │
                   │                              │                          │
                   ╰──────────────────────────────┼─────────────────────────╯
                                                  ▼
                   ╭───────  CONSTRAINED GENERATION  ─────────────────────╮
                   │  context-only system prompt                          │
                   │  JSON schema enforcement                             │
                   │  citation token contract                             │
                   ╰─────────────────────┬────────────────────────────────╯
                                         ▼
                              CITATION BACKEND
                              (verify · resolve spans · persist)
                                         │
                                         ▼
                              CONFIDENCE THRESHOLD CHECK
                              hit threshold? ──yes──► ship grounded result
                                       │
                                       no
                                       ▼
                              HALLUCINATION FALLBACK
                              broaden retrieval · stronger rerank · retry
                                       │
                                  still below?
                                       ▼
                              INSUFFICIENT_EVIDENCE
                              ship null + reason; never fabricate
```

The vertical arrows are the read path during analysis or Q&A; the
horizontal sub-module on top is the write path that runs once per
document.

---

# Part A — The ingestion sub-module

Self-contained: it has its own interface, its own storage, and its own
versioning. Other parts of HIPA see only the artifacts it emits.

> Detailed PDF→tree mechanics (OCR, password unlock, raster generation,
> tree shape) are in [05 — Document ingestion](05-document-ingestion.md).
> This section covers the **RAG-side** of ingestion: normalization,
> deduplication, metadata, versioning, and indexing.

## A.1 Normalization

Goal: identical wording produces identical embeddings and identical
fingerprints, regardless of how the PDF was generated.

Applied in order:

1. **Unicode NFC** — composed form; collapses lookalikes.
2. **Whitespace** — collapse runs of space, normalize line endings,
   strip zero-width characters; preserve table cell boundaries.
3. **Quote / dash normalization** — curly→straight quotes, en/em-dash→`-`.
4. **Number formatting** — preserve original token in the text, but
   emit a parallel `normalized_numbers[]` for the chunk (rupee amounts
   to integers, percentages to floats, dates to ISO). Used by the
   retrieval keyword index, not by display.
5. **Hyphenated line-break repair** — `"pre-\nexisting"` → `"preexisting"`
   (only when the next line starts lower-case).
6. **Case folding** — for fingerprints only. The display text keeps
   original casing.
7. **Boilerplate stripping** — page headers/footers (page number,
   policy number repeated on every page) are detected by per-document
   frequency analysis and stripped from chunk text; the chunk metadata
   records what was stripped so they can be restored for display.

Two parallel views per chunk are emitted:

- `display_text` — what the UI shows and the citation backend matches against.
- `index_text` — normalized, used for embedding and BM25.

The contract: extraction always *matches verbatim quotes against `display_text`*
(the canonical original wording). The normalized view is internal.

## A.2 Deduplication — at three levels

| Level | Key | When it fires | Result |
|---|---|---|---|
| **Whole-document** | sha256 of original bytes | upload time | Re-upload returns existing `content_hash`; analysis cache hit |
| **Near-duplicate doc** | MinHash signature of normalized text | post-ingest | Same document with metadata-only diffs (different DPI scan of the same policy) — linked via `near_dup_of` |
| **Chunk-level** | sha1 of normalized chunk text (the chunk `fingerprint`) | indexing | Identical boilerplate chunks (definitions, standard exclusions, regulatory boilerplate) get a single embedding but many `(content_hash, node_id)` references |

Whole-doc dedup is the most important — it directly enforces the
"no expensive re-runs" guarantee in
[15](15-data-persistence-and-rerun.md).

Chunk-level dedup matters because Indian insurance policies share huge
swaths of boilerplate (IRDAI-mandated language). Storing one embedding
that 500 documents point to is a 500× saving on the dense index and
makes "is this paragraph boilerplate?" a fast cardinality check during
extraction (boilerplate chunks deprioritize during reranking for
differentiated fields like `exclusions.policy_specific_exclusions`).

## A.3 Metadata extraction (the cheap first pass)

A small LLM call (model class `cheap_router`, ~600 tokens out) runs over
the first 3 pages of the normalized text and fills:

```json
{
  "insurer_name": "ICICI Lombard General Insurance Co. Ltd",
  "uin": "ICIHLIP21107V012021",
  "product_name": "Corona Kavach Policy",
  "doc_type": "wordings | schedule | endorsement | brochure",
  "financial_year": "2020-2021",
  "approval_date": "2020-09-07",
  "domain_hint": "health"
}
```

This metadata is the only data that flows out of ingestion *before* the
full extraction pipeline runs. Three downstream consumers depend on it:

- The orchestrator's **skill matching** (see [10](10-skills-and-hitl-feedback.md)) — needs insurer/UIN/product to know which skills apply.
- The **domain classifier** in [04](04-multi-agent-orchestration.md) — `domain_hint` is one signal among several.
- The **master library browse UI** in [16](16-api-and-auth.md) — sortable/filterable rows.

Every field returned by this pass carries its own source span. The
metadata is **rechecked** by the `policy_identity` sub-agent during the
full analysis; on disagreement the full-pass wins and the metadata is
corrected. The pre-pass exists to seed skill matching, not to be authoritative.

## A.4 Versioning

Documents are **immutable** once ingested. A re-ingest with corrected
OCR or a newer extractor produces a **new** record that supersedes the
old:

```
master_documents {
  _id: <content_hash>,
  ingest_version: 3,              # bumped when normalization / OCR / parser changes
  parser_version: "pdfium@v0.3 + tesseract@5.4",
  embedding_model_version: "text-embedding-3-large@v1",
  superseded_by: <content_hash> | null,
  supersedes:    <content_hash> | null,
  near_dup_of:   [<content_hash>, ...],
  ...
}
```

Three concurrent version axes:

- **`ingest_version`** — bumps when the ingestion pipeline itself changes
  in a way that would produce a different tree or different chunks for
  the same input bytes.
- **`embedding_model_version`** — bumps when we switch embedding models
  (recorded per Chroma record, not per document; allows lazy
  re-embedding).
- **`schema_version`** — independent; see [07](07-extraction-schema-spec.md).

A record is **never deleted**. Old records remain queryable for audit
and for analyses that ran against them; new traffic uses the latest
`superseded_by` chain head.

## A.5 What gets written to Chroma

Per-document collection (`docs`) plus a global collection
(`master_library`). Each record:

```json
{
  "id": "<content_hash>::<node_id>",
  "document": "<display_text of chunk>",
  "embedding": [...],
  "metadata": {
    "content_hash": "...",
    "ingest_version": 3,
    "embedding_model_version": "...",
    "node_id": "policy_wording/waiting_periods/specific_diseases",
    "section_path": "Policy Wording > Waiting Periods > Specific Diseases",
    "section_number": "4.3",
    "depth": 3,
    "kind": "leaf | section_summary",
    "page_start": 18,
    "page_end": 19,
    "char_start": 33102,
    "char_end": 34418,
    "fingerprint": "sha1:e4f1...",
    "is_boilerplate": false,
    "ingested_at": "2026-05-25T03:14:00Z",
    "trust_tier": "primary",        // see B.4
    // master-library-only fields:
    "domain": "health",
    "insurer": "ICICI Lombard",
    "uin": "ICIHLIP21107V012021",
    "product_name": "Corona Kavach Policy",
    "financial_year": "2020-2021",
    "approval_date": "2020-09-07",
    "doc_type": "wordings"
  }
}
```

Plus a parallel **BM25 keyword index** over the `document` field
(see B.1). Same chunk, two indices, same metadata.

---

# Part B — The retrieval pipeline

Called by sub-agents during extraction (per-document scope) and by the
Q&A agent (per-document or master-library scope).

## B.1 Hybrid: BM25 + dense, fused

Both indices are queried for every retrieval call:

1. **BM25 candidate set.** Tantivy-backed (or Chroma's built-in lexical
   mode if it benchmarks well). Scores tokens with BM25-Okapi over the
   normalized `index_text`. Strong on exact numbers (`"₹3,00,000"`, `"4.3"`,
   `"36 months"`), regulatory phrases, and rare insurer-specific terms.
2. **Dense candidate set.** Embedding model lookup. Strong on
   paraphrase (`"PED"` ↔ `"pre-existing disease"`, `"OPD"` ↔ `"out-patient"`).
3. **Reciprocal rank fusion.**

   ```
   score_fused(chunk) = Σ over indices  1 / (k + rank_in_index(chunk))
   ```

   with `k = 60`. Robust to score-scale differences between BM25 and
   cosine.

Why both, non-negotiable: BM25 alone misses paraphrases; dense alone
silently mis-ranks numerical or rare-token queries (a top-1 dense
match for "₹3,00,000" is often a different ₹-amount that's
semantically nearby). The fusion catches both failure modes.

## B.2 ANN candidate fetch (the fast tier)

Dense retrieval runs against an **HNSW** (Hierarchical Navigable Small
World) index — Chroma's default backend, tuned for our workload:

- `M = 32` (graph fan-out)
- `ef_construction = 200` (build-time quality)
- `ef_search = 128` (query-time recall vs latency knob; raised to 256
  for retry tier — see C.4)

Latency target: top-100 in < 30 ms on the master library at expected scale.
ANN is **approximate** — we accept ~95% recall vs exact search in
exchange for sub-linear time.

K1 = top-100 candidates from ANN; BM25 returns top-100 in parallel; the
union after RRF is at most 200 (often fewer with overlap).

## B.3 Cross-encoder rerank (the slow, accurate tier)

The 200-ish candidates are fed to a cross-encoder reranker:

- Model class: `reranker` (default `bge-reranker-v2-m3` or `cohere-rerank-3`;
  configurable per the LLM gateway in [11](11-llm-provider-abstraction.md)).
- Input: `(query, candidate.document)` per pair; the model emits a
  relevance score.
- Output: re-sorted candidates; we keep top-K2 (default 8; up to 16 for
  hard queries).

The cross-encoder pass is 50–200 ms for 200 pairs and is the single
most accuracy-lifting step in the retrieval pipeline — it lets us be
sloppy in the ANN tier (high recall, modest precision) and tight in the
final tier (high precision over a small set).

For latency-critical paths (Q&A streaming) the reranker can be skipped
in favor of pure RRF; the analysis path always reranks.

## B.4 Section-scoped expansion

The top-K2 reranked chunks are then **expanded** to coherent subtrees,
not shipped as isolated snippets:

1. For each surviving chunk, walk up the markdown tree to the nearest
   node at `depth ≤ 3` → that's the "anchor subtree."
2. Include all siblings and descendants of the anchor subtree, in
   document order.
3. Always force-include three structural nodes (policy schedule, member
   table, premium block) — they're tiny and universally relevant.
4. Deduplicate by `node_id`.
5. Enforce a token budget (default 4k); if exceeded, drop the
   lowest-ranked anchor's expansion first.

This is the core trick. The model sees contiguous, in-order, contextualized
slices, not seven dis-ordered top-K chunks. Without it, hybrid retrieval +
reranking is still only ~80% as good as it could be.

## B.5 Source confidence scoring

Every retrieved chunk carries a per-chunk `source_confidence` ∈ [0, 1],
composed from three signals:

```
source_confidence(chunk, query_bundle) =
    w_c · retrieval_consistency  +
    w_t · trust_score            +
    w_f · freshness_score

w_c = 0.50, w_t = 0.35, w_f = 0.15   # tuned on the regression corpus
```

### B.5.1 Retrieval consistency

How many of the *N queries* a sub-agent runs surfaced this chunk?

```
retrieval_consistency(chunk) = (# queries that ranked chunk in top-K2) / N
```

A chunk that appears for one keyword query and one paraphrase query
(say, both `"PED"` and `"pre-existing disease"`) is more likely to be
genuinely relevant than a chunk that surfaced for only one. This is
self-checking: if our query set is comprehensive, a real answer will be
hit by multiple queries.

### B.5.2 Trust score (per-source)

Not all sources are equal. The trust tier is set at ingest time on the
document record and inherited by all its chunks:

| Tier | Score | Example |
|---|---|---|
| `primary` | 1.00 | Insurer-issued original wording PDF (native text, IRDAI-registered UIN) |
| `derived` | 0.85 | Issued schedule + endorsement; reflects but doesn't define wording |
| `scanned` | 0.70 | Customer-uploaded scan of an original (OCR quality dependent) |
| `unverified` | 0.55 | Brochure, marketing material, third-party summary |
| `untrusted` | 0.30 | User-typed text, claimed extracts; surfaced for context, never as sole evidence |

A field whose extraction depends only on `unverified` chunks gets a
flag — `low_trust_source` — surfaced in the UI and the report's
compliance variant.

Trust tier can be overridden by skill files when a particular source is
known to be reliable for a specific insurer.

### B.5.3 Freshness score

Documents age. Freshness matters more for some queries (claim-settlement
ratios, regulator notifications) than others (policy wording, which is
locked at issue). Default scoring:

```
freshness_score(chunk, query_intent) =
    intent.freshness_weight = 0   →  1.0  (timeless query; e.g. policy wording)
    intent.freshness_weight > 0   →  exp(-age_years / half_life)
                                      half_life = 2 years default;
                                      overridable per intent
```

`query_intent` is set by the caller: extraction queries default to
`freshness_weight = 0` (policy text doesn't age within a policy's
issue date); Q&A queries about "current" anything (`current CSR`,
`latest hospital network`) set a positive weight so an old chunk loses
ground to a recent one.

### B.5.4 Aggregate

The aggregate `source_confidence` is computed once per (chunk, query_bundle)
and travels with the chunk into generation. Sub-agents and the verifier
both see it.

## B.6 Retrieval call shape

```
retrieve(
  query_bundle = [
    { "text": "waiting period initial 30 days", "intent": {"freshness_weight": 0} },
    { "text": "pre-existing disease waiting period", "intent": {...} },
    ...
  ],
  filters = {                          # Chroma metadata filters
    "content_hash": "..."              # per-doc analysis
    # OR for Q&A:
    # "customer_id": "...", "domain": "health"
  },
  scope = "per_document" | "master_library",
  K1 = 100, K2 = 8,
  expand_subtrees = true,
  enforce_trust_floor = "scanned",     # exclude anything below
  rerank = true,
) -> [
  { chunk, source_confidence, retrieval_consistency, trust_score, freshness_score }
]
```

---

# Part C — Constrained generation

Retrieval delivers grounded context. The generation step must respect it.

## C.1 The system-prompt contract

Every sub-agent prompt and the Q&A prompt include a non-negotiable
preamble:

```
You answer EXCLUSIVELY from the CONTEXT block below.
You MUST NOT use any prior knowledge, training-data facts, or assumptions
about typical policies, typical numbers, typical insurer behavior, or
typical regulations.

If the context does not contain enough information to answer a question
or fill a field, return null + reason "INSUFFICIENT_EVIDENCE". Do NOT
guess, infer beyond what the context literally states, or paraphrase
loosely.

Every value you return MUST be paired with a verbatim quote from the
context. The quote must appear literally in the context.

CONTEXT:
[retrieved chunks with provenance markers]
```

This preamble is shared between sub-agents (extraction) and the Q&A
agent. It is the single most important text in the system and is
versioned like any other prompt (see [10](10-skills-and-hitl-feedback.md)).

## C.2 Structural enforcement

Prompt-level instructions aren't enough. We also enforce structurally:

- **JSON schema mode.** Whenever the provider supports it (Anthropic
  Tool Use, OpenAI `response_format: json_schema`), the response schema
  requires `{ value, source_span: { quote, node_id, char_start, char_end } }`
  for every field. The model literally can't return a value without a
  span; if it tries, the provider rejects.
- **No tool access to general web.** Extraction sub-agents have
  `tools_allowed: []`; they can't search out for "typical" answers. The
  Q&A agent's `web_search` is whitelisted to IRDAI + insurer official
  sites (see [12](12-agent-tools.md)).
- **`temperature: 0.0`** for extraction; minimal sampling for
  synthesis. Deterministic enough to reproduce.

## C.3 Context provenance markers

Each retrieved chunk is rendered in the prompt with explicit provenance:

```
<chunk id="policy_wording/waiting_periods/specific_diseases" page="18-19" trust="primary" conf="0.92">
The Company shall not be liable to make any payment for any expenses
incurred during the first thirty-six (36) months of the policy
period...
</chunk>
```

The model is instructed to copy the `id` into `source_span.node_id` and
the page into `source_span.page`. This means the model never invents an
ID — it copies one out of the provided context. The verifier then
checks that the copied ID was actually present.

## C.4 What the model returns

```json
{
  "field_path": "waiting_periods.specific_disease_waiting_period",
  "value": "36 months for the listed specific diseases.",
  "source_span": {
    "node_id": "policy_wording/waiting_periods/specific_diseases",
    "page": 18,
    "char_start": 33102,
    "char_end": 33245,
    "quote": "shall not be liable to make any payment for any expenses incurred during the first thirty-six (36) months"
  },
  "model_confidence": 0.94
}
```

`model_confidence` is the model's self-reported confidence (sampled via
log-prob heuristics where available). It is **one input** to the
combined confidence below, not the answer.

---

# Part D — Citation backend

The system of record for "where did this fact come from."

> The customer-facing citation UX (side-by-side highlight, layered
> hallucination defenses, derived-field provenance) is documented in
> [09 — Citation & grounding](09-citation-and-grounding.md). This
> section is the backend that makes that UX possible.

## D.1 The citation index

A normalized Mongo collection populated at analysis-persist time:

```
citations {
  _id: ObjectId,
  analysis_id, content_hash,
  field_path: "waiting_periods.specific_disease_waiting_period",
  index_in_field: 0,                          # for multi-span fields
  node_id, page, char_start, char_end, quote,
  source_confidence,                          # aggregate from B.5
  retrieval_consistency, trust_score, freshness_score,
  model_confidence,
  combined_confidence,                        # see D.3
  flags: []                                   # populated by verifier
}
```

Two access patterns:

- `(analysis_id)` → list all citations (for compliance / report appendix).
- `(analysis_id, field_path)` → resolve on click in the side-by-side viewer.

Both are indexed; the second is the hot path.

## D.2 Span resolution & verification

When persisting a sub-agent's output, the citation backend:

1. **Verifies the quote.** The `quote` must literally appear in the
   document's `display_text` between `char_start ± 50` and
   `char_end ± 50` (allowing for whitespace drift). For OCR'd pages the
   threshold is relaxed to fuzzy match ≥ 0.92.
2. **Resolves the bounding box.** From the tree node's `bbox_by_page`,
   produces the rectangle the UI overlays on the page raster.
3. **Computes the combined confidence** (D.3).
4. **Flags failures** — `quote_not_in_document`, `provenance_mismatch`,
   `offset_drift_too_large` — and routes flagged fields back into the
   retry layer (see C.4 → E.3).

## D.3 Combined confidence

The number the verifier uses for thresholding:

```
combined_confidence(field) =
    0.55 · source_confidence       +    # retrieval-side; B.5
    0.30 · verifier_alignment      +    # quote-vs-value alignment, [0,1]
    0.15 · model_confidence              # model self-report
```

- `verifier_alignment` = how well the extracted `value` is reconciled
  with the `quote` (substring presence, normalized-number match,
  unit alignment, keyword proximity inside the ±200-char window).
- `model_confidence` is included but down-weighted because models are
  reliably over-confident.

A field below threshold goes to the retry tier; below the floor after
retries, it gets `INSUFFICIENT_EVIDENCE` (E.2).

## D.4 Derived-field provenance

Some fields are computed from other fields (e.g.
`advisor_assessment.overall_policy_score`, `total_sum_insured = base + bonus`).
These don't cite document spans directly — they cite **other field paths**:

```
{
  "field_path": "advisor_assessment.overall_policy_score",
  "value": 78,
  "derived_from": [
    { "field_path": "sum_insured_and_premium.total_sum_insured", "weight": 0.20 },
    { "field_path": "exclusions.permanent_exclusions", "weight": 0.15 },
    ...
  ]
}
```

The UI walks the chain until it hits leaf-extracted fields with real
citations. This keeps derived insights honest and auditable.

---

# Part E — Confidence threshold & hallucination fallback

The retrieval and grounding pipeline can fail. The contract: it must
fail **loudly and safely**, never silently.

## E.1 The threshold check

Per field type, a `confidence_floor` is configured in the schema. Default
`0.60`; numeric / monetary fields default higher (`0.75`); free-text
synthesis fields default lower (`0.50`). Per-field overrides live with
the schema:

```jsonc
"sum_insured_and_premium": {
  "properties": {
    "total_sum_insured": { ..., "x-confidence-floor": 0.85 }
  }
}
```

Fields below floor are **not shipped** as the analysis result. They
enter the fallback flow.

## E.2 Insufficient-evidence response

When a field fails to meet floor after the retry tier (E.3), it is
emitted as:

```json
{
  "value": null,
  "source_span": null,
  "combined_confidence": 0.41,
  "flags": ["INSUFFICIENT_EVIDENCE"],
  "fallback_reason": "Retrieved chunks did not contain a verifiable value for total_sum_insured.",
  "retrieval_attempts": [
    { "queries": [...], "top_chunks": [{"node_id": "...", "score": 0.62}, ...] },
    { "queries": [...], "top_chunks": [...] }
  ]
}
```

The UI renders this as **"Not stated"** with a question-mark chip; the
report's compliance variant lists it under "gaps." We never fabricate a
plausible value.

The `retrieval_attempts` trace is what the prompt-review queue uses to
diagnose whether the failure is a retrieval issue (improve queries),
a model issue (improve prompt), or a genuine document gap.

## E.3 The fallback / retry tier

Before giving up, the system tries progressively wider retrieval +
stronger ranking:

| Attempt | Retrieval change | Generation change |
|---|---|---|
| 1 (default) | Default queries, K1=100, K2=8, rerank on | Default model |
| 2 (broaden) | + skill-declared fallback queries, K1=200, K2=16, `ef_search=256`, lower `trust_floor` by one tier | Same model, prompt augmented with "Previous attempt flagged: [...]; do not repeat." |
| 3 (escalate) | Cross-section retrieval (siblings of the section, not just within) + full-document scan as last-resort context | Promote `model_class: extraction` → `extraction_strong` |
| Final | — | If still below floor: emit `INSUFFICIENT_EVIDENCE` |

The retry tier reuses the constrained-generation contract throughout
(C.1) — broadening retrieval doesn't loosen the no-invention rule.

## E.4 Aggregate quality flags

A whole analysis whose flagged-field rate exceeds a per-domain
threshold (default 30%) is marked `needs_review` and routed to the
prompt-review queue in [10](10-skills-and-hitl-feedback.md). The
analysis still ships — partial information is better than none — but
the UI shows a banner and the report's customer variant suppresses any
section with > 50% of fields flagged (the compliance variant keeps
them, visibly marked).

---

## Cross-document retrieval (Q&A side)

The same pipeline serves Q&A over the master library. Differences:

| Knob | Per-document (analysis) | Master library (Q&A) |
|---|---|---|
| Filter | `content_hash = X` | `customer_id = X` (own library) or `domain = X, insurer = Y` (cross) |
| K1 / K2 | 100 / 8 | 200 / 16 |
| Reranker | Always on | On for non-streaming; off for streaming chat tokens |
| Source confidence weighting | `w_f = 0.0` (policy wording is timeless) | `w_f` per query intent (Q&A often asks for "current" anything) |
| Trust floor | `scanned` | `unverified` (Q&A may surface marketing material; the agent must mark provenance in its answer) |
| Constrained generation | Same contract; same `INSUFFICIENT_EVIDENCE` fallback | Same contract; same `INSUFFICIENT_EVIDENCE` fallback |

Example Q&A: "is dental covered across any of my plans?" → retrieve
across all `content_hash` in the customer's library, rerank, ground the
answer per-policy with citations; if no policy has a dental clause we
can extract, the answer is *"None of your three policies state dental
coverage"* — not *"dental is typically not covered" — and a citation
of `INSUFFICIENT_EVIDENCE` for the implicit claim where applicable.

---

## Operational notes

- **Embedding cost.** Re-embedding the master library on a model upgrade
  is a multi-hour batch job. Done via a parallel worker pool, with
  records left at old version until upgraded — old queries still work.
- **Chroma horizontal scale.** Single node up to a few million chunks
  is fine. Past that, shard by `(domain, insurer)` — the metadata
  filters we use cluster naturally on those keys.
- **BM25 backend.** If Chroma's built-in lexical mode proves weak, we
  add a Tantivy sidecar indexed off the same chunk store. Either way,
  the hybrid pipeline above is unchanged; only the BM25 implementation swaps.
- **Reranker latency.** Cross-encoder calls are the slowest part of the
  read path (50–200 ms for 200 pairs). For the Q&A streaming path we
  use a cached reranker model and skip when the user types fast; for
  analysis we always rerank — accuracy beats latency in extraction.
- **Cache invalidation.** Anything touching the retrieval pipeline
  output (changing reranker, lowering ef_search, swapping embedding
  model) bumps `prompt_bundle_hash` in [15](15-data-persistence-and-rerun.md)
  via a separate `retrieval_config_hash`. Cached analyses remain valid
  only if the retrieval config they ran under still matches.
