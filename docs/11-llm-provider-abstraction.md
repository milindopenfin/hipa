# 11 — LLM Provider Abstraction

> Models are config, not code. Any sub-agent's model can be switched between
> Anthropic, OpenAI, OpenRouter, or Ollama by flipping a value. Per-role
> classes (`cheap_router`, `extraction`, `extraction_strong`, `synthesis`,
> `advisory`) let us assign cheap models to dumb work and strong models to
> hard work without touching agent code.

Cross-refs: [04 — Multi-agent orchestration](04-multi-agent-orchestration.md),
[06 — Extraction pipeline](06-extraction-pipeline.md),
[12 — Agent tools](12-agent-tools.md),
[17 — Tech stack](17-tech-stack.md).

---

## The gateway

```
sub-agent ──► LLM gateway ──► provider (Anthropic | OpenAI | OpenRouter | Ollama | …)
                  │
                  ├── model resolution (role → model id)
                  ├── token counting / budget enforcement
                  ├── rate limiting (per provider, per account, per job)
                  ├── retries (network / 5xx / rate-limit)
                  ├── prompt + response logging (with hash)
                  ├── tool-use bridge (uniform tool-call shape across providers)
                  └── cost telemetry
```

The gateway is a Go service. Every sub-agent in the system goes through it.
No agent imports a provider SDK directly.

---

## Provider matrix (MVP)

| Provider | Why we want it | Notes |
|---|---|---|
| **Anthropic** | First-class Claude API; strong for synthesis and adversarial extraction | Default for `synthesis`, `extraction_strong`, `advisory` |
| **OpenAI** | GPT-4o-class as fallback / second opinion | Use for embeddings and as `extraction_strong` alternate |
| **OpenRouter** | Cheap access to many models; useful for OSS models w/o self-hosting | Use for `cheap_router`, `extraction` |
| **Ollama** | Local, free, air-gapped capability | Use for batch / dev / disaster-mode fallback |

Adding a provider is one Go file implementing a small interface — see
"Provider interface" below.

---

## Model classes (roles), not model IDs

Sub-agents declare a **role**, not a model. The gateway resolves the role
to a model at call time:

| Role | Intended use | Default model (Anthropic-preferred config) |
|---|---|---|
| `cheap_router` | Classification, decision routing | `claude-haiku-4-5-20251001` |
| `extraction` | Default sub-agent extraction | `claude-haiku-4-5-20251001` or `gpt-4o-mini` |
| `extraction_strong` | Retry tier; insurer-quirk extraction | `claude-sonnet-4-6` |
| `synthesis` | Layman narrative; Q&A; advisory | `claude-sonnet-4-6` |
| `advisory_deep` | Score & gaps; Q&A on adversarial prompts | `claude-opus-4-7` |
| `embedding` | Vector indexing | `text-embedding-3-large` or `bge-large` |
| `local_dev` | Air-gapped dev / batch | `qwen2.5:14b-instruct` (Ollama) |

A resolution table lives in `config/llm.yaml`:

```yaml
roles:
  cheap_router:
    provider: anthropic
    model: claude-haiku-4-5-20251001
    max_tokens: 800
    timeout_s: 20
  extraction:
    provider: anthropic
    model: claude-haiku-4-5-20251001
    max_tokens: 1500
    timeout_s: 30
  extraction_strong:
    provider: anthropic
    model: claude-sonnet-4-6
    max_tokens: 2000
    timeout_s: 45
  synthesis:
    provider: anthropic
    model: claude-sonnet-4-6
    max_tokens: 3000
    timeout_s: 60
  advisory_deep:
    provider: anthropic
    model: claude-opus-4-7
    max_tokens: 4000
    timeout_s: 120
  embedding:
    provider: openai
    model: text-embedding-3-large
    batch_size: 100
```

Environment-level overrides allow a deployment to swap one provider for
another without code changes:

```bash
# switch all "extraction" calls to OpenRouter / Kimi
LLM_ROLE_EXTRACTION_PROVIDER=openrouter
LLM_ROLE_EXTRACTION_MODEL=moonshotai/kimi-k2
```

---

## Why role-based, not model-based

v1 had per-role env vars (`MODEL_EXTRACTION`, `MODEL_SYNTHESIS`, etc.) but
inlined the role names in agent code. v2 makes the role part of the agent's
declared `agent.yaml` and lets the gateway resolve. The benefit:

- Swap providers per role in one place.
- A/B testing two models against the same role is one config flag.
- "Disaster mode" — point everything at Ollama — is one config flag.
- Cost projections become readable: tokens by role, multiplied by role's
  model price.

---

## Provider interface (Go)

```go
type Provider interface {
    Name() string
    Complete(ctx context.Context, req CompletionRequest) (CompletionResponse, error)
    Embed(ctx context.Context, req EmbedRequest) (EmbedResponse, error)
    SupportsTools() bool
    SupportsStreaming() bool
}

type CompletionRequest struct {
    Model       string
    System      string
    Messages    []Message
    Tools       []ToolSchema     // gateway-normalized
    MaxTokens   int
    Temperature float64
    Stop        []string
    JSONMode    bool
    Stream      bool
}

type CompletionResponse struct {
    Text         string
    ToolCalls    []ToolCall
    UsageIn      int
    UsageOut     int
    FinishReason string
    Raw          json.RawMessage   // provider-native; logged for audit
}
```

