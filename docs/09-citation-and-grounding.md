# 09 — Citation & Grounding

> Every fact HIPA emits must be provable against the source document. The UI
> renders analyses side-by-side with the original PDF, with highlights linking
> each fact to its exact source span. No more "trust me, the LLM said so."

Cross-refs: [01 — Legacy observations](01-legacy-hipa-observations.md) (item 3),
[05 — Document ingestion](05-document-ingestion.md),
[06 — Extraction pipeline](06-extraction-pipeline.md),
[07 — Schema spec](07-extraction-schema-spec.md),
[13 — Q&A & explanation](13-qa-and-explanation.md),
[14 — Report generation](14-report-generation.md).

---

## The source-span contract

Every leaf field in an analysis carries:

```json
"source_span": {
  "node_id": "policy_wording/waiting_periods/initial",
  "page": 18,
  "char_start": 33212,
  "char_end": 33305,
  "quote": "no claim shall be admissible for any illness contracted within the first 30 days"
}
```

Plus the field's `value`, `confidence`, and `flags`. See
[07 — Schema spec](07-extraction-schema-spec.md).

The verifier in [06](06-extraction-pipeline.md) checks at minimum:

- the `node_id` exists in the document tree,
- the `quote` literally appears (or fuzzy-matches ≥ 0.9) at
  `[char_start, char_end]`,
- the `value` is reconcilable with the `quote`.

Any field that doesn't satisfy these is flagged and retried. Anything that
ships without a valid source span is a bug.

---

## Multiple spans per field

Some fields are stated in two places (schedule + wording). Some require
combining: e.g. `total_sum_insured` = `base_sum_insured` + `bonus`, and
both numbers come from different sentences.

The contract supports an array:

```json
"total_sum_insured": {
  "value": "₹6,00,000",
  "source_spans": [
    { "node_id": "schedule/sum_insured", "quote": "Base SI: ₹5,00,000" },
    { "node_id": "schedule/bonus", "quote": "Accrued NCB: ₹1,00,000" }
  ],
  "derivation": "base_sum_insured + accrued_ncb"
}
```

The optional `derivation` field is free-text the model may attach when the
value is computed; the UI surfaces it on hover.

---

## Citation index

Source spans are extracted out of the analysis JSON into a dedicated Mongo
collection at persist time:

```
citations {
  _id: ObjectId,
  analysis_id: "...",
  field_path: "waiting_periods.initial_waiting_period",
  index_in_field: 0,                       // for multi-span fields
  node_id: "policy_wording/waiting_periods/initial",
  page: 18,
  char_start: 33212,
  char_end: 33305,
  quote: "...",
  content_hash: "sha256:..."               // points to the doc
}
```

Why a separate collection: the citation viewer paginates over a
potentially-very-large list of spans, and bounding it on Mongo keeps the
analysis document compact and fast to load.

---

## UI — the side-by-side viewer

```
┌──────────────────────────────┬──────────────────────────────┐
│  Analysis (left)             │  Source PDF (right)          │
│  ──────────────────────      │  ──────────────────────      │
│  Section 4 — Waiting Periods │  [page 18 raster]            │
│                              │                              │
│  Initial waiting period      │  ┌─────────────────────────┐ │
│  30 days. Exempt: accidents. │  │ ░░░░░░░░░░░░░░░░░░░░░░  │ │
│  [📄 view source]  ←─────────────│ ░░ no claim shall be ░ ←──── highlight overlay
│                              │  │ ░░ admissible for any ░ │ │
│  PED waiting period          │  │ ░░ illness contracted ░ │ │
│  36 months                   │  │ ░░ within the first 30 ░│ │
│  [📄 view source]            │  │ ░░ days...           ░░ │ │
│                              │  └─────────────────────────┘ │
└──────────────────────────────┴──────────────────────────────┘
```

Mechanics:

- The UI loads the analysis (left) and the per-page WebP rasters from S3
  (right; rendered once at ingest time — see [05](05-document-ingestion.md)).
- Each fact card on the left has a "view source" affordance.
- Clicking it:
  1. fetches the citation(s) for that field from the citation index,
  2. navigates the right pane to the cited page,
  3. overlays a highlight rectangle. The rectangle's bounding box is
     pre-computed at ingest time and stored on the node, so the UI
     doesn't need to re-rasterize or run OCR.
