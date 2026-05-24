# 04 — Multi-Agent Orchestration

> The active design choice that fixes v1's biggest performance and accuracy
> problems: replace a single 4k-token "universal agent" with a tree of tight,
> single-purpose sub-agents that fan out in parallel, each grounded in a
> narrow slice of the document.

Cross-refs: [02 — Architecture overview](02-architecture-overview.md),
[03 — Modular domain system](03-modular-domain-system.md),
[06 — Extraction pipeline](06-extraction-pipeline.md),
[09 — Citation & grounding](09-citation-and-grounding.md),
[11 — LLM provider abstraction](11-llm-provider-abstraction.md).

---

## Three layers

```
            ┌─────────────────────────────┐
   L0       │  Orchestrator (core)        │   domain-agnostic
            │  pick domain · spawn agents │
            └─────────────┬───────────────┘
                          │
            ┌─────────────▼───────────────┐
   L1       │  Domain agent (per module)  │   knows the schema
            │  fan-out · verify · retry   │   coordinates sub-agents
            └─────────────┬───────────────┘
                          │  spawns N in parallel
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   L2  Sub-agent     Sub-agent         Sub-agent
      "policy_       "waiting_         "exclusions"
       identity"      periods"
   each fills one schema section
   each owns its prompt, RAG queries, tools
```

L0 is in `core/`. L1 is generic code parameterized by the domain's
`manifest.yaml`. L2 is the data — prompts + RAG queries + agent.yaml — that
lives inside the domain module.

---

## L0 — The orchestrator

A Go process consuming a job queue. Per job:

1. **Auto-detect domain** unless caller specified one.
   - Run cheap keyword classifiers from each registered domain in parallel.
   - If exactly one is confident (≥ 0.8) → pick it.
   - If multiple, run an LLM tiebreaker (small/cheap model class) with all
     candidates as choices.
   - If none → mark job as `needs_review` with reason `domain_unknown`.

2. **Cache lookup.** Key = `(content_hash, domain, schema_version,
   prompt_bundle_hash)`. Hit → return stored analysis. See
   [15 — Persistence](15-data-persistence-and-rerun.md).

3. **Load domain.** Reads `domains/<name>/manifest.yaml`, resolves
   sub-agent list, loads prompts, RAG queries, skill files.

4. **Match skills.** Skills whose `match:` block applies to this document
   (matched on insurer name, UIN, product name, or arbitrary regex) are
   attached to the relevant sub-agents. See
   [10 — Skills & HITL feedback](10-skills-and-hitl-feedback.md).

5. **Spawn the domain agent** with the resolved bundle.

6. **Persist** the result and source-span index. Mark job `succeeded` or
   `needs_review`.

The orchestrator is the only component that talks to the queue, the job
table, and the cache. Everything else is pure functions over already-loaded
state.

---

## L1 — The domain agent

For each job, the domain agent runs three phases.

### Phase A — Fan-out extraction

Spawn N sub-agents concurrently. N = number of sub-agents declared in
`manifest.yaml` (12 for health). Each gets:

```
SubAgentInput {
    schema_fragment: <slice of schema.json for this section>
    rag_context:     <top-K chunks from per-document index, scoped to this section's queries>
    skill_patches:   <list of skill overrides that matched this document & this section>
    tools_allowed:   <subset of manifest.required_tools + optional_tools>
    document_tree:   <pointer to the markdown tree; sub-agent can tool-call to fetch deeper>
    prompt:          <versioned prompt for this sub-agent>
    model_class:     <e.g. "extraction" — resolved to a model by LLM gateway>
}
```

Each sub-agent returns:

```
SubAgentOutput {
    section: "waiting_periods"
    fields: {
        initial_waiting_period: { value: "30 days", source_span: {...} },
        ped_waiting_period:     { value: "36 months", source_span: {...} },
        ...
    }
    notes:        <free-text notes the sub-agent wanted to leave for the synthesizer>
    tools_called: <audit trail>
}
```

There is no inter-sub-agent communication. Sub-agents are intentionally
independent so they can run on different machines / different model
providers / fail independently.

### Phase B — Verification

For every field returned:

1. **Schema validity.** Does the value match the schema type (string,
   array, etc.)? Required fields present?
2. **Source span exists** in the document tree, at the offsets given?
3. **Value ↔ quote alignment.** Does the value plausibly appear in or
   near the quote? (Reuse v1's `verifier.py` algorithm: substring or
   fuzzy match + ±200 char context window + keyword relevance score.)
4. **Skill assertions.** If a skill declared post-conditions (e.g.
   "`sum_insured` must be ≥ ₹3L for Corona Kavach") run them.

Each field gets a `confidence` and an optional list of `flags`.

### Phase C — Retry

For each `flag` that's recoverable (`source_span_missing`,
`low_confidence`, `provenance_mismatch`), the domain agent re-runs the
single sub-agent with:

