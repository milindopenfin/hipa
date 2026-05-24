# 05 — Document Ingestion

> Turn an arbitrary policy file into a clean, addressable, hierarchical
> markdown tree with stable node IDs and character offsets. Every downstream
> agent operates on this tree, not on raw text.

Cross-refs: [01 — Legacy observations](01-legacy-hipa-observations.md)
(items 1, 2 — silent on scans, no document structure),
[06 — Extraction pipeline](06-extraction-pipeline.md),
[08 — RAG & document store](08-rag-document-store.md),
[09 — Citation & grounding](09-citation-and-grounding.md).

---

## Why a markdown tree, not flat text

A 50-page health-insurance policy is structurally regular:

```
Policy Schedule
├── Policyholder Information
├── Member Table
├── Sum Insured & Premium
└── Endorsements

Policy Wording (the long, mostly-boilerplate part)
├── Definitions
├── Covered Benefits
│   ├── Hospitalisation
│   ├── Day Care
│   └── ...
├── Waiting Periods
├── Exclusions (the section that actually varies by product)
├── Claims Process
└── Annexures

Annexures / Riders
├── Critical Illness Add-on
└── ...
```

v1 throws all of this into a flat string and asks BM25 to find the right
bits. That works ~70% of the time. The 30% miss rate is where v1's
inaccuracy lives.

v2 preserves the structure and exposes addressable subtrees:

```json
{
  "node_id": "policy_wording/exclusions/permanent_exclusions",
  "depth": 3,
  "heading": "Permanent Exclusions",
  "page_start": 22,
  "page_end": 24,
  "char_start": 41028,
  "char_end": 44915,
  "children": [...],
  "tables": [...]
}
```

A sub-agent for `exclusions` can request "give me the subtree
`policy_wording/exclusions/**`" and get exactly that — no irrelevant
chunks, no missing context.

---

## Pipeline

```
upload ──► format detect ──► unlock? ──► extractor ──► tree builder ──► persist
                                          │
                                          ▼ if scan-like
                                         OCR ──┐
                                                ▼ merged back into extractor output
```

### 1. Format detect

By magic bytes, not extension. We trust filenames the same way we trust
user input.

Supported v2 MVP: PDF, DOCX, XLSX, CSV. Images (JPG/PNG/TIFF) go through
the OCR path directly.

### 2. Password unlock

PDFs are checked with `pikepdf` (or Go equivalent — `pdfcpu`). If encrypted:

- The API responds `409 PASSWORD_REQUIRED` with a `pending_id`.
- The SPA prompts the user for the password.
- The user posts the password to `/api/v1/upload/unlock`; the server
  decrypts in-memory, the cleartext byte stream is then ingested.
- Passwords are **never** persisted. We log only the fact that an unlock
  succeeded.

### 3. Text extraction

Two parallel paths:

- **Native text path.** Layout-aware extractor (Go binding to `pdfium`, or
  shell to `pdftotext` / `marker` / `markitdown`-equivalent depending on
  what produces the cleanest markdown for Indian insurance policies —
  benchmarked during MVP). Output: markdown text with tables in GFM
  format, plus a per-page char-offset map.

- **OCR path.** Triggered when any page has fewer than
  `MIN_CHARS_PER_PAGE` (e.g. 100) of native text. Use Tesseract for
  English; consider PaddleOCR if we see meaningful volume of
  English-Hindi mixed documents. OCR output replaces the native text for
  those pages.

The decision is per-page, not per-document — many real policies are
mostly native-text with a scanned signature page or a scanned health-
declaration annexure.

### 4. Tree builder

Take the markdown stream and parse into the tree shape above. Heuristics:

- Markdown headings (`#`, `##`, ...) → tree levels.
- All-caps lines followed by paragraph text → treat as heading.
- Numbered section markers (`4.2`, `4.2.1`) → assemble into a path.
- Tables (GFM `|...|`) → captured as `tables[]` on their nearest enclosing
  node, not as text-content children. Each row + cell retains char offsets.

Every node gets a **stable `node_id`** derived from the path, slugified.
Stability matters because skills (see [10](10-skills-and-hitl-feedback.md))
can target specific node_ids for an insurer.

### 5. Persistence

The ingested artifact is stored as:

