# 06 — Extraction Pipeline

> The hot path of HIPA: given an ingested document tree, fill the domain
> schema with grounded, verified fields in parallel.

Cross-refs: [04 — Multi-agent orchestration](04-multi-agent-orchestration.md),
[05 — Document ingestion](05-document-ingestion.md),
[07 — Extraction schema spec](07-extraction-schema-spec.md),
[08 — RAG & document store](08-rag-document-store.md),
[09 — Citation & grounding](09-citation-and-grounding.md).

---

## Inputs / outputs

**Input**
- `document_tree`: see [05](05-document-ingestion.md)
- `domain`: e.g. `health`
- `schema`: the domain's `schema.json` (see [07](07-extraction-schema-spec.md))
- `applicable_skills`: per-document, already matched (see [10](10-skills-and-hitl-feedback.md))

**Output**
- `analysis`: the schema, filled, with `source_span` on every leaf
- `flags`: list of `{field_path, code, message}` for unrecoverable failures
- `metadata`: prompt-bundle hash, model usage per sub-agent, wall time

---

## Sequence

```
                ┌──────────────────────────────┐
                │  load domain bundle          │
                │  (sub-agents, prompts, RAG)  │
                └──────────────┬───────────────┘
                               │
                ┌──────────────▼───────────────┐
                │  schedule N sub-agents       │
                │  (one per schema section)    │
                └──────────────┬───────────────┘
                               │ fan-out, unbounded
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                      ▼
   sub-agent A            sub-agent B           sub-agent N
   ─ retrieve subtree     ─ retrieve subtree    ─ ...
   ─ build prompt         ─ build prompt
   ─ call LLM             ─ call LLM
   ─ parse JSON           ─ parse JSON
        │                      │                      │
        └──────────────────────┼──────────────────────┘
                               ▼
                ┌──────────────────────────────┐
                │  verifier (deterministic)    │
                │  schema · spans · alignment  │
                └──────────────┬───────────────┘
                               │
                ┌──────────────▼───────────────┐
                │  retry layer                 │
                │  re-run failed sub-agents    │
                │  with broader RAG + stronger │
                │  model                       │
                └──────────────┬───────────────┘
                               │
                ┌──────────────▼───────────────┐
                │  synthesis (optional)        │
                │  produce layman summary      │
                │  cited via verified fields   │
                └──────────────┬───────────────┘
                               │
                               ▼
                       analysis + flags
```

---

## How a sub-agent fills its section

For every required field in the schema fragment the sub-agent owns, the
agent must produce:

```json
{
  "value": "30 days",
  "source_span": {
    "node_id": "policy_wording/waiting_periods/initial",
    "page": 18,
    "char_start": 33212,
    "char_end": 33305,
    "quote": "no claim shall be admissible for any illness contracted within the first 30 days"
  }
}
```

The prompt insists on JSON-only output. The schema fragment is sent to
the model. The RAG context is the relevant subtree(s) of the document,
selected by the sub-agent's `rag_queries.yaml` plus, when applicable,
skill-declared anchors. Tables are serialized to GFM inline; the
sub-agent can ask (via tool call) for "show me the whole table at
`node.tables[2]`" if the inline summary clips it.

---

## RAG selection — section-scoped, not document-scoped

Each sub-agent retrieves chunks **scoped to a list of candidate
subtrees**. The retrieval algorithm:

1. From `rag_queries.yaml`, run hybrid search (BM25 + dense embedding)
   against the per-document Chroma index.
2. From the matched chunk's `node_id`, walk **up** to the nearest tree
   node at depth ≤ 3 — this becomes the "anchor subtree."
3. Include **all** sibling and child chunks of the anchor subtree, in
   document order, until a token budget cap (e.g. 4k tokens).
4. Always force-include three structural nodes regardless of search:
   policy schedule, member table, premium block — these are tiny and
   universally relevant.

This is the core trick v1 misses. By following the document tree v2
hands the model a contiguous, in-order, fully-contextualized slice
instead of seven dis-ordered top-K chunks.

Detail: [08 — RAG & document store](08-rag-document-store.md).

---

## Skill patches

A skill matched for this document and this section is **appended to the
prompt** as an extra instruction block, after the schema fragment and
RAG context but before the model is called.

A skill may also:
- declare additional `rag_queries` for the section,
- pre-populate fields it knows are constants for that insurer/product,
- declare **post-conditions** the verifier must check.

See [10 — Skills & HITL feedback](10-skills-and-hitl-feedback.md) for the
skill file format.

---

## Verifier

A deterministic, no-LLM check. For each field returned by a sub-agent:

| Check | Pass condition |
|---|---|
| Schema type | matches the JSON-schema declaration |
| Required-ness | required fields present and non-empty |
| Source span exists | `node_id` resolves to a real tree node |
| Quote-in-document | `quote` appears literally OR fuzzy-matches (≥ 0.9) inside the node's text |
| Offsets align | `char_start..char_end` ± 50 char around the quote match |
| Value ↔ quote | normalized value appears within ±200 chars of the matched quote |
| Skill post-conditions | (if any) pass |