- a **broader RAG slice** (top-K doubled, plus structural sections like
  the schedule and the table of contents),
- a **stricter prompt** (an extra instruction at the bottom: "the
  previous attempt produced these flags: ...; do not repeat them"),
- and a **stronger model class** (one tier up from the default).

Up to two retries per sub-agent. Anything still flagged ships to the UI
with `needs_review = true` and is included in the prompt-review queue.

### Phase D — Synthesis (optional)

A small layman-summary sub-agent reads the verified fields (not the raw
document) and produces a customer-friendly narrative for each section.
Because it cites the already-verified fields, it inherits their citations
transitively. See [09 — Citation & grounding](09-citation-and-grounding.md).

---

## L2 — Sub-agents

Each sub-agent is defined by a tiny directory:

```
sub_agents/waiting_periods/
├── agent.yaml
├── prompt.md
└── rag_queries.yaml
```

### `agent.yaml`

```yaml
name: waiting_periods
model_class: extraction          # cheap/fast model
max_tokens: 1500
temperature: 0.0
tools_allowed: []                # extraction agents usually need no tools
retry:
  max_attempts: 3
  on_flags: [source_span_missing, low_confidence, provenance_mismatch]
```

### `prompt.md`

```markdown
---
version: 3
description: Extract waiting-period fields with verbatim source quotes.
---

You are extracting **waiting periods** from a health insurance policy.

For each required field in the schema fragment below, return:
- `value`: the cleaned value (string, follow the schema description)
- `source_span`: a verbatim quote from the document text, plus its `node_id`

Rules:
1. If the field is not in the document, return `null`. Do NOT guess.
2. The `source_span.quote` must appear LITERALLY in the document. No paraphrasing.
3. Prefer the most specific section (e.g. "Section 4.3 — Waiting Periods")
   over surrounding text.
4. Maternity waiting period: if the policy is a male-only or non-floater,
   set the value to "Not applicable — no female members covered".

Schema fragment:
{{ schema_fragment_json }}

Document subtree:
{{ rag_context }}

{{ skill_patches_appended_here }}
```

### `rag_queries.yaml`

```yaml
- "waiting period initial 30 days 90 days"
- "pre-existing disease waiting period"
- "specific disease waiting period named diseases"
- "maternity benefit waiting period"
- "first year exclusion second year exclusion"
- "continuity benefits portability prior policy"
```

These are the BM25/embedding queries used to retrieve the section-scoped
RAG slice. They are deliberately overlapping — the retrieval layer
deduplicates chunks across queries before sending to the sub-agent.

---

## Why this design

| Problem in v1 | How v2 fixes it |
|---|---|
| 4k-token monolithic prompt loses focus | 12 sub-prompts of ~500 tokens each |
| Adding a section means editing one giant Python string | Add a new sub-agent directory |
| A/B testing a prompt requires a new constant + branch | Version the prompt file; orchestrator logs prompt hash |
| Failures are silent | Verifier flags failures; retry layer recovers most; rest visible in UI |
| Insurer quirks coded inline | Skills patch prompts/RAG/post-conditions at match time |
| No grounding for narrative | Synthesizer reads only verified, cited fields |

---

## Concurrency budget

The fan-out is unbounded by design but the LLM gateway enforces:

- per-provider rate limit (RPM, TPM),
- per-job concurrency cap (default 16),
- per-account QPS.

Sub-agents queue inside the gateway. From the domain agent's perspective
it's all one `await Promise.all`-equivalent; the gateway is where
back-pressure lives.

See [11 — LLM provider abstraction](11-llm-provider-abstraction.md).

---

## Failure semantics

| What failed | What the orchestrator does |
|---|---|
| Single sub-agent returns invalid JSON | Retry up to `retry.max_attempts`. After that, mark fields as `extraction_failed`; do not block other sub-agents. |
| Sub-agent times out | Same as invalid JSON. |
| Verifier rejects all fields of a sub-agent | Retry once with broader RAG + stronger model. |
| Whole domain has > 30% flagged fields | Mark job `needs_review`; ship partial result. |
| Orchestrator process dies mid-job | Job stays in `running`; a stalled-job sweeper (per minute) re-queues anything older than `n × max_attempt_duration`. |

Crucially: **a sub-agent failure never blocks siblings**. The UI shows
partial results with a "this section needs review" banner. Compare to v1's
synthesis call — if it failed, the whole analysis was lost.

---

## Auditability

Every job persists:

- the domain version + schema version,
- the prompt-bundle hash (one hash over all sub-agent prompt files used),
- the skill list applied,
- per-sub-agent: model id, token counts, latency, attempt count, final flags,
- per-field: source span(s) and confidence.

Reproducing a past analysis means re-running with the same prompt-bundle
hash and the same model id. See
[15 — Persistence & re-run guard](15-data-persistence-and-rerun.md).