- Hovering the highlight on the right reverse-resolves to the left card.

### Highlight rectangles

For native-text PDFs we have exact glyph positions from the extractor;
the rectangle is the union of glyph boxes between `char_start` and
`char_end`. For OCR'd pages we have word-level bounding boxes from
Tesseract; same union approach with a small padding.

Each tree node persists a `bbox_by_page` map:

```json
"bbox_by_page": {
  "18": [
    { "char_start": 33212, "char_end": 33305, "x": 72, "y": 410, "w": 470, "h": 22 }
  ]
}
```

The citation viewer is a thin renderer over this data — no compute at view
time.

---

## Grounding the *narrative*

Sub-agents return structured fields with source spans. The synthesizer
(see [06 — Phase D](06-extraction-pipeline.md)) writes prose, but only
from the verified field values. To preserve grounding, the synthesizer
emits prose with inline field references:

```
This policy waits ${field:waiting_periods.initial_waiting_period} before
any non-accidental claim is paid, and ${field:waiting_periods.ped_waiting_period}
before pre-existing conditions are covered.
```

At render time the UI:

1. Replaces each `${field:...}` token with the actual value.
2. Wraps it in an inline citation marker that, on click, behaves
   identically to the structured-field "view source" — looking up the
   span from the citation index.

Net effect: the narrative is grounded too, **transitively** through the
field references. Nothing in the prose can fabricate; it can only
re-arrange and explain what's already verified.

If a synthesizer attempts a claim it can't reduce to a field, the
verifier (a tiny token-level pass) flags the unbound claim and the prose
is regenerated.

---

## Q&A grounding

Q&A answers cite either:

- analysis fields (preferred, via the same `${field:...}` mechanism), or
- raw document spans (when the answer needs nuance the analysis fields
  don't capture).

Either way, the answer ships with a citation list; the chat UI renders
the same "view source" affordance as the analysis side. See
[13 — Q&A & explanation](13-qa-and-explanation.md).

---

## Report grounding

The PDF report (see [14](14-report-generation.md)) embeds cited text as
small footnote markers. Each marker points to a numbered footnote at the
bottom of the page, listing the source quote + page number. The
generated PDF is therefore self-auditable without access to the
original — useful for compliance.

---

## Hallucination defenses, layered

```
1. Sub-agent prompt: "every field MUST include a verbatim source quote;
                      return null if not in document"
2. Schema-aware parser: reject responses with missing source_span keys
3. Verifier: reject if quote not in document, value not near quote, etc.
4. Retry layer: re-run sub-agent with broader RAG / stronger model
5. Synthesizer can only reference fields, not raw text
6. Synthesizer's prose is unbound-claim-scanned before shipping
7. UI exposes "view source" for every fact — users self-verify in seconds
```

Defense in depth. Even if one layer slips, the user sees an unfilled
"view source" and reports it via the per-section feedback widget — which
becomes a skill or a prompt-review ticket. See
[10 — Skills & HITL feedback](10-skills-and-hitl-feedback.md).

---

## When to ship a field WITHOUT a citation

Never, except `advisor_assessment.*`. Those are computed, not extracted;
their "citation" is a list of field paths they depended on:

```json
"advisor_assessment.overall_policy_score": {
  "value": 78,
  "derived_from": [
    "sum_insured_and_premium.total_sum_insured",
    "exclusions.permanent_exclusions",
    "waiting_periods.ped_waiting_period",
    "...etc"
  ]
}
```

The UI renders this as a "see what went into this score" chip, expanding
into a list of contributing fields, each of which has its own real
citation. Recursion bottoms out at extracted (cited) fields.

---

## Failure mode: a fact has no real source

If, after retries, a required field genuinely cannot be sourced, the
analysis ships with `value: null`, `confidence: 0`, `flags:
["unrecoverable"]`. The UI surfaces it as **"Not stated"** with a
question-mark icon — no fabricated value, no hidden gap. The customer or
RM is left with an explicit unknown they can address (call the insurer,
upload an endorsement, etc.).

This is the single most important UX outcome of the grounding system.
The legacy HIPA hides uncertainty inside confident-sounding prose. v2 makes
uncertainty *visible*, which is what builds trust with users and
compliance reviewers alike.
