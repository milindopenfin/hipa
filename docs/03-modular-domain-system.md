# 03 — Modular Domain System

> This is the most important architectural property of HIPA v2. The platform is
> not "a health insurance analyzer that one day will support other lines." It's
> "a generic insurance-document analyzer whose first module happens to be
> health." Adding fire, motor, life or marine is a directory drop, not a refactor.

Cross-refs: [02 — Architecture overview](02-architecture-overview.md),
[04 — Multi-agent orchestration](04-multi-agent-orchestration.md),
[07 — Extraction schema spec](07-extraction-schema-spec.md),
[10 — Skills & HITL feedback](10-skills-and-hitl-feedback.md).

---

## The contract

A **domain module** is a self-contained directory under `domains/`. It exposes
a manifest and a fixed set of artifacts; the orchestrator knows how to
consume them and does not need to know what the domain is *about*.

```
domains/
├── health/
│   ├── manifest.yaml               # required: declares the module
│   ├── schema.json                 # required: canonical extraction shape
│   ├── sub_agents/                 # required: one folder per sub-agent
│   │   ├── policy_identity/
│   │   │   ├── agent.yaml          # which model class, which tools, retries
│   │   │   ├── prompt.md           # versioned prompt template
│   │   │   └── rag_queries.yaml    # retrieval keywords for this section
│   │   ├── policyholder_and_members/
│   │   ├── sum_insured_and_premium/
│   │   ├── hospitalisation_and_core_benefits/
│   │   ├── value_added_and_wellness_benefits/
│   │   ├── waiting_periods/
│   │   ├── financial_limits_copay_deductibles/
│   │   ├── network_claims_tpa/
│   │   ├── exclusions/
│   │   ├── renewal_and_continuity/
│   │   ├── riders_and_addons/
│   │   └── advisor_assessment/
│   ├── skills/                     # per-insurer, per-product overrides
│   │   ├── icici_lombard__corona_kavach__2020.yaml
│   │   └── ...
│   ├── question_bank.json          # post-analysis Q&A questions (data-driven)
│   ├── report_template/            # HTML/CSS for PDF reports
│   │   ├── template.html
│   │   ├── style.css
│   │   └── partials/
│   ├── classifier/                 # detect whether an uploaded doc is health
│   │   ├── keywords.yaml
│   │   └── prompt.md
│   └── tools/                      # optional domain-specific tools
│       └── ...
├── fire/                           # future — same shape, different content
└── motor/                          # future — same shape, different content
```

Nothing in `core/` mentions `health`, `fire`, or `motor`. The orchestrator
discovers modules at startup by scanning `domains/*/manifest.yaml`.

---

## `manifest.yaml`

```yaml
name: health
display_name: "Health Insurance"
version: 0.1.0
schema_version: 1
schema_path: schema.json
classifier:
  keywords: classifier/keywords.yaml
  prompt: classifier/prompt.md
sub_agents:
  - policy_identity
  - policyholder_and_members
  - sum_insured_and_premium
  - hospitalisation_and_core_benefits
  - value_added_and_wellness_benefits
  - waiting_periods
  - financial_limits_copay_deductibles
  - network_claims_tpa
  - exclusions
  - renewal_and_continuity
  - riders_and_addons
  - advisor_assessment
report_template: report_template/template.html
question_bank: question_bank.json
skills_dir: skills/
required_tools:
  - system_time
  - master_library_lookup
optional_tools:
  - pincode_lookup
  - web_search
```

That's it. The orchestrator can load and run this module with no further
knowledge.

---

## Module-level isolation rules

1. **Schemas live in their module.** No global "insurance schema." Each
   domain owns its `schema.json` and bumps its own `schema_version`. See
   [07](07-extraction-schema-spec.md).
2. **Sub-agents are domain-local.** A sub-agent in `health/` cannot import
   from `motor/`. Shared utilities live in `core/` or `platform/`.
3. **Skills are scoped to a module.** A skill that fixes a quirk in
   Niva Bupa's Health Premia 2024 wording is health-only. Cross-domain
   skills are forbidden; if a pattern is truly cross-domain, it goes into
   `core/` as a platform feature instead.
4. **Prompts are versioned per sub-agent.** Each prompt file has a
   `version:` front-matter line; analyses log the prompt hash for
   reproducibility.
5. **Report templates are module assets.** A health PDF report has nothing
   in common with a fire PDF report; the renderer just executes whichever
   template the module declares.