A field gets `confidence` = weighted score; below a per-field-type
threshold, flag with a code:

- `schema_violation`
- `source_span_missing`
- `quote_not_in_document`
- `provenance_mismatch`
- `skill_assertion_failed`
- `low_confidence`

Reuses the algorithm from v1's `agents/verifier.py` but with stricter
quote-matching (v1 used 0.85 SequenceMatcher; v2 starts at 0.9 and
tunes from feedback data).

---

## Retry layer

Per flagged field, in order of cheapness:

1. **Local retry (free).** Re-parse / re-clean the value; sometimes a
   model returns `"value": "30 days"` but the span resolved to `"30
   day"`. Run a normalization pass before retrying.
2. **Same sub-agent, broader RAG.** Doubled top-K + extra retrieval
   queries pulled from a "fallback queries" list maintained per
   sub-agent.
3. **Same sub-agent, stronger model.** Promote `model_class` from
   `extraction` → `extraction_strong` (resolved by LLM gateway to a more
   expensive model). See
   [11 — LLM provider abstraction](11-llm-provider-abstraction.md).
4. **Different sub-agent.** A fallback "generic extractor" sub-agent
   that takes the field name + description + entire document tree slice
   and tries again from scratch.

Cap: 3 retries per field. After that, the field ships with
`needs_review = true` and an explanatory flag. The orchestrator surfaces
this to the UI and routes it into the prompt-review queue (see
[10](10-skills-and-hitl-feedback.md)).

---

## Synthesis (optional, last)

After verification, a tiny synthesizer sub-agent reads only the verified
fields — never the raw document — and produces a friendly narrative per
section ("In short, this policy waits 30 days before any non-accidental
claim is paid, and 36 months before pre-existing conditions are
covered..."). Citation is inherited from the underlying fields: the
synthesizer must reference fields by their `field_path`, and the UI
expands the path → source span at render time. See
[09 — Citation & grounding](09-citation-and-grounding.md).

If a section has many flagged fields, the synthesizer is **skipped** for
that section to avoid putting confident-sounding prose over uncertain
data.

---

## Performance shape

Rough targets for a 50-page health-insurance policy on default models:

| Phase | Wall time | Cost (approx.) |
|---|---|---|
| Ingest (no OCR) | 5–10 s | $0.001 |
| Ingest (with OCR) | 30–60 s | $0.01 |
| 12 sub-agents in parallel | 15–25 s | $0.10–0.20 |
| Verifier | < 1 s | $0 |
| Retry (≤ 30% of fields) | 5–15 s | $0.05–0.10 |
| Synthesis | 5–10 s | $0.02 |
| Total | **30–60 s** | **$0.18–0.34** |

Compare v1: 60–120 s analysis wall time, comparable cost, but lower
accuracy (no verifier, no retry, no section-scoped RAG). We expect v2
to be both faster (more parallelism, smaller prompts) and cheaper
overall (retries on a minority of fields cost less than re-running a
4k-token monolith).

---

## Backpressure & queueing

- The orchestrator places jobs on a Mongo-backed work queue (we'll
  consider Redis Streams if Mongo becomes a bottleneck).
- Workers consume jobs and run the pipeline above.
- The LLM gateway enforces per-provider rate limits; sub-agents queue
  inside the gateway if needed. See
  [11](11-llm-provider-abstraction.md).
- A single job's sub-agents may block on the gateway, but jobs do not
  serialize against each other; concurrency is bounded by gateway capacity,
  not job count.

---

## Idempotency

Two analyses of the same `(content_hash, domain, schema_version,
prompt_bundle_hash)` must produce identical results modulo nondeterminism
inside the LLM. Because we cache the final analysis, the second
"analysis" is a Mongo read.

For the LLM-touching first run, we set `temperature: 0.0` for extraction
sub-agents — non-zero only for synthesis (small effect on token sampling
keeps prose readable). This makes the pipeline maximally
reproducible without sacrificing prose quality.

---

## Telemetry per analysis

```json
{
  "analysis_id": "...",
  "content_hash": "sha256:...",
  "domain": "health",
  "schema_version": 1,
  "prompt_bundle_hash": "sha256:...",
  "skills_applied": ["niva_bupa__health_companion__2024", "..."],
  "sub_agents": [
    {
      "name": "waiting_periods",
      "model": "claude-haiku-4-5-20251001",
      "tokens_in": 1234,
      "tokens_out": 567,
      "wall_ms": 8421,
      "attempts": 1,
      "flags": []
    },
    ...
  ],
  "verifier": { "fields_total": 113, "fields_passed": 109, "fields_flagged": 4 },
  "retries": [{ "field_path": "exclusions.permanent_exclusions", "attempts": 2, "final_outcome": "succeeded" }],
  "wall_ms": 41280,
  "cost_usd": 0.22
}
```

Stored on the analysis document — see
[15 — Persistence & re-run guard](15-data-persistence-and-rerun.md).
