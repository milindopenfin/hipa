# 16 — API & Auth

> A small, versioned, REST API. Better Auth for sessions, RBAC for module
> access, scoped tokens for third-party API consumers. v1's hand-rolled
> UUID-token system is one of the things we are explicitly replacing.

Cross-refs: [01 — Legacy observations](01-legacy-hipa-observations.md) (items 10, 11),
[02 — Architecture overview](02-architecture-overview.md),
[15 — Persistence & re-run guard](15-data-persistence-and-rerun.md),
[17 — Tech stack](17-tech-stack.md).

---

## API surface (v1 of the API; "v1" here ≠ the legacy product)

All under `/api/v1/`. JSON in / JSON out, except multipart for uploads.

### Documents

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/uploads` | Initiate upload (returns presigned URL + `pending_id`) |
| `POST` | `/uploads/{pending_id}/finalize` | Confirm S3 upload, returns `file_id` and detected domain |
| `POST` | `/uploads/{pending_id}/unlock` | Submit password if PDF was encrypted |
| `GET` | `/documents/{file_id}` | Document metadata + ingest tree URL |
| `GET` | `/documents/{file_id}/pages/{n}` | Page raster (presigned URL) |

### Analyses

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/analyses` | Start an analysis. Body: `{ file_id, domain?, options? }`. Returns `job_id`. |
| `GET` | `/jobs/{job_id}` | Job status |
| `GET` | `/jobs/{job_id}/stream` | SSE stream of state transitions + per-sub-agent progress |
| `GET` | `/analyses/{analysis_id}` | Full analysis JSON |
| `GET` | `/analyses/{analysis_id}/citations` | List citations (paginated) |
| `GET` | `/analyses/{analysis_id}/citations?field_path=...` | Citations for one field |

### Reports

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/analyses/{analysis_id}/reports` | Generate report. Body: `{ variant }`. Returns `report_id`. |
| `GET` | `/reports/{report_id}` | Status + presigned PDF URL when ready |

### Q&A

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/qa/sessions` | Start a session over one or more analyses |
| `POST` | `/qa/sessions/{sid}/messages` | Send a user message (chat) |
| `GET` | `/qa/sessions/{sid}/messages/stream` | SSE assistant tokens |
| `POST` | `/qa/sessions/{sid}/guided/answer` | Answer a guided-flow question |
| `GET` | `/qa/sessions/{sid}/payload` | Compiled quote payload |

### Feedback

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/feedback` | Submit per-section / per-field feedback |
| `POST` | `/skill_drafts` | Submit a skill draft (UI-built) |
| `POST` | `/prompt_reviews` | Submit a prompt-review ticket |

### Master library

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/library` | Browse master library; filters: insurer, uin, year, status |
| `GET` | `/library/{content_hash}` | Document metadata |
| `GET` | `/library/customers/{customer_id}` | One customer's policy slice |

### Admin

| Method | Path | Purpose |
|---|---|---|
| `GET/POST` | `/admin/skills` | List / approve / reject skills |
| `GET/POST` | `/admin/prompts` | Prompt versioning UI backend |
| `GET` | `/admin/feedback` | Triage queue |
| `GET` | `/admin/jobs` | Stuck-job operations |
| `GET/POST` | `/admin/roles` | RBAC management |

### Auth (Better Auth handlers)

Handled by Better Auth's routes; we mount them under `/api/v1/auth/`.

---

## Authentication: Better Auth

