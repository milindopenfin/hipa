# 07 — Extraction Schema Spec

> The contract between the orchestrator, every sub-agent, the verifier, the
> Q&A layer, and the report renderer. A domain's schema defines what "fully
> analyzed" means for that domain.

Cross-refs: [`health_insurance_extraction.json`](health_insurance_extraction.json),
[03 — Modular domain system](03-modular-domain-system.md),
[04 — Multi-agent orchestration](04-multi-agent-orchestration.md),
[06 — Extraction pipeline](06-extraction-pipeline.md),
[09 — Citation & grounding](09-citation-and-grounding.md).

---

## Schema = JSON Schema + grounding contract

We use plain JSON Schema (draft 2020-12) as the base, with a tiny set of
extensions:

1. **Every leaf carries `value` + `source_span` at runtime.** The schema
   author writes the *intended* type; the orchestrator wraps it.
2. **Field descriptions are part of the prompt.** The `description` on
   each property is fed directly to the sub-agent prompt as the "what to
   look for" hint.
3. **Top-level keys partition sub-agents.** Each top-level object becomes
   one sub-agent's section. See
   [04 — Multi-agent orchestration](04-multi-agent-orchestration.md).

The canonical example is [`health_insurance_extraction.json`](health_insurance_extraction.json)
(406 lines, lifted from v1's `agents/policy_extraction_schema.json`).

---

## Top-level structure (health module)

```
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "policy_identity":                       { ... },
    "policyholder_and_members":              { ... },
    "sum_insured_and_premium":               { ... },
    "hospitalisation_and_core_benefits":     { ... },
    "value_added_and_wellness_benefits":     { ... },
    "waiting_periods":                       { ... },
    "financial_limits_copay_deductibles":    { ... },
    "network_claims_tpa":                    { ... },
    "exclusions":                            { ... },
    "renewal_and_continuity":                { ... },
    "riders_and_addons":                     { ... },
    "advisor_assessment":                    { ... }
  }
}
```

12 top-level keys → 12 sub-agents in the health module. Add a key to the
schema, add a sub-agent directory; the orchestrator picks it up.

---

## Author-time vs. runtime

The schema author writes:

```json
"initial_waiting_period": {
  "type": "string",
  "description": "Initial / first-year waiting period (30/90 days) and exempt illnesses."
}
```

A sub-agent returns at runtime:

```json
"initial_waiting_period": {
  "value": "30 days. Exempt: accidents and emergency hospitalisation.",
  "source_span": {
    "node_id": "policy_wording/waiting_periods/initial",
    "page": 18,
    "char_start": 33212,
    "char_end": 33361,
    "quote": "no claim shall be admissible ... within the first 30 days ... except injuries caused by accident"
  },
  "confidence": 0.94,
  "flags": []
}
```

The wrapper (`value` / `source_span` / `confidence` / `flags`) is applied
**uniformly** by the core. Authors never write it.

---

## Required vs. optional

JSON Schema's `required` array on each object identifies must-have
fields. Behavior:

- Required field absent → verifier flag `schema_violation`.
- Optional field absent → emitted as `null` in the output.
- Required field with low confidence → flag `low_confidence`, but
  result still shipped.

The `required` list per section is the same as v1's — already curated for
real Indian-market policies.

---

## Array fields

Members (one per insured), riders (one per add-on), continuity benefits
(one per member) — these are arrays of objects. The schema declares the
shape of each item; the sub-agent returns an array of items, each with
its own `value` (for primitive fields) and own `source_span`.

```json
"members": {
  "type": "array",
  "items": { "type": "object", "properties": { ... } }
}
```

At runtime each member object's leaf fields carry their own source spans.
The "member object" itself doesn't get a source span — its leaves do.

---

## Conditional requirements

JSON Schema's `if`/`then` and `dependentRequired` are allowed. Example:

```json
"maternity_benefit": { ... },
"maternity_waiting_period": { ... },
"dependentRequired": {
  "maternity_benefit": ["maternity_waiting_period"]
}
```

If maternity is covered, the waiting period becomes required. The
verifier honors this.

For things JSON Schema can't express cleanly (e.g. "if plan_type ==
'Floater', members.length ≥ 2"), use **skill post-conditions** instead
(see [10](10-skills-and-hitl-feedback.md)).

---

## Versioning

Two version numbers travel with the schema:

- `schema_version` (integer): bumps whenever **any** field is added,
  removed, renamed, or its type changes.
- `prompt_bundle_hash` (sha256): not part of the schema file but derived
  from the sub-agent prompt files alongside; bumps automatically.

Both are persisted with every analysis. The cache key is
`(content_hash, domain, schema_version, prompt_bundle_hash)` so that a
schema update or prompt update **invalidates the right slice of cache**
without nuking everything. See
[15 — Persistence & re-run guard](15-data-persistence-and-rerun.md).

### Migration

When `schema_version` bumps:

1. Existing analyses are not auto-recomputed.
2. The new schema includes a `migrations.yaml` describing how to map old
   fields to new ones (rename, copy, default, drop). A background job
   produces a "migrated view" of old analyses for the UI to render
   without re-billing the LLM. Re-extraction is opt-in.

---

## Why this schema, not Pydantic / Protobuf / Avro

| Option | Why not |
|---|---|
| **Pydantic** | Python-locked. The core is Go in v2. |
| **Protobuf** | Field descriptions are second-class. We feed descriptions to the LLM. |
| **Avro** | Same — focused on serialization, not human/LLM contracts. |
| **JSON Schema** | First-class descriptions, ubiquitous tooling, language-agnostic, browser-friendly for the UI's schema-aware renderers. |

The cost is that JSON Schema has weak expressiveness for cross-field
constraints. That gap is filled by **skill post-conditions** — small,
declarative, scoped to where they apply.

---

## Skill-augmented schemas

The schema is the **base** contract. A skill can:

- declare **extra fields** for a particular insurer's product (e.g.
  Niva Bupa's `safeguard_plus_si`),
- **constrain** an existing field (e.g. "for product `ICIHLIP21107V012021`,
  `sum_insured` must be one of [500000, 750000, 1000000]"),
- **default** a value (e.g. "for Star Health Family Health Optima, set
  `claim_administrator = 'Star Health TPA Services'` if not extractable").

Skill-declared extra fields are emitted under a `skill_fields:` namespace
in the output, not in the base schema, so they never collide with the
canonical contract. See [10](10-skills-and-hitl-feedback.md).

---

## Reading the health schema

The MVP schema includes (per `health_insurance_extraction.json`):

| Section | Required leaves (count) | Notes |
|---|---|---|
| policy_identity | 9 | UIN, dates, GSTIN |
| policyholder_and_members | varies | members[] each with own required leaves |
| sum_insured_and_premium | 5 | SI total, net/gross premium |
| hospitalisation_and_core_benefits | 10 | room rent, ICU, ambulance, AYUSH |
| value_added_and_wellness_benefits | 6 | restoration, NCB, maternity, PA, CI |
| waiting_periods | 4 | initial, PED, specific, maternity |
| financial_limits_copay_deductibles | 5 | copay, deductible, sub-limits |
| network_claims_tpa | 5 | TPA, cashless, portability |
| exclusions | 2 | permanent_exclusions, key_exclusions |
| renewal_and_continuity | 3 | renewal guarantee, grace, free look |
| riders_and_addons | 1 | riders_opted[] |
| advisor_assessment | derived | scored from the above |

`advisor_assessment` is special: its fields aren't extracted directly
from the policy. They're **derived** by a small reasoning sub-agent that
reads the verified extraction (not the raw doc) and produces a score,
key strengths, and gaps. Its citations refer to **other field paths**,
not raw document spans — see
[09 — Citation & grounding](09-citation-and-grounding.md).

---

## Schema authoring rules

1. **Each leaf has a `description`.** This is shown to the LLM and to
   reviewers. Be explicit; ambiguity in the description = noise in the
   extraction.
2. **Prefer string over enum.** Real-world policies say "30 days", "30
   Days", "thirty days", "30 (Thirty) Days." Normalize at the *display*
   layer, not the *extraction* layer.
3. **Don't model the form, model the meaning.** "room_rent_limit" not
   "field_14_column_b."
4. **Arrays for repeating entities, not for multi-valued strings.** "Members
   is an array of objects" — fine. "Exclusions is an array of strings" — fine.
   "Riders is an array of riders, each with name and limit" — fine.
   "Sum insured is sometimes string sometimes array" — not fine.
5. **Top-level keys are stable.** Renaming a top-level key forces a
   sub-agent rename. Add and deprecate; never rename in place.

---

## What's intentionally not in the schema

- **Layman explanations.** These are produced by the synthesizer, derived
  from the schema. See [13](13-qa-and-explanation.md).
- **Comparisons to other policies.** Computed at query time by the Q&A
  layer; not stored on the analysis.
- **Scores.** `advisor_assessment.overall_policy_score` is derived; the
  formula lives in the advisory sub-agent, versioned with prompts.
- **Customer profile data.** That's the Q&A flow's output, kept separate
  from the policy analysis. See [13](13-qa-and-explanation.md).

The schema is *what the policy says*, not *what the customer needs*.