---

## How a new domain gets added

Concrete walkthrough for adding `domains/fire/`:

1. **Author a schema.** `domains/fire/schema.json` — the fire-policy
   equivalent of the health schema. Fields like `sum_insured`,
   `peril_coverage`, `excluded_perils`, `building_age`, `valuation_basis`.
2. **Write a classifier.** A regex/keyword file plus a one-shot LLM prompt
   that decides "is this PDF a fire insurance policy?". The orchestrator
   uses classifiers to **auto-select** a domain when the user uploads
   without specifying. See [04 — Multi-agent orchestration](04-multi-agent-orchestration.md).
3. **Create sub-agents.** One per top-level schema section. Each gets a
   prompt template scoped to that section and a `rag_queries.yaml` of
   retrieval keywords vocabulary that matches fire-policy wording.
4. **Write a report template.** HTML/CSS, lays out the fields the
   underwriter / customer care about.
5. **Seed `skills/`** with empty directory; reviewers will populate it as
   real fire policies expose insurer quirks.
6. **Register tools.** If fire needs a `weather_risk_lookup` tool, ship it
   under `domains/fire/tools/` and declare it in `manifest.yaml`.
7. **Drop the directory into `domains/`.** The orchestrator's next start
   picks it up. No core code changes.

This is the **acceptance test for modularity**: if step 7 fails, the
abstraction is leaky and the leak must be fixed before more modules ship.

---

## What the core *does* know about

The core is allowed to know about:

- **The shape of `manifest.yaml`** and the discovery rules.
- **The schema spec format** (a JSON-schema dialect with `value` +
  `source_span` requirements). See [07](07-extraction-schema-spec.md).
- **The sub-agent contract** (input: schema fragment + RAG context;
  output: filled schema fragment with source spans). See
  [04](04-multi-agent-orchestration.md).
- **The skill file format**. See [10](10-skills-and-hitl-feedback.md).
- **The prompt versioning convention** — every prompt file starts with
  YAML front-matter declaring `version`, `model_class`, `max_tokens`.

The core does **not** know:

- Anything about insurance products, perils, sum insured, premiums, etc.
- The names of insurers.
- The visual layout of any report.
- Which fields are mandatory for a given line of business.

If you ever find yourself adding a special case in the orchestrator for a
domain, that's a defect — push the logic into the module.

---

## Health module — what's pre-built for MVP

The health module ships with:

- `schema.json`: the existing
  [`health_insurance_extraction.json`](health_insurance_extraction.json),
  promoted to be the canonical schema for v2.
- 12 sub-agents matching the 12 top-level keys in the schema. See
  [04](04-multi-agent-orchestration.md) for the per-sub-agent contract.
- Classifier keywords drawn from v1's `router.py` insurer list, plus
  health-specific terms (sum insured, hospitalisation, PED, ICU, copay).
- Insurer-specific skills for the 16 insurers v1 supported, each carrying
  the table-layout hints currently inline in
  `hipa/agents/universal_agent.py` and `structural_extractor.py`.
- A report template covering the 12 sections.

---

## What the user-facing surface looks like

```
POST /api/v1/upload
  → { "file_id": "...", "detected_domain": "health", "domain_confidence": 0.97 }

POST /api/v1/analyze
  body: { "file_id": "...", "domain": "health" }    # domain optional; defaults to detected
  → { "job_id": "..." }

GET /api/v1/jobs/{job_id}
  → { "state": "succeeded", "domain": "health", "schema_version": 1, "result_url": "..." }
```

The same three endpoints serve every domain. The UI fetches the result and
renders it with the **domain's report template** — there is no
domain-specific frontend code. (The Q&A and citation viewer are
domain-agnostic; they render whatever the schema says exists.)

---

## Anti-patterns to avoid

- **Don't share sub-agents across domains.** If two domains "almost" use
  the same prompt, copy it. Duplication is cheaper than a leaky abstraction.
- **Don't put insurer names in `core/`.** All insurer enumeration belongs to
  a domain.
- **Don't hardcode field names in the orchestrator.** It iterates over the
  schema; it doesn't look up specific keys.
- **Don't auto-detect across domains by trying every classifier.** Run a
  cheap top-level classifier first that returns the *domain*, then the
  domain's own internal classifier disambiguates product/insurer. (Two-tier
  classification; see [04](04-multi-agent-orchestration.md).)
