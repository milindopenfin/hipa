# 13 — Q&A & Layman Explanation

> A completed analysis is a structured JSON document. Real users — customers
> and RMs — don't read JSON. They ask questions, and the system answers in
> plain language, grounded in the analysis and the underlying policy.
>
> One interface, two voices: customer-friendly ("is dental covered?"),
> RM-friendly ("show me the room rent sub-limit and any proportionate-deduction
> trigger").

Cross-refs: [06 — Extraction pipeline](06-extraction-pipeline.md),
[08 — RAG & document store](08-rag-document-store.md),
[09 — Citation & grounding](09-citation-and-grounding.md),
[12 — Agent tools](12-agent-tools.md).

---

## Two flows, one agent

### Flow A — Guided Q&A (capture customer profile)

The post-analysis question bank, ported from v1's `qa_agent.py` but
upgraded:

- Questions live in `domains/<domain>/question_bank.json` — data, not code.
- The agent may **inject follow-ups dynamically** based on the analyzed
  policy (e.g. "this is a Family Floater for 4 — how old is each
  member?" is asked only when the analysis says `plan_type` =
  `Family Floater` and member ages aren't yet known).
- Output: a customer profile that's compiled into a quote payload at
  the end. (See `build_quote_payload()` in v1's `qa_agent.py` for the
  shape; we keep it.)

### Flow B — Free-form chat (over completed analysis)

The customer / RM types a question. The agent answers, cites, and
remembers the conversation in a chat session.

Both flows are served by the **same agent runtime**, with different
seed prompts and tool sets.

---

## The Q&A agent

```
question ──► Q&A agent ──┬──► answer (with citations)
                          │
                          ├── tool: field_lookup        (read analysis fields)
                          ├── tool: master_library_lookup (cross-policy queries)
                          ├── tool: pincode_lookup
                          ├── tool: system_time
                          └── tool: web_search (RM tier only, rate-capped)
```

Prompt skeleton (per turn):

```
You are a health insurance policy expert. The user is a {customer | relationship_manager}.
The user has uploaded the policy summarized below.
Answer the user's question grounded ONLY in:
 1. the analysis fields available via field_lookup,
 2. the original document via node_text,
 3. the tools listed.
If you don't know, say so.
Every claim in your answer must be followed by an inline citation marker
referring to a field path (preferred) or a document node_id.

Policy summary (compact): {compressed analysis JSON}
Conversation history: {prior turns}
User question: {message}
```

The agent must produce JSON like:

```json
{
  "answer_markdown": "Yes — dental treatment is covered when hospitalisation is required. The policy notes ${field:exclusions.dental_optical_exclusions}.",
  "citations": [
    { "field_path": "exclusions.dental_optical_exclusions" }
  ],
  "follow_ups": [
    "Want me to check the day-care list for dental procedures?"
  ]
}
```

The renderer expands `${field:...}` to the actual value and attaches the
"view source" affordance, identical to the static analysis UI. See
[09 — Citation & grounding](09-citation-and-grounding.md).

---

## Customer voice vs RM voice

The same prompt skeleton with a one-line role switch:

- **Customer**: "Use simple language. Avoid jargon. Translate legal terms
  into plain English. Show ₹ amounts in lakhs where appropriate."
- **RM**: "Use technical insurance vocabulary. Be precise about clauses,
  sub-limits, and clause numbers. Prefer table-format answers for
  numerical comparisons."

Both flows have access to the same tools; the difference is presentation,
not data. We never hide information from the customer — we explain it.

---

## Layman explanation (passive)

Beyond the Q&A flow, the system produces a one-line layman summary for
every section of the analysis. These are pre-computed at synthesis time
(see [06 — Phase D](06-extraction-pipeline.md)) and shown as the
collapsed view of each section:

```
▼ Waiting periods
   "You'll wait 30 days before any non-accidental claim, and 36 months
    before pre-existing conditions are covered."

▼ Exclusions
   "11 permanent exclusions, including cosmetic surgery and self-inflicted
    injury. 3 conditions covered after waiting periods."
```

These summaries are also fully cited — they reference field paths and
inherit citations from them.

---

## Grounding rules

The Q&A agent is bound by the same grounding contract as extraction:

1. **Prefer `field_lookup`.** If the answer is already in the analysis,
   read it from there. No re-extraction.
2. **Fall back to `node_text`.** If the answer requires policy detail
   that wasn't extracted, fetch the relevant tree node and reason over it.
3. **Cross-policy questions go through `master_library_lookup`.** No
   guessing about other policies from memory.
4. **Refuse cleanly.** If neither analysis fields, document tree, nor
   tools have the answer, the agent says "I don't have that information
   in this policy."

The agent prompt explicitly forbids reasoning from general training-data
knowledge about policies. If a customer asks "what's typical for room
rent?" the agent must call `master_library_lookup` for similar policies,
not invent a number.

---

## Cross-policy questions

A customer has uploaded three policies. They ask: "Which of my policies
covers home care, and to what limit?"

```
agent → field_lookup(analysis_id=POLICY_A, field=value_added.home_care_treatment)
agent → field_lookup(analysis_id=POLICY_B, field=value_added.home_care_treatment)
agent → field_lookup(analysis_id=POLICY_C, field=value_added.home_care_treatment)
agent → composes answer with three citations
```

The orchestrator's Q&A API accepts a `policy_ids` array; the agent
iterates without re-loading documents.

---

## Q&A session shape (Mongo)

```
qa_sessions {
  _id: "<session_id>",
  user_id, customer_id,
  voice: "customer" | "rm",
  policy_ids: [...],              // one or many
  guided_question_bank: [...],    // for Flow A
  guided_answers: { ... },
  guided_status: "in_progress" | "completed",
  chat_turns: [
    {
      ts, role: "user" | "assistant",
      message,
      citations: [...],
      tool_calls: [...],
      thumbs: "up" | "down" | null,
      comment: "..."
    }, ...
  ],
  created_at, last_activity
}
```

Per-turn thumbs and comments feed the feedback router. See
[10 — Skills & HITL feedback](10-skills-and-hitl-feedback.md).

---

## Question bank (data-driven)

```
// domains/health/question_bank.json (excerpt)
[
  { "id": "coverage_type", "category": "Coverage",
    "text": "Are you looking at Individual cover or Family Floater?",
    "type": "choice",
    "choices": ["Individual", "Family Floater", "Both"],
    "ask_if": { "policy.plan_type_in": ["Individual", "Family Floater"] }
  },
  { "id": "member_ages", "category": "Members",
    "text": "Please tell me the age of each member to be covered.",
    "type": "list_of_int",
    "ask_if": { "policy.plan_type_eq": "Family Floater" },
    "follow_up_for_each_item": {
      "id": "ped_per_member",
      "text": "Does member {age}-year-old have any pre-existing conditions?",
      "type": "string"
    }
  },
  { "id": "desired_si", "category": "Coverage",
    "text": "What sum insured are you considering?",
    "type": "currency_inr"
  }
  // ...
]
```

`ask_if` lets questions condition on the analyzed policy.
`follow_up_for_each_item` lets list questions spawn per-item follow-ups.

A separate per-domain `question_bank_validator` (small LLM call, or
deterministic) checks free-text answers for sanity before recording them.
v1's `validate_answer_with_claude()` was a no-op (regex only); v2 makes
it real, but only for questions that need it.

---

## Quote payload

Once the guided Q&A is complete, the system compiles a quote-ready
payload:

```
{
  "customer_profile": { age, pincode, occupation, health, ... },
  "preferences": { coverage_type, sum_insured, riders, network, term, budget, copay_ok, deductible_ok },
  "analyzed_policies": [analysis_id, ...]
}
```

This is the same shape v1's `build_quote_payload()` produces, slightly
extended with `analyzed_policies` so a downstream quote engine can
compare new quotes against the customer's existing covers.

In MVP we expose this as JSON via `GET /api/v1/qa/sessions/{id}/payload`
for the RM team to consume. Wiring it to live insurer-API quote endpoints
is post-MVP. See [18 — Roadmap](18-roadmap.md).

---

## Customer/RM voice toggle and access control

Customers see the customer voice. RMs see a "Show technical detail"
toggle in the chat UI that switches the voice. Both voices use the same
underlying agent and citations; only the prompt and a couple of
tool-permissions change.

---

## Why a single agent, not two

v1 had a `qa_agent.py` (guided flow) and a separate `/api/chat`
(free-form). Splitting led to:

- Two prompt sources of truth.
- Two places to fix a bug.
- No shared chat memory between flows.

v2 unifies them. The "guided" flow is just the same agent driven by a
preset script of questions; the "free-form" flow is the same agent
driven by user messages. Switching between them mid-session is
seamless (a customer can interrupt the guided flow to ask an ad-hoc
question and resume).

---

## Conversational memory

Each chat session keeps the last N turns in context (windowed). For
longer conversations the agent has a `summarize_session` tool that
compresses old turns into a recap embedded at the top of the prompt.
We do **not** persist a generic "user memory" across unrelated
sessions in MVP — v1 had one (`memory.py`, 579 lines) and the value/cost
ratio is unclear. We'll add it if usage patterns demand it.

Per-customer state we *do* keep across sessions:

- Their uploaded policy library (read-only context).
- Their guided-Q&A answers (the customer profile).
- Their thumbs-up/down history (for prompt-review prioritization).

That's enough to give continuity without the privacy cost of free-form
recall.
