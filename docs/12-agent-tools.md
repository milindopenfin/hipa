# 12 — Agent Tools

> Sub-agents shouldn't have to *infer* facts they could just look up. This doc
> defines the tool registry, per-agent permission scoping, and the safety
> rules around side-effectful tools (mostly: web search).

Cross-refs: [04 — Multi-agent orchestration](04-multi-agent-orchestration.md),
[08 — RAG & document store](08-rag-document-store.md),
[11 — LLM provider abstraction](11-llm-provider-abstraction.md),
[13 — Q&A & explanation](13-qa-and-explanation.md).

---

## What's a "tool"?

A typed function the model can call mid-completion. The gateway
intercepts the tool-call, executes the host function, returns the
result, and lets the model continue. Standard tool-use loop.

Tools are **registered** in the agent runtime by name. Sub-agents
declare in their `agent.yaml` which tools they may use:

```yaml
# domains/health/sub_agents/policy_identity/agent.yaml
name: policy_identity
model_class: extraction
tools_allowed:
  - system_time
  - master_library_lookup
```

Anything not listed is invisible to that sub-agent — even if it exists
globally. Permission is allow-list, not block-list.

---

## Tool catalog (MVP)

### Always-on, side-effect-free

| Tool | Purpose | Side effects |
|---|---|---|
| `system_time` | Get current ISO timestamp + IST date | none |
| `calc` | Evaluate simple arithmetic / date math safely | none |
| `pincode_lookup` | India pincode → city, state, region | none — local table |
| `node_text` | Fetch the full text of a tree node by `node_id` (when the RAG slice clipped a table) | none |
| `master_library_lookup` | Query the master library with insurer / UIN / product filters | none — read-only |
| `field_lookup` | Read another already-extracted field from the same analysis (for cross-section consistency checks) | none |

### Side-effectful, scoped

| Tool | Purpose | Side effects |
|---|---|---|
| `web_search` | Bounded internet search (whitelisted domains + IRDAI) | external HTTP |
| `fetch_url` | Fetch a specific URL (only from a whitelist) | external HTTP |

`web_search` and `fetch_url` are off by default. A sub-agent may use
them only if its `agent.yaml` lists them AND the orchestrator policy
allows the calling user's tier (`enterprise` users only, for example).
Both tools cap at N calls per analysis.

### Future / not in MVP

- `claim_settlement_ratio_lookup` — IRDAI scrape; we'll bake into a
  curated table first, then automate.
- `network_hospital_search` — by pincode + insurer.
- `irda_compliance_check` — query the IRDAI product registry.

---

## Why tools matter for this product

Several user-asked features collapse to one or two tool calls:

| Use case | Tool(s) |
|---|---|
| "Is this policy currently in force?" | `system_time` + a `field_lookup` for `policy_period_end` |
| "Find similar policies in our database" | `master_library_lookup` |
| "Network cashless hospitals near my pincode" | `pincode_lookup` + (future) `network_hospital_search` |
| "What's the latest IRDAI claim-settlement ratio for this insurer?" | (future) `claim_settlement_ratio_lookup` — falls back to `web_search` on irdai.gov.in |
| "What changed in this UIN's 2022 → 2023 wording?" | `master_library_lookup` filtered to same UIN |

Without tools, the model invents these answers. With tools, the model
*either* gets a real answer *or* refuses with "I don't have access to
that data" — which is acceptable.

---

## Tool schemas (examples)

### `system_time`

```json
{
  "name": "system_time",
  "description": "Returns the current UTC and IST timestamps and ISO date.",
  "input_schema": { "type": "object", "properties": {}, "additionalProperties": false }
}
```

Implementation: trivial. Returns `{utc, ist, iso_date, day_of_week}`.

### `master_library_lookup`

```json
{
  "name": "master_library_lookup",
  "description": "Search the archive of all policies. Filters are optional; at least one filter or query is required.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": { "type": "string", "description": "Free-text search over policy text." },
      "insurer": { "type": "string" },
      "uin": { "type": "string" },
      "product_name_contains": { "type": "string" },
      "financial_year": { "type": "string", "description": "e.g. 2022-2023" },
      "limit": { "type": "integer", "default": 5, "maximum": 20 }
    },
    "additionalProperties": false
  }
}
```