We use [Better Auth](https://www.better-auth.com) for session management.
What we get for free:

- Email/password + magic link + OAuth (Google, Microsoft) sign-in.
- Server-side session storage in Mongo (Better Auth has a Mongo adapter).
- HttpOnly, SameSite-strict session cookies.
- CSRF protection out of the box.
- 2FA hooks if/when we want them.
- A small, well-typed client for the SPA.

What we still own:

- The **role model** and module-access rules (see RBAC below).
- The **API token system** for third-party consumers (separate from
  user sessions).

We do not roll our own crypto. v1's UUID-as-bearer-token model
(`hipa/app_.py` `_TOKEN_CACHE`) is being deleted on purpose.

---

## Session shape

Better Auth issues a session cookie that maps to a `sessions` document in
Mongo:

```
sessions {
  _id, userId, expiresAt, ipAddress, userAgent,
  ...
}
```

Our middleware resolves the cookie → user → user's roles/permissions on
every request and attaches them to the request context.

---

## RBAC

Roles are defined data, not code:

```
roles {
  _id, name: "admin" | "rm" | "customer" | "compliance" | "<custom>",
  is_system: bool,
  permissions: ["analyses.read", "analyses.create", "library.read.master", "library.read.own", "reports.create", "admin.skills.approve", ...]
}

user_roles {
  user_id, role_id, scope: "global" | "tenant:<id>" | "customer:<id>"
}
```

Permission strings are flat-namespaced. The middleware checks per-route:

```go
@require("analyses.create")
func PostAnalysis(...) { ... }
```

A user with the `customer` role can read their own analyses
(`scope = customer:<their_id>`) but not others'. An `rm` can read and
analyze for any customer in their tenant. Admin permissions gate skill
approval, prompt edits, and library browsing across customers.

The legacy `AccessManager` (`hipa/db.py`) had a sketch of this: custom
roles + module access flags. v2 promotes it to first-class.

### Module access

Beyond fine-grained permissions, the SPA needs to know which top-level
modules a user can see (Analyses, Library, Q&A, Reports, Admin). The
`/me` endpoint returns:

```json
{
  "user": { ... },
  "permissions": ["analyses.read", ...],
  "modules_visible": ["analyses", "library", "qa", "reports"]
}
```

The SPA uses `modules_visible` for navigation visibility; permissions
gate individual actions. Defense-in-depth: the SPA hides what the user
can't do, the API enforces it regardless.

---

## API tokens for third-party consumers

Other systems need to consume HIPA programmatically (e.g. an
underwriter dashboard, a CRM ingesting analyses). They authenticate with
**API tokens**, not user cookies.

```
api_tokens {
  _id, owner_user_id, tenant_id,
  name: "Acme CRM ingestion",
  prefix: "hipa_at_",                   # first 8 chars; displayed in UI
  hash: <sha256 of full token>,         # we never store the plaintext
  scopes: ["analyses.read", "library.read.own"],
  expires_at, revoked_at,
  last_used_at, created_at
}
```

- Tokens are minted in the UI, shown once, never retrievable.
- Sent via `Authorization: Bearer hipa_at_...`.
- Scopes are a subset of the owner's permissions; cannot exceed them.
- Rate-limited per token.
- All actions log `actor: { kind: "api_token", token_id }` for audit.

---

## Idempotency

`POST` endpoints that create resources (`/analyses`, `/reports`,
`/qa/sessions`, `/uploads`) support an idempotency key:

```
Idempotency-Key: <client-generated-uuid>
```

Server stores `(idempotency_key, response)` for 24h; replays of the
same key return the same response. Prevents duplicate analyses when an
SPA retries on a flaky network.

---

## SSE for live job progress

```
GET /api/v1/jobs/{job_id}/stream
Accept: text/event-stream

event: state
data: { "state": "running", "completed_sub_agents": 4, "total": 12 }

event: state
data: { "state": "verifying", "completed_sub_agents": 12, "total": 12 }

event: state
data: { "state": "succeeded", "analysis_id": "..." }
```

The SPA opens an SSE connection right after `POST /analyses` and
animates progress. Falling back to polling `GET /jobs/{job_id}` works
identically; SSE is just nicer UX.

---

## Versioning

`/api/v1/` is the stable surface. Breaking changes increment to
`/api/v2/`. Within v1, additive changes are free. Field additions to
JSON responses don't break.

Schema-level versions (the extraction schema) are exposed via response
metadata:

```json
{ "analysis": { ... }, "metadata": { "schema_version": 1, "prompt_bundle_hash": "..." } }
```

Clients can ignore them or assert on them.

---

## CORS, CSRF, transport

- CORS: SPA origin allow-list per environment.
- CSRF: Better Auth handles cookies; for token-auth API consumers,
  CSRF is N/A.
- TLS only. HSTS in production.
- Bodies: max 50MB on `/uploads`; max 1MB on JSON endpoints.

---

## Rate limiting

Per-tier limits:

| Tier | Rate (rps) | Burst | Analyses/day |
|---|---|---|---|
| Free | 1 | 10 | 5 |
| Standard | 10 | 50 | 100 |
| Enterprise | 50 | 200 | 5000 (soft) |

Implemented as a token bucket per `(user_id OR token_id OR ip, route_class)`.
429s return `Retry-After`.

---

## Audit log

Every state-changing call writes a row:

```
audit_log {
  _id, ts, actor: { kind: "user"|"api_token", id }, ip, ua,
  route, method, params (subset), result_status,
  resource: { kind, id },
  diff (for updates)
}
```

We don't log request bodies wholesale — only structurally interesting
subsets. PII is redacted (we never log a member's full name; we log
record IDs that point to records the auditor can fetch with the right
scopes).

---

## Error shape

```json
{
  "error": {
    "code": "PASSWORD_REQUIRED",
    "message": "This PDF is password-protected. Submit the password to /uploads/{pending_id}/unlock.",
    "details": { "pending_id": "..." }
  }
}
```

Stable `code` strings. Human-readable `message`. Optional structured
`details`. The SPA branches on `code`, not on `message`.

---

## Health & readiness

| Path | Purpose |
|---|---|
| `GET /healthz` | Liveness: 200 if the process is up |
| `GET /readyz` | Readiness: 200 only if Mongo + S3 + Chroma + LLM gateway are reachable |
| `GET /version` | Build SHA, app version, schema version per domain |

---

## What's not exposed

- **Raw LLM access.** We don't proxy raw model APIs through HIPA.
- **Skill / prompt files.** Only readable via admin endpoints, never as
  public assets.
- **Other customers' data.** Even with `library.read.master`, the
  master-library responses strip customer attribution; cross-customer
  joins are admin-only and audited.
