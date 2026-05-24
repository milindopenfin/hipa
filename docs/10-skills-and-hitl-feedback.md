# 10 — Skills & Human-in-the-Loop Feedback

> The system will get things wrong. Two questions: how does a human fix the
> wrong thing, and how does the fix scale to every future analysis that would
> have hit the same wrong thing?
>
> Answer: **skills** for policy/insurer/product-specific overrides,
> **versioned prompts** for the everything-else case, and a triage UI that
> routes every piece of feedback into one or the other.

Cross-refs: [03 — Modular domain system](03-modular-domain-system.md),
[04 — Multi-agent orchestration](04-multi-agent-orchestration.md),
[06 — Extraction pipeline](06-extraction-pipeline.md),
[09 — Citation & grounding](09-citation-and-grounding.md).

---

## Two failure shapes, two fixes

| Shape | Example | Fix |
|---|---|---|
| **Systemic** — the prompt is wrong or weak for *every* policy | "Sub-agent confuses `pre-hospitalisation_days` with `post-hospitalisation_days` on policies that list them as a single combined number" | Edit the prompt template; bump version |
| **Specific** — a particular insurer / product / wording violates the prompt's general assumption | "Niva Bupa's `Safeguard+` SI sits in column 4 of the member table, not in `sum_insured_and_premium`" | Author a skill scoped to that insurer / product |

Both flow through the same review queue, but the resolution is different.

---

## Skill files

A skill is a YAML file under `domains/<domain>/skills/`. It declares
**what document(s) it applies to** and **what overrides to apply**.

```yaml
# domains/health/skills/niva_bupa__health_companion__2024.yaml
name: niva_bupa__health_companion__2024
version: 2
created_by: avik@johnsonrealestate.co
created_at: 2026-05-10
description: |
  Niva Bupa's Health Companion 2024 places Safeguard+ and Booster+ sum insureds
  in the member-level table rather than the policy-level sum_insured block.
  Without this skill, sub-agent `sum_insured_and_premium` reports only the base SI.

match:
  insurer: "NIVA BUPA"
  uin_regex: "^NBHLIP24.*"
  product_name_contains: ["Health Companion"]

applies_to:
  - sum_insured_and_premium
  - policyholder_and_members

overrides:
  policyholder_and_members:
    extra_fields:
      - name: safeguard_plus_si
        type: string
        description: "Safeguard+ rider SI (Niva Bupa)."
      - name: booster_plus_si
        type: string
        description: "Booster+ rider SI (Niva Bupa)."
    rag_queries_append:
      - "safeguard plus sum insured"
      - "booster plus sum insured"
    prompt_append: |
      ADDITIONAL CONTEXT FOR NIVA BUPA HEALTH COMPANION 2024:
      The member-level table has columns: Member | Base SI | Safeguard+ SI | Booster+ SI | Total.
      Always extract all four sub-totals per member into the corresponding fields.

  sum_insured_and_premium:
    post_conditions:
      - id: total_si_matches_member_sum
        expr: |
          total_sum_insured == sum(members[*].total_sum_insured)
        on_fail_flag: skill_assertion_failed
```

### What a skill can do

- `match`: select which documents this skill applies to. Operators:
  `insurer`, `uin_regex`, `product_name_contains`, `node_text_contains`
  (last is rare; reserved for products without a clean UIN).
- `applies_to`: list of sub-agent names (= schema top-level keys).
- `overrides`:
  - `prompt_append`: extra instructions appended to the sub-agent prompt.
  - `rag_queries_append`: extra retrieval queries.
  - `force_include_nodes`: tree nodes always retrieved when this skill is active.
  - `extra_fields`: schema additions emitted under `skill_fields:`.
  - `defaults`: field defaults applied if extraction returns null.
  - `value_normalizers`: regex/replacement rules for known dirty formats.
  - `post_conditions`: deterministic assertions the verifier must check.

### What a skill *can't* do

- Modify the canonical schema (only add `skill_fields`).
- Change which sub-agent runs.
- Reach across domains.
- Run arbitrary code. (Post-condition expressions are a tiny safe
  expression language: comparisons, arithmetic, regex, sum/min/max
  aggregators. No function calls, no I/O.)

---

## Skill lifecycle

```
       feedback
         │
         ▼
   reviewer triage
       /     \
      /       \
     v         v
  prompt    skill draft  ← reviewer fills + tests on the failing doc
  edit         │
  (new         ▼
  version)  approval
              │
              ▼
          shipped to skills/
              │
              ▼
   matched by orchestrator on every future analysis
```

Skills are **versioned** (file name carries no version; the `version:`
key inside does). Bumping a skill version invalidates the analysis cache
for any document the skill matches — those docs will re-analyze when
next requested.

---

## Skill matching (orchestrator)

At job start, before sub-agent fan-out:

1. Quick metadata pre-filter: read the doc's `policy_identity` from a
   cheap, no-skill first pass (only `insurer_name`, `uin`,
   `product_name`).
2. Walk the domain's `skills/` directory, evaluate each skill's `match`
   block against the metadata.
3. Build a `{ sub_agent_name: [applicable_skill, ...] }` map.
4. When dispatching each sub-agent, attach its applicable skills' patches.

The "cheap first pass" is itself a tiny sub-agent (single LLM call,
single schema fragment) that only fills `insurer_name`, `uin`,
`product_name`. Its skill list is empty — it bootstraps the rest.