```
s3://<bucket>/docs/<content_hash>/
   ├── original.pdf            # the raw upload (or .docx, etc.)
   ├── ocr_pages.json          # which pages OCR-ed and the OCR engine version
   ├── tree.json               # the hierarchical tree
   ├── full_markdown.md        # the linearized markdown (for human review)
   └── pages/                  # per-page rasters for the citation viewer
       ├── page-0001.webp
       ├── page-0002.webp
       └── ...
```

- The content hash is over the **original bytes** of the upload, so a
  later re-upload of the same file deduplicates instantly. See
  [15 — Persistence & re-run guard](15-data-persistence-and-rerun.md).
- Per-page rasters (rendered at a fixed DPI) are produced once at ingest
  time so the citation viewer never has to re-rasterize. See
  [09 — Citation & grounding](09-citation-and-grounding.md).

---

## Markdown tree — example node

```json
{
  "node_id": "policy_wording/waiting_periods/specific_diseases",
  "path": ["Policy Wording", "Waiting Periods", "Specific Diseases"],
  "depth": 3,
  "heading": "Specific Diseases Waiting Period",
  "section_number": "4.3",
  "page_start": 18,
  "page_end": 19,
  "char_start": 33102,
  "char_end": 34418,
  "text": "The Company shall not be liable to make any payment...",
  "tables": [],
  "children": [],
  "fingerprint": "sha1:e4f1..."     // stable hash of normalized content
}
```

The `fingerprint` is computed over normalized (whitespace-collapsed,
case-folded) text so that the same wording across two policies produces
the same fingerprint — useful for the cross-document master library to
detect boilerplate. See [08](08-rag-document-store.md).

---

## OCR specifics

- **Trigger:** per-page native-text char count below threshold,
  or PDF page reports zero embedded fonts.
- **Engine:** Tesseract 5 with `eng` traineddata; fall back to PaddleOCR
  for poor scans (we'll know which during MVP benchmarking).
- **Preprocessing:** binarize → deskew → upscale to 300 DPI if lower.
- **Post-processing:** strip ligature artifacts; join hyphenated
  line-breaks; recover table structure where possible (Tesseract layout
  mode + Tabula-like row inference). If we cannot recover a table, leave
  it as fenced raw text and let the extractor know via a `tables_lost`
  flag on the node so its sub-agents can ask for retries with a wider
  context.
- **Audit:** `ocr_pages.json` records which pages OCR-ed, the engine
  version, the confidence histogram, and the preprocessing applied — so
  later re-ingest with a better OCR engine is deterministic and traceable.

OCR cost is non-trivial; we cache OCR output by `(page_image_hash,
ocr_engine_version)`.

---

## Password-protected files — UX contract

```
POST /api/v1/upload
  multipart: file=<encrypted.pdf>
  ← 409 { "pending_id": "...", "reason": "password_required" }

POST /api/v1/upload/unlock
  body: { "pending_id": "...", "password": "..." }
  ← 200 { "file_id": "...", "detected_domain": "health" }
```

The pending bytes live on the API node's tmpfs for at most 5 minutes; an
unlock failure or a timeout discards them. Passwords are scrubbed from
logs.

---

## Limits & failure modes

- **Max file size:** 50 MB. We've seen real policies push 20 MB; 50 MB
  is the soft cap for the MVP.
- **Max pages:** 200. Above this, mark as `oversized` and route to a
  manual queue.
- **All pages OCR-only and OCR confidence below 0.6:** mark as
  `low_quality_scan`. Ingest still proceeds but the analysis carries a
  banner warning users about confidence.
- **No detectable headings:** the tree degrades to a single root node with
  page-bounded children. Extraction still works but cross-document
  retrieval and section-scoped RAG are less effective.

---

## What this enables downstream

- Sub-agents request **subtree-scoped retrieval**, not full-doc retrieval.
  See [06 — Extraction pipeline](06-extraction-pipeline.md).
- Citations point at `(node_id, char_offset, char_offset)`, which the UI
  can resolve to a page rectangle for highlighting. See
  [09 — Citation & grounding](09-citation-and-grounding.md).
- The master library indexes nodes individually, so cross-document
  searches can find "all Star Health policies' Section 4.3 wording." See
  [08 — RAG & document store](08-rag-document-store.md).
- The same tree powers OCR-aware reports and the Q&A agent. See
  [13 — Q&A & explanation](13-qa-and-explanation.md),
  [14 — Report generation](14-report-generation.md).
