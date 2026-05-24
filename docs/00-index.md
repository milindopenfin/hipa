# HIPA v2 — Architecture & Planning Docs

> A modular, multi-agent insurance policy analyzer.
> Health insurance is the **first** module; the platform is built so that any
> future line (fire, motor, life, marine, …) can be added as a plug-and-play
> domain package without touching the core.

This `docs/` folder is the planning index. It captures the design decisions,
the trade-offs, and the rationale behind HIPA v2 before any code is written.
The companion file [`health_insurance_extraction.json`](health_insurance_extraction.json)
is the canonical schema for the first module and is referenced throughout.

---

## How to read

If you're new, read in order. If you're hunting a specific decision, jump
straight to the topic — every doc links to the others it touches.

| # | File | What it answers |
|---|---|---|
| 00 | **This file** | Index, reading order, conventions |
| 01 | [Legacy HIPA observations](01-legacy-hipa-observations.md) | What v1 does, where it breaks, what we keep |
| 02 | [Architecture overview](02-architecture-overview.md) | Layers, components, end-to-end flow |
| 03 | [Modular domain system](03-modular-domain-system.md) | Plug-and-play package format (health / fire / motor …) |
| 04 | [Multi-agent orchestration](04-multi-agent-orchestration.md) | Orchestrator → category agent → sub-agents |
| 05 | [Document ingestion](05-document-ingestion.md) | PDF → markdown tree, OCR, password unlock |
| 06 | [Extraction pipeline](06-extraction-pipeline.md) | Schema-driven, parallel, grounded extraction |
| 07 | [Extraction schema spec](07-extraction-schema-spec.md) | Schema format, value+source contract, versioning |
| 08 | [RAG pipeline (ingest → retrieve → ground)](08-rag-document-store.md) | Normalize · dedupe · metadata · version · hybrid (BM25+dense) · ANN+rerank · source confidence · constrained gen · citation backend · insufficient-evidence fallback |
| 09 | [Citation & grounding](09-citation-and-grounding.md) | Side-by-side highlight, hallucination guards |
| 10 | [Skills & HITL feedback](10-skills-and-hitl-feedback.md) | Policy-specific overrides, approval flow, feedback → skill |
| 11 | [LLM provider abstraction](11-llm-provider-abstraction.md) | Anthropic / OpenAI / OpenRouter / Ollama, model classes |
| 12 | [Agent tools](12-agent-tools.md) | Web search, system time, calculator, doc lookup |
| 13 | [Q&A & layman explanation](13-qa-and-explanation.md) | Post-analysis chat for customer + RM |
| 14 | [Report generation](14-report-generation.md) | HTML/CSS → headless Chromium → PDF |
| 15 | [Persistence & re-run guard](15-data-persistence-and-rerun.md) | Mongo, S3, dedupe, no expensive re-runs |
| 16 | [API & auth](16-api-and-auth.md) | REST surface, Better Auth, RBAC |
| 17 | [Tech stack](17-tech-stack.md) | Go, TS/React, Mongo, Chroma, infra |
| 18 | [Roadmap](18-roadmap.md) | Phased delivery; future modules |

---

## Cross-cutting principles

These principles bind the docs together. Whenever a design choice references
"the principles," it's pointing at this list.

1. **Modularity over generality** — every insurance line is a self-contained
   domain package; the core knows nothing about health, fire, or motor.
   See [03 — Modular domain system](03-modular-domain-system.md).
2. **No ungrounded claim** — every extracted field carries a verbatim source
   span. The UI must be able to highlight that span in the original document.
   See [09 — Citation & grounding](09-citation-and-grounding.md).
3. **Parallel-first** — the legacy pipeline's single-agent monolith is the
   thing we are explicitly rejecting. Fan-out is the default.
   See [04 — Multi-agent orchestration](04-multi-agent-orchestration.md) and
   [06 — Extraction pipeline](06-extraction-pipeline.md).
4. **Schema-first** — a hard, versioned JSON schema defines what "done" means
   for each domain. See [07 — Extraction schema spec](07-extraction-schema-spec.md).
5. **Human-in-the-loop is a first-class flow**, not a bolt-on. Reviewers can
   approve, reject, or attach a *skill* that fixes the same defect everywhere
   the rule applies. See [10 — Skills & HITL feedback](10-skills-and-hitl-feedback.md).
6. **No expensive re-runs** — analyses are cached by content hash; reports
   can be regenerated from stored intermediate state without re-billing the
   LLM. See [15 — Persistence & re-run guard](15-data-persistence-and-rerun.md).
7. **Provider-agnostic LLMs** — model selection (Anthropic / OpenAI /
   OpenRouter / Ollama) is config, not code. Per-role overrides allow
   cheap/fast for routers and strong/slow for synthesis.
   See [11 — LLM provider abstraction](11-llm-provider-abstraction.md).

---

## Glossary

| Term | Meaning in this codebase |
|---|---|
| **Domain** | An insurance line packaged as a plug-and-play module (health, fire, motor). See [03](03-modular-domain-system.md). |
| **Schema** | The canonical JSON shape that an analysis must fill, per domain. See [07](07-extraction-schema-spec.md). |
| **Skill** | A small, scoped override (prompt patch, regex, rule) attached to a specific policy / insurer / product. See [10](10-skills-and-hitl-feedback.md). |
| **Orchestrator** | The top-level agent that picks a domain and dispatches sub-agents. See [04](04-multi-agent-orchestration.md). |
| **Sub-agent** | A scoped agent that fills one section / one field group, in parallel with siblings. See [04](04-multi-agent-orchestration.md). |
| **Source span** | `{document_id, page, char_start, char_end, quote}` — proof that a field came from the document. See [09](09-citation-and-grounding.md). |
| **Master library** | The IRDAI-style searchable archive of every policy doc ever seen, keyed by UIN + insurer + year. See [08](08-rag-document-store.md). |

---

## File naming & conventions

- Files are prefixed `NN-` so the natural sort matches the reading order.
- Cross-references are markdown links to sibling files.
- Diagrams are ASCII unless a real diagram adds something words can't.
- This index stays under one screen; deep detail lives in the per-topic files.