Four implementations at MVP:

- `anthropic.go` — wraps the Messages API.
- `openai.go` — wraps the Chat Completions / Responses API.
- `openrouter.go` — OpenAI-compatible passthrough.
- `ollama.go` — OpenAI-compatible passthrough at `http://localhost:11434/v1`.

Tool-call shape is normalized inside the gateway so sub-agents see one
schema regardless of provider. See [12 — Agent tools](12-agent-tools.md).

---

## Rate limiting & back-pressure

Each provider has:

- a **requests-per-minute** budget,
- a **tokens-per-minute** budget,
- and **per-key** sub-budgets (if multiple keys are pooled).

The gateway tracks these with a leaky bucket and blocks callers when
exceeded. A sub-agent's `await llm.complete(...)` may simply wait —
the orchestrator's view is unchanged. From there, three escape valves:

1. **Provider failover.** If one provider is saturated, the gateway can
   route to a configured alternate for the same role. Failover is
   *opt-in per role* (we don't blindly use OpenAI as a fallback for a
   role configured to Anthropic; the config has to declare it).
2. **Job-level concurrency cap.** Default 16. Prevents one huge job
   from starving others.
3. **Spot capacity for batch workloads.** Embeddings and master-library
   re-indexing route through a separate cheap queue that yields to
   interactive traffic.

---

## Cost telemetry

Every completion logs:

```json
{
  "request_id": "...",
  "analysis_id": "...",
  "sub_agent": "waiting_periods",
  "role": "extraction",
  "provider": "anthropic",
  "model": "claude-haiku-4-5-20251001",
  "tokens_in": 1234,
  "tokens_out": 567,
  "wall_ms": 8421,
  "cost_usd": 0.00123,
  "prompt_hash": "sha256:...",
  "tools_called": [],
  "finish_reason": "end_turn"
}
```

Two views downstream:

- **Per analysis.** Sum across sub-agents → "this analysis cost $0.23
  and used $0.05 in retries." Drives the re-run guard in [15](15-data-persistence-and-rerun.md).
- **Per provider × role × day.** Spend dashboard. Catches drift early.

---

## Auditability & prompt versioning

The gateway logs the **prompt hash** (sha256 of the fully-rendered
prompt) on every call, plus the prompt body in S3 cold storage (90 day
retention). This means:

- Re-running the same analysis with the same prompt hash produces a
  comparable result (token-level diff highlights model drift).
- Compliance reviewers can pull the exact text the model saw at any
  past analysis.
- A regression in extraction accuracy can be diffed against the prompt
  changelog.

Prompt body retention is bytes, not blobs in Mongo — Mongo only stores
the hash and S3 key.

---

## JSON-mode and structured output

Wherever possible the gateway uses provider-native JSON modes:

- Anthropic: enforce JSON via system prompt + stop sequences; (when
  available) use Anthropic's structured-output features.
- OpenAI: `response_format: { type: "json_schema", json_schema: ... }`.
- OpenRouter/Ollama: depends on the underlying model; the gateway falls
  back to "JSON only" prompting + post-parse validation when not.

The gateway exposes an `expected_schema` parameter on `Complete()`. When
set, it:

1. Hints the provider to JSON-mode-ish if supported.
2. Post-parses the response. If parse fails, retries once with a "your
   previous response was not valid JSON; here's the parse error" follow-up.
3. Returns a parse error after retry — caller decides what to do.

Sub-agents always pass an `expected_schema`. The orchestrator's verifier
is the second line of defense.

---

## Streaming

Sub-agent calls are non-streaming (JSON output, deterministic, full-then-
parse). Streaming is only used for:

- Q&A chat responses (token-by-token to the SPA).
- The synthesizer's prose (so the UI can show progress).

The gateway supports both modes through the same `Complete()` call by
flipping the `Stream` bool.

---

## Local / Ollama escape hatch

Ollama is a first-class provider, not a curiosity. Specifically:

- Dev environments can run end-to-end on Ollama with no API keys.
- A "disaster mode" switch (`LLM_DISASTER_MODE=ollama`) routes **every**
  role to Ollama. Quality drops; the system stays up.
- Batch operations on the master library (embedding, fingerprinting)
  can be configured to Ollama embeddings to save cost.

Ollama models tested at MVP: `qwen2.5:14b-instruct`, `llama3.1:8b-instruct`,
`nomic-embed-text` (embeddings).

---

## What we don't do

- **No fine-tuning at MVP.** We have no labeled dataset of Indian
  insurance extractions large enough to justify it. Prompt engineering
  and skills do more, faster.
- **No tool-use orchestration in the gateway.** Tool-call loops are
  handled by the *agent runtime* in [12 — Agent tools](12-agent-tools.md);
  the gateway just transports the tool-call objects.
- **No prompt template system in the gateway.** Templating happens in
  the sub-agent runtime, where it has type-safe access to the schema
  fragment and RAG context. The gateway sees fully-rendered prompts.

---

## Drop-in test plan

For a new provider:

1. Implement `Provider` interface.
2. Add a config entry.
3. Point a single role at it (e.g. `cheap_router`).
4. Run the regression suite (a corpus of ~50 analyzed documents).
5. Compare verifier flag rate against the baseline.
6. Promote to more roles if the flag rate is comparable or better.

No agent code changes; no schema changes; no domain code changes.