---

## Versioned prompts

Every sub-agent prompt is a markdown file with YAML front-matter:

```markdown
---
version: 7
description: Extract waiting-period fields with verbatim source quotes.
changelog:
  - v7 (2026-05-12): clarified maternity not-applicable handling
  - v6 (2026-04-30): forced verbatim quote per field
---
You are extracting waiting periods from a health insurance policy. ...
```

`prompt_bundle_hash` = sha256 over all sub-agent prompt files used in an
analysis. Stored on every analysis (see [15](15-data-persistence-and-rerun.md)).

Editing a prompt and bumping its version automatically:

- changes the bundle hash,
- invalidates the analysis cache for that prompt's sub-agent (analyses
  involving other sub-agents' caches remain valid — the cache is keyed
  on bundle hash but matched at per-sub-agent granularity for
  partial-rerun efficiency).

Rollback is `git revert` on the prompt file. The orchestrator picks up
the previous version on next analysis.

---

## Feedback widget (UI)

Identical surface to v1, upgraded with triage actions.

Each analysis section has:

- 👍 / 👎 buttons
- A free-text "what's wrong?" box
- A **what kind of problem is it?** chip selector:
  - `Wrong value for this field` → opens skill-draft modal
  - `Prompt is unclear in general` → opens prompt-review ticket
  - `Missing context / wrong section retrieved` → opens RAG-tuning ticket
  - `Other / I'm not sure` → goes to a generic queue
- A **per-field** thumbs option in the side-by-side view: a 👎 on a
  single field opens the skill draft pre-filled with that field's
  current value, expected value, and the failing document context.

A 👎 with no chip selection is OK — it lands in the generic queue for
later triage. The point is to never lose the signal.

---

## Skill-draft modal (UI)

Pre-filled from the failing analysis:

```
Document:          POLICY_COPY_X.pdf   (UIN: ICIHLIP21107V012021)
Section:           sum_insured_and_premium
Field:             total_sum_insured
Extracted value:   ₹5,00,000
What should it be: [        ]
Why is it wrong:   [text area...]

Apply to:  ○ This document only        (creates per-doc override — rare)
           ● This UIN                  (matches uin == ICIHLIP21107V012021)
           ○ This insurer (broader)
           ○ Custom matcher (advanced)

[Generate skill YAML]   [Save as draft]   [Submit for approval]
```

Behind the scenes the modal authors a YAML skill file in
`skills/_draft/`, attaches the failing document for regression testing,
and submits the draft for an admin to approve.

---

## Approval flow

A queue UI (admin-only) shows pending skill drafts and prompt-review
tickets. Per item the reviewer can:

- **Run** the draft against the failing document(s) to verify it fixes
  the field without regressing siblings.
- **Run** the draft against a regression suite of N other documents (we
  seed this from the master library; specifically, all docs matching the
  skill's `match` block, plus 5 random docs that *don't* match — to
  ensure the skill doesn't accidentally over-match).
- **Diff** the analysis output before vs after.
- **Approve** → moves the file from `skills/_draft/` to `skills/`,
  versions it, invalidates cached analyses that match.
- **Reject** with a comment → returns to the submitter.

The same UI handles prompt edits: side-by-side prompt diff + regression
suite + approval.

---

## Why not auto-apply skills without review?

Two reasons:

1. **Liability.** Skills change policy interpretations. A bad skill
   could mis-read every Star Health policy uploaded that day. Human
   review is the gate.
2. **Generalization risk.** A skill authored from one document may
   over-fit. The regression-suite run during review catches this.

The cost of human review is small relative to the cost of a wrong
analysis going out to a customer.

---

## Data shapes

### `feedback`

```
{
  _id, analysis_id, user_id, section, field_path?,
  rating: "up" | "down",
  comment: <string>,
  problem_kind: "wrong_value" | "prompt_unclear" | "rag_miss" | "other" | null,
  created_at,
  triaged_to: "skill:<draft_id>" | "prompt_review:<ticket_id>" | null,
  status: "new" | "triaged" | "resolved"
}
```

### `skill_drafts`

```
{
  _id, domain, name, yaml_body,
  failing_analysis_ids: [...],
  submitted_by, submitted_at,
  regression_results: { passed: [...], regressed: [...] },
  status: "draft" | "in_review" | "approved" | "rejected",
  reviewer, reviewed_at, review_comment
}
```

### `prompt_reviews`

```
{
  _id, domain, sub_agent_name,
  current_version, proposed_version, proposed_body,
  failing_analysis_ids: [...],
  diff,
  regression_results,
  status, ...
}
```

All stored in Mongo. See [15 — Persistence](15-data-persistence-and-rerun.md).

---

## Feedback metrics dashboard (post-MVP, sketch)

The data exists; the dashboard is a downstream concern:

- Thumbs-down rate per sub-agent over time (does prompt v7 reduce v6's
  errors?)
- Skill coverage (how many analyses match at least one skill?)
- Skill churn (how often are skills edited; sign of brittle matching)
- Per-insurer accuracy proxy (thumbs-down rate)

These guide where to invest improvement effort.

---

## Bottom line

A wrong analysis is OK once. The system must learn from it. **Skills**
fix policy-specific drift cheaply and locally; **prompt versioning**
fixes systemic drift globally and auditably; **the review queue** is the
single funnel both flow through. No fix is silent; every fix is
reproducible; every regression-test pass is recorded.
