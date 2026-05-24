# 14 — Report Generation

> An analysis isn't useful as raw JSON. The customer (or RM, or compliance
> reviewer) wants a polished, citation-rich PDF report. HTML + CSS, rendered
> via headless Chromium, is the most customizable, debuggable, and cheap path.

Cross-refs: [03 — Modular domain system](03-modular-domain-system.md),
[06 — Extraction pipeline](06-extraction-pipeline.md),
[09 — Citation & grounding](09-citation-and-grounding.md),
[15 — Persistence & re-run guard](15-data-persistence-and-rerun.md).

---

## Why HTML → PDF, not a PDF library

| Approach | Verdict |
|---|---|
| `reportlab` / `pdfkit` programmatic | Painful for non-trivial layouts; designers can't iterate |
| LaTeX | Beautiful, but engineering-heavy; designers can't iterate |
| HTML + CSS → headless Chromium | Designer-friendly, browser-debuggable, supports modern CSS (grid, flexbox, page-break rules), trivial templating |
| Word → PDF | Brittle, server-side licensing trap |

We pick HTML + Chromium for the obvious reason: designers and engineers
can both work on it, with all the tooling each prefers.

---

## Pipeline

```
analysis JSON ──► template engine ──► rendered HTML ──► headless Chromium ──► PDF ──► S3
                                                                    │
                                                          stored under doc/content_hash/reports/
```

- **Template engine.** Go's `html/template` or a small templating
  layer that handles loops, conditionals, and citation interpolation.
- **Headless Chromium.** Either ChromeDP from a Go service or a sidecar
  microservice using Puppeteer/Playwright. We'll pick based on
  operational simplicity during MVP.
- **Storage.** Generated PDFs go to S3 under
  `docs/<content_hash>/reports/<report_version>/<variant>.pdf`. Generation
  is keyed on `(analysis_id, template_version, variant)` so the same
  combination never regenerates.

---

## Template living in the domain module

```
domains/health/report_template/
├── template.html          # entry point
├── style.css              # base + print styles
├── partials/
│   ├── header.html
│   ├── policy_identity.html
│   ├── members.html
│   ├── coverage.html
│   ├── waiting_periods.html
│   ├── exclusions.html
│   ├── advisor_assessment.html
│   └── footer.html
├── assets/
│   ├── logo.svg
│   └── icons/
└── variants/
    ├── customer.html      # the default — friendly tone, layman summaries
    ├── rm.html            # technical, dense; includes all sub-limits
    └── compliance.html    # full citation appendix, all footnote sources
```

Three variants from the same data set. Adding a "broker" variant is one
file.

---

## Anatomy of the customer variant

1. **Cover page.** Policy identity card: insurer logo, product name, UIN,
   policy period, members.
2. **At a glance.** A 4-tile grid: total SI, premium, copay/deductible
   highlights, key exclusions count. Each tile cites a field.
3. **What this policy covers** — the layman synthesizer output for
   `hospitalisation_and_core_benefits` and `value_added_and_wellness_benefits`,
   with cited values inline.
4. **What you'll wait for** — `waiting_periods`.
5. **What it doesn't cover** — `exclusions`. Permanent vs. temporary,
   visually separated.
6. **Money** — premium breakdown, copay, deductibles, sub-limits.
7. **Claims & network** — TPA, cashless, hospital count.
8. **Advisor's take** — `advisor_assessment` — score, strengths, gaps,
   recommended action.
9. **Citations appendix** — every claim, with page number and verbatim quote.

The appendix is what makes the PDF self-auditable.

---

## Inline citations in PDF

In the HTML template, citations are emitted as superscript markers next
to facts:

```html
<p>
  The room rent limit is <strong>{{ field.value "hospitalisation_and_core_benefits.room_rent_limit" }}</strong>.<sup>{{ cite "hospitalisation_and_core_benefits.room_rent_limit" }}</sup>
</p>
```

At template time, `cite` is replaced with a numeric footnote marker.
Each marker resolves to an entry in the citation appendix:

```
[12]  Section 4 — Room Rent Limit (page 14)
      "Reimbursement of room rent up to ₹3,000 per day or actual, whichever is lower."
```

The appendix entry uses the same source spans surfaced in the side-by-side
UI. Reproducibility: the PDF and the UI agree on every quote.

---

## CSS for print

- A4 default; letter as a config switch.
- `@page` rules for margins, headers, footers.
- `page-break-inside: avoid` on cards.
- A consistent type scale and color palette in `style.css`. Tailwind-like
  approach (utility classes) keeps the partials readable.
- Embedded fonts (Inter + a serif for headings) for predictable rendering.

---

## Generation API

```
POST /api/v1/analyses/{analysis_id}/reports
  body: { variant: "customer" | "rm" | "compliance" }
  ← 200 { report_id, status: "queued" | "ready", pdf_url }
```

Generation is async; the SPA polls (or SSE) for status. When ready, the
`pdf_url` is a presigned S3 URL. The report doc on Mongo:

```
{
  _id: <report_id>,
  analysis_id, content_hash, domain, variant,
  template_version: 4,
  generated_at,
  s3_key: "docs/<hash>/reports/v4/customer.pdf",
  bytes,
  generated_by, generated_for,    // user IDs
}
```

The same `(analysis_id, template_version, variant)` won't regenerate.
See [15 — Persistence & re-run guard](15-data-persistence-and-rerun.md).

---

## What "regeneration without re-billing" means

Templates change ("our designer added a 'previous insurer' tile").
Bumping the template version invalidates the PDF cache for that
variant but **does not** require re-running the analysis. The same
analysis JSON renders a new PDF. Cost: ≈ $0.0001 of Chromium CPU.

This is the property that the user explicitly asked for — and it's only
possible because the analysis is structured data, not opaque prose.

---

## Branding & white-labeling

Each customer-facing variant supports a small `branding.yaml`:

```yaml
logo_url: s3://branding/<tenant>/logo.svg
primary_color: "#1f4f8b"
contact_block: |
  ABC Insurance Advisors
  www.example.com
  +91 98xxx xxxxx
```

Loaded at template time, applied via CSS variables. New tenant onboarding
is one YAML file.

---

## Compliance variant — what's different

- Every field rendered, even the unfilled ones (shown as "Not stated").
- Per-field source quote inline (not just appended).
- Per-section confidence scores.
- The full `analysis.metadata` block: prompt-bundle hash, model id,
  schema version, skills applied.

This produces a long PDF (60+ pages for a complex policy) suitable for
internal audit but not customer-friendly. It's the receipts.

---

## Failure modes

- **Chromium crash:** retry once with a fresh instance; on second
  failure mark report `failed` with reason `renderer_crashed`. The
  analysis JSON is unaffected.
- **Missing required field:** the template renders "Not stated" with an
  inline marker indicating data quality. We never abort generation
  because of analysis gaps.
- **Citations broken:** if a field's citation can't be resolved (shouldn't
  happen — verifier catches it earlier), the inline marker shows "[?]"
  and the appendix lists the field as missing source. A loud, visible
  failure beats a silent one.

---

## What we explicitly don't do at MVP

- **No live edit-in-browser of the report.** The report is generated
  server-side from canonical data. Users can request a regeneration with
  a different variant; they can't hand-edit text. Live editing breaks
  reproducibility and citation guarantees.
- **No alternate output formats.** PDF only at MVP. DOCX is doable
  later via Pandoc; XLSX (raw fields export) is one schema-to-table
  pass.
- **No interactive PDFs (form fields, JS).** We render print-quality
  static PDFs.
- **No watermarking by default.** Tenant-specific watermarks are
  available via branding config; ungated by default.