Returns a list of `{ uin, insurer, product_name, financial_year,
approval_date, doc_type, summary, content_hash }`.

### `web_search` (gated)

```json
{
  "name": "web_search",
  "description": "Search a restricted set of authoritative sources (IRDAI, official insurer websites). NOT a general web search.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": { "type": "string" },
      "site": { "type": "string", "enum": ["irdai.gov.in", "policyholder.gov.in", "<insurer-site-list>"] }
    },
    "required": ["query"]
  }
}
```

Returns top-3 hits with snippet + URL. Cached for 24h on
`(query, site)`. Domain whitelist is configurable; calls to non-whitelist
URLs are rejected at the host function level — the model can't even ask
for them.

---

## Tool-use loop

```
gateway ──► model: prompt + tool schemas
model    ──► gateway: tool_call("master_library_lookup", { uin: "...", limit: 3 })
gateway  ──► tool runtime: execute
tool     ──► gateway: result JSON
gateway  ──► model: tool_result(...)
model    ──► gateway: final answer (or another tool call)
```

Loop cap: 6 tool calls per sub-agent call. After 6 the model is told
"no more tool calls; finalize your output." Sub-agents that hit the cap
emit a flag `tool_call_limit` which the verifier treats as a "needs
review" signal.

Tools that fail (timeout, host error) return `{ error: "..." }` to the
model rather than throwing. The model decides whether to retry, switch
strategy, or surface "couldn't find this" in its output.

---

## Permission scopes

| Tier | Tools allowed |
|---|---|
| Extraction sub-agents | side-effect-free only |
| Synthesizer | side-effect-free only |
| Q&A agent | side-effect-free + `web_search` if user tier permits |
| Advisory sub-agent | side-effect-free + `master_library_lookup` |
| Customer-facing Q&A | side-effect-free; no `web_search` (latency + cost) |
| RM-facing Q&A | side-effect-free + `web_search` + `master_library_lookup` |

The orchestrator validates each sub-agent's declared `tools_allowed`
against the agent type's tier policy at load time. A misconfigured
agent fails at startup, not at runtime.

---

## Logging & cost

Every tool call logs:

```
{
  request_id, analysis_id, sub_agent, tool_name,
  args, result_summary, wall_ms, cost_usd, error?
}
```

`web_search` and `fetch_url` are the only tools with non-trivial cost
(API + bandwidth). Their cost is added to the analysis-level cost
telemetry. See [11](11-llm-provider-abstraction.md).

---

## Safety rules

1. **No filesystem access.** Tools may read from S3 / Mongo / Chroma
   via well-defined query interfaces; arbitrary file access is not a tool.
2. **No code execution.** No `python_exec`, no `shell`, no anything that
   would let the model run arbitrary code.
3. **No PII exfiltration.** `web_search` and `fetch_url` strip any input
   that looks like PII (member name, DOB, address) before sending — and
   refuse if the query *requires* PII.
4. **No claim/payment endpoints.** Tools never write to insurer or
   payment systems. Period. The system produces *advice and payloads*,
   not transactions.

---

## Why a tool registry instead of inline functions

- Each tool is implemented once, tested once, used by any sub-agent.
- Adding `master_library_lookup` cost zero changes to existing sub-agents
  — they pick it up by listing it in their `agent.yaml`.
- The gateway can enforce uniform retry, timeout, cost, and logging
  policy across tools.
- New domain modules can ship domain-specific tools under
  `domains/<name>/tools/` without polluting the global registry; the
  orchestrator merges domain tools into the registry at module load.

---

## Q&A-specific notes

The Q&A agent (see [13](13-qa-and-explanation.md)) is the heaviest tool
user. It's not extracting fields — it's answering ad-hoc questions over a
completed analysis and (optionally) the master library. Expect 2–4 tool
calls per Q&A turn (`field_lookup`, `master_library_lookup`,
occasionally `web_search`). The Q&A agent's prompt explicitly tells the
model: "Prefer answering from cached extraction; consult tools only when
the answer requires data outside the analysis."
