# Security-First MCP CRUD Architecture
 
**Design philosophy: the agent is an untrusted, potentially adversarial component. Every decision that matters (identity, authorization, approval, execution) is made by the server, never by the agent. The system fails closed by default.**
 
---
 
## 1. Architecture Diagram (logical view)
 
```
┌───────────────────────────────────────────────────────────────────────────┐
│ CLIENT / UI LAYER (OS-specific)                                            │
│  macOS app | Windows app | Linux app | Web client | Other MCP clients      │
│  - Renders approval prompts                                                │
│  - Fires OS notification (desktop/sound/browser/terminal)                  │
│  - Holds the user's authenticated session (never the agent)                │
└───────────────────────────────────────────────────────────────────────────┘
                 │ (user-approved actions only travel this direction)
                 ▼
┌───────────────────────────────────────────────────────────────────────────┐
│ MCP TRANSPORT LAYER (stdio / HTTP+SSE / WebSocket)                         │
│  - Carries JSON-RPC MCP messages only                                     │
│  - No business logic, no auth decisions                                   │
└───────────────────────────────────────────────────────────────────────────┘
                 │
                 ▼
┌───────────────────────────────────────────────────────────────────────────┐
│ MCP SERVER — SECURITY PIPELINE (this is the only door to data)            │
│                                                                             │
│  1. AuthN Middleware        →  resolves trusted identity from session      │
│  2. Request Validation      →  schema, type, size, sanity checks           │
│  3. Correlation/Request ID  →  assigned server-side, never agent-supplied  │
│  4. Policy/Authorization    →  RBAC/ABAC + ownership check                 │
│  5. Action Proposal Engine  →  builds the EXACT diff to be approved        │
│  6. Approval Service        →  issues single-use, scoped approval token    │
│  7. Human Approval Gate     →  blocks until real user approves/denies      │
│  8. Re-validation Layer     →  re-checks auth + resource version pre-write │
│  9. Transaction/Concurrency →  optimistic locking, DB transaction          │
│ 10. Repository Layer        →  parameterized queries, least privilege      │
│ 11. Audit Logger            →  append-only, tamper-evident audit trail     │
│ 12. Security Monitoring     →  anomaly/rate alerts, replay detection       │
└───────────────────────────────────────────────────────────────────────────┘
                 │
                 ▼
┌───────────────────────────────────────────────────────────────────────────┐
│ DATABASE (least-privilege role, no agent connection ever exists)           │
│  users | resources | ownership | approvals | audit_events (append-only)   │
└───────────────────────────────────────────────────────────────────────────┘
 
              ▲
              │ (agent only ever talks to the MCP server's proposal tools)
┌───────────────────────────────────────────────────────────────────────────┐
│ AI AGENT (UNTRUSTED)                                                       │
│  - Can only call request_action-style tools                               │
│  - Cannot call repository/db tools                                        │
│  - Cannot supply user_id, approval, or "approved=true"                    │
│  - Every claim it makes ("user approved this") is ignored by the server   │
└───────────────────────────────────────────────────────────────────────────┘
```
 
**Key invariant expressed in the diagram:** there is exactly one path from "intent" to "database write," and it passes through authentication → authorization → proposal → human approval → re-validation → transaction. There is no side channel.
 
---
 
## 2. Trust Boundaries
 
| Boundary | Trusted side | Untrusted side | Enforcement |
|---|---|---|---|
| B1: Agent ↔ MCP Server | MCP Server | AI Agent | Server treats all tool-call arguments as attacker-controlled input |
| B2: Client UI ↔ MCP Server | MCP Server | Client UI process | Server still re-validates even though the UI is "closer" to the user — a compromised or buggy client must not gain trust |
| B3: MCP Server ↔ Auth Provider | Auth Provider (OIDC/SSO/session store) | Everything else | Identity is derived only from a verified token/session, never from message fields |
| B4: Approval Service ↔ Human | Human, via out-of-band UI/notification channel | Agent | Approval tokens are minted only by the Approval Service after a real human interaction; the agent cannot mint or forge one |
| B5: Application ↔ Database | Database with least-privilege role | Application code (including any bug in it) | DB-level constraints, FKs, and a separate audit-writer role that the app role cannot ALTER/DELETE from |
| B6: Application ↔ Audit Store | Audit Store (append-only) | Application runtime | Application has INSERT-only grant on audit tables; deletion/rotation is a separate privileged, logged, offline process |
 
The core rule: **nothing that crosses B1 or B2 is ever trusted for identity, authorization, or approval — those three facts are only ever established on the trusted side of B3 and B4.**
 
---
 
## 3. Component Responsibilities
 
- **MCP Transport** — moves JSON-RPC messages. Knows nothing about users, records, or SQL. Enforces message-size limits and protocol correctness only.
- **Authentication Service** — validates session/token issued by a real IdP (OAuth2/OIDC, platform SSO, signed session cookie, etc.). Produces a `Principal { user_id, session_id, auth_level }`. This is the *only* source of `user_id` used anywhere downstream — tool arguments named `user_id` are ignored/rejected if present.
- **Identity Context** — a per-request, immutable object built once by AuthN and passed down; never rebuilt from agent input.
- **Request Validator** — schema validation (types, ranges, enum values, string lengths), rejects malformed or oversized payloads before they reach business logic.
- **Policy Engine (AuthZ)** — evaluates RBAC/ABAC rules: "can this Principal perform this operation on this resource class at all." Independent of ownership.
- **Ownership Service** — loads the authoritative record from the repository and compares `record.owner_id` to `Principal.user_id` (or checks a delegation/ACL table). This check is *re-run*, never cached across the approval gap.
- **Action/Proposal Engine** — given a validated intent, produces a canonical `ActionProposal`: exact operation, resource id, resource version, field-level diff (current → proposed), destructive flag, reversibility flag.
- **Approval Service** — persists the `ActionProposal` as a row with a unique `approval_id`, TTL, and cryptographically random token; pushes it to the client for human review; accepts exactly one `approve`/`deny` decision tied to that `approval_id`.
- **Human Approval Gate** — a blocking (or async-callback) step in the pipeline; the pipeline literally cannot proceed to execution without a decision record for this exact `approval_id`.
- **Service Layer** — orchestrates the steps above in order; contains no direct SQL.
- **Repository/Data Access Layer** — the only code allowed to hold a DB connection; exclusively parameterized queries; enforces optimistic-lock version checks in the `WHERE` clause of writes.
- **Transaction Manager** — wraps read-verify-write in a single DB transaction at an appropriate isolation level (see §14).
- **Concurrency Control** — resource `version`/`updated_at` column checked with `WHERE id = ? AND version = ?`; zero rows affected ⇒ conflict ⇒ abort.
- **Audit Logger** — writes an immutable audit event synchronously in the same transaction (or via reliable outbox) for every attempted and completed mutation, success or failure.
- **Security Monitoring** — consumes audit stream/metrics for anomaly detection: repeated authorization failures, approval-forgery attempts, unusual bulk-operation patterns, replay attempts.
---
 
## 4. Why naive tool designs are unsafe
 
```
edit_record(id, new_values, approved=true)
delete_record(id, confirm=true)
```
 
These are unsafe because:
 
1. **The approval flag is agent-controlled.** Anything the agent can set, the agent can set incorrectly or dishonestly — whether from a bug, a prompt injection, or an adversarial instruction telling it to "just pass approved=true."
2. **No binding to a specific reviewed action.** Even if a human once said "yes," `approved=true` doesn't say *yes to what*. The agent could change `id` or `new_values` after the human looked at something else entirely.
3. **Replayable.** The same call, or the same flag, can be issued again for a different record or a different field set.
4. **No record of what was actually shown to the human.** You cannot prove, after the fact, that the human saw the real diff.
5. **Encourages the agent to "own" the decision**, which is exactly backwards for a zero-trust design — the decision must be owned by server-held state (the approval record), not by a value passed through the same channel you don't trust.
---
 
## 5. MCP Tool / API Design (proposal → approval → execution)
 
Tools exposed to the agent (this is the **entire** surface — no other data tools exist):
 
```
get_data(resource_type, filters, pagination)          // read-access policy applies, no approval needed
request_action(operation, resource_type, resource_id, proposed_changes, reason?)
                                                        // CREATE / EDIT / DELETE go through this ONLY
check_approval_status(approval_id)                     // agent may poll, cannot influence outcome
```
 
Notice what is **not** exposed: no `edit_record`, no `delete_record`, no `run_sql`, no `db_query`, no `approved` parameter anywhere, no `bulk_*` tool by default (see §8).
 
`request_action` never mutates data. It:
1. Authenticates (via transport-level session, not arguments).
2. Validates the shape of `proposed_changes`.
3. Runs AuthZ + ownership.
4. Builds an `ActionProposal` and persists it with a fresh `approval_id`.
5. Triggers the client-side approval UI/notification.
6. Returns `{ approval_id, status: "pending", expires_at }` to the agent — nothing more.
Execution happens **only** from the server-side approval callback (the human's "Approve" click hitting the server directly, or a signed webhook from the approval UI) — **never** from another agent tool call. Even if the agent calls `check_approval_status` and sees `approved`, calling that tool does not trigger execution; execution is triggered by the approval event itself, server-side.
 
---
 
## 6. Approval State Machine
 
```
                 ┌─────────────┐
                 │   CREATED   │  (request_action validated, proposal stored)
                 └──────┬──────┘
                        │ notify user
                        ▼
                 ┌─────────────┐
        ┌────────┤   PENDING   ├────────┐
        │        └──────┬──────┘        │
   expires_at        user           user denies
     reached        approves            │
        │              │                │
        ▼              ▼                ▼
  ┌───────────┐  ┌─────────────┐  ┌──────────┐
  │  EXPIRED  │  │  APPROVED   │  │  DENIED  │
  └───────────┘  └──────┬──────┘  └──────────┘
                        │
              re-validate auth + version
                        │
              ┌─────────┴─────────┐
           match ok           conflict/fail
              │                    │
              ▼                    ▼
      ┌───────────────┐     ┌──────────────┐
      │   EXECUTING   │     │   REJECTED   │
      └───────┬───────┘     │ (stale/auth) │
              │              └──────────────┘
     ┌────────┴────────┐
   success            error
     │                  │
     ▼                  ▼
┌──────────┐     ┌──────────────┐
│ EXECUTED │     │   FAILED     │
└──────────┘     └──────────────┘
```
 
Rules enforced by the state machine, not by convention:
- Only `PENDING → APPROVED/DENIED/EXPIRED` transitions are valid; any other transition attempt is rejected and logged.
- `APPROVED` is a one-time-use state: the moment execution is attempted (success or failure), the approval is marked `CONSUMED` and can never be reused (blocks "replay an old approval").
- Every state row stores a hash of the exact `ActionProposal` it was created for. If the operation the caller now wants to execute doesn't hash-match the approved proposal, the transition is rejected (blocks "modify the operation after approval" and "approval substitution").
- `approval_id` is a high-entropy, unguessable token (e.g., UUIDv4 or 256-bit random), scoped to one `(user_id, operation, resource_id, resource_version)` tuple.
---
 
## 7. Authentication Flow
 
1. Client establishes identity via a real IdP: OS keychain-backed OAuth token, enterprise SSO, or equivalent — **not** anything the MCP agent can see or influence.
2. Client attaches a bearer token / signed session identifier to the MCP transport session (HTTP header for HTTP/SSE transport; an out-of-band authenticated local channel for stdio, e.g., the launching desktop app injects a session token via an env var or local socket handshake that the agent process cannot read).
3. MCP Server validates the token against the IdP (signature, expiry, audience) on **every** request — not just at connection time — and derives `Principal.user_id` from the verified claims.
4. `Principal` is attached to the request context by middleware, before any tool handler runs. Tool handlers receive `ctx.principal.user_id`; they do not receive and must not accept a `user_id` field from the agent's tool-call arguments. If a `user_id` field is present in arguments, the server ignores it or rejects the call as malformed (fail closed).
5. Session binding: the approval channel (client UI) and the acting session must be the same authenticated identity — an approval cannot be issued by a different, unauthenticated party.
---
 
## 8. Authorization / Ownership Flow
 
For **every** EDIT/DELETE (and CREATE against a parent resource with ownership implications):
 
```
1. Load Principal from verified session            (never from agent input)
2. Load resource fresh from repository by resource_id
   → if not found: fail closed (404, no info leak about existence to unauthorized users)
3. Evaluate Policy Engine: is this operation class allowed for Principal at all?
4. Evaluate Ownership: record.owner_id == Principal.user_id
     OR Principal has an explicit delegated grant in an ACL table
   → if neither: reject, audit "authorization_denied", return generic error
5. Only if 3 and 4 both pass: proceed to build ActionProposal
6. Re-run steps 1–4 again, immediately before the DB write, after approval
   (closes the TOCTOU/race window — see §9)
```
 
GET follows a lighter, separate **read-access policy** (can be broader — e.g., team-visible read) but still resolves identity the same trusted way and never trusts an agent-asserted identity or role.
 
---
 
## 9. Race Conditions / TOCTOU Handling
 
Scenario from the prompt, mapped to controls:
 
1. Agent requests EDIT #123 → proposal built with `resource_version = 7` captured at proposal time.
2. Approval issued, scoped to `(resource_id=123, resource_version=7)`.
3. User approves.
4. **Another user edits #123 concurrently → `version` becomes 8.**
5. Agent attempts execution using the approved proposal.
6. Execution step re-reads the record, re-checks authorization, and issues:
```sql
   UPDATE resources
   SET field = ?, version = version + 1, updated_at = now()
   WHERE id = ? AND version = ?   -- version = 7 (the approved version)
```
   Zero rows affected ⇒ **conflict detected** ⇒ transaction aborted, approval marked `REJECTED (stale_version)`, audit event written, agent is told to re-request (fresh proposal, fresh approval — no auto-retry with silently updated data).
 
This is optimistic concurrency control (OCC): cheap, and correct because approval + execution are separated by human-scale time, so blocking locks across that gap would be impractical.
 
---
 
## 10. Database Access Architecture
 
- The agent process has **zero** database credentials, connection strings, or driver access — enforced at the process/container level (separate process, no shared secrets, no shared filesystem access to config).
- Only the Repository layer inside the MCP server holds a DB connection pool, using a **least-privilege DB role**:
  - No `DROP`, `ALTER`, `TRUNCATE`.
  - `SELECT/INSERT/UPDATE` on business tables only, never `DELETE` on `audit_events` (see §12).
  - Row-level security (if the DB supports it, e.g., Postgres RLS) as defense-in-depth, keyed on `owner_id = current_setting('app.current_user_id')`, set per-transaction by the app after authentication — a second, DB-enforced layer behind the application-level ownership check.
- **All queries are parameterized** (prepared statements / query builder with bound parameters) — no string concatenation, ever, closing SQL injection.
- **Foreign keys** enforce `resources.owner_id → users.id`, `approvals.resource_id → resources.id`, `audit_events.approval_id → approvals.id` (nullable, since some audit events precede an approval, e.g., authz denials).
- **Soft delete** for resources: a `deleted_at` timestamp + `status` column rather than physical row deletion, so audit history and undo remain possible; a separate, rarely-run, heavily-audited hard-purge job (not reachable from the MCP tool surface) handles real erasure for compliance (e.g., GDPR erasure requests), itself requiring separate authorization.
- **Isolation level:** `READ COMMITTED` is sufficient given explicit version-column OCC; use `SERIALIZABLE` (or `REPEATABLE READ` with explicit `SELECT ... FOR UPDATE`) only where multi-row invariants must hold across a single transaction (e.g., moving a value between two records).
---
 
## 11. Logging & Audit Architecture
 
**Every** pipeline stage emits a structured audit event keyed by `request_id` (generated server-side at ingress, propagated through every layer, returned to the client for correlation).
 
Recommended event fields (superset of the prompt's list):
```
request_id, approval_id, authenticated_user_id, session_id, client_type,
operation, resource_type, resource_id, resource_version_before,
authorization_decision, ownership_decision, proposed_changes (diff),
approval_status, approval_created_at, approval_decided_at, approval_decided_by,
execution_attempted_at, execution_result, resource_version_after,
error_code, error_reason, ip/client_context (if available)
```
 
**Never logged:** passwords, API keys/tokens, session cookies/bearer tokens (log a hash or last-4 only if needed for correlation), full PII beyond what's needed, raw request bodies containing secrets.
 
**Platform handling:**
- **stdio transport (macOS/Windows/Linux desktop MCP clients):** application/audit logs must **never** go to stdout/stdin — those are reserved for the MCP JSON-RPC stream. Logs go to: a rotating file under the OS-appropriate app-data directory (`~/Library/Logs/<app>` on macOS, `%APPDATA%\<app>\Logs` on Windows, `~/.local/state/<app>/logs` or `/var/log/<app>` on Linux), and/or `stderr`, and/or a local syslog/Event Log sink.
- **Containers:** log to `stdout/stderr` of the *container* (not the MCP stdio stream, which is a distinct in-process pipe) so the container runtime's log driver (Docker/Kubernetes) can ship it to the platform's aggregator (e.g., CloudWatch, Loki, ELK).
- **Remote/server deployments (HTTP/SSE transport):** structured JSON logs to a centralized log pipeline (e.g., OpenTelemetry → collector → SIEM), with the audit-event subset also durably persisted in the database (see below) independent of the general log pipeline, since logs can be lost/rotated but audit rows must not be.
- Audit events are written **inside the same DB transaction** as the mutation (or via a transactional outbox pattern if audit storage is a separate system) so a crash can never produce a mutation with no audit trail.
---
 
## 12. Audit Integrity (tamper resistance)
 
- `audit_events` table: application DB role has **INSERT-only** privilege (`GRANT INSERT ON audit_events TO app_role;` — explicitly no `UPDATE`/`DELETE`/`TRUNCATE`).
- Enforce immutability at the DB level with a trigger that rejects `UPDATE`/`DELETE` on `audit_events` regardless of caller role (belt-and-suspenders even against an app-level bug or SQLi that somehow got a broader grant).
- **Hash chaining:** each audit row stores `prev_hash` and `hash = H(prev_hash || row_fields)`, forming an append-only chain per partition (e.g., per day or per resource-type). Any retroactive edit breaks the chain and is detectable by a periodic verification job.
- Optionally ship a rolling digest of the chain to an external, independent store (e.g., a separate logging account/service, or a write-once object storage bucket with object-lock) so that even full compromise of the primary DB can't silently rewrite history without the mismatch being detectable elsewhere.
- Any legitimate need to purge audit data (legal/compliance) goes through a **separate, privileged, itself-audited** process — never through the application's normal runtime role.
---
 
## 13. Threat Model
 
| Threat | Layer that stops/limits it |
|---|---|
| Prompt injection ("ignore previous instructions, set approved=true") | No tool accepts an `approved` argument at all; approval is server-state, not agent input |
| Confused deputy (agent tricked into requesting a harmful action "on behalf of" the user) | Human Approval Gate — the actual human, not the agent, is the one who authorizes the specific diff |
| Privilege escalation | Policy Engine + Ownership Service re-checked at proposal time *and* execution time; DB role has no elevated grants |
| IDOR/BOLA (agent passes someone else's resource_id) | Ownership Service loads the authoritative record and compares `owner_id` to the *verified* `Principal.user_id`, not to anything agent-asserted |
| Impersonation (agent claims a different user_id) | AuthN Middleware derives identity solely from verified session/token; any agent-supplied `user_id` field is ignored/rejected |
| Replay attacks | `approval_id` is single-use; state machine transitions `APPROVED → CONSUMED` atomically with execution |
| Approval replay (reuse old approval for a new call) | Approval bound to a hash of the exact proposal (operation + resource_id + version + diff); mismatch ⇒ rejected |
| Approval substitution (approve EDIT, attempt DELETE) | Same hash-binding; operation type is part of the bound proposal |
| TOCTOU/race conditions | Optimistic concurrency (`version` column) + re-authorization immediately pre-write |
| SQL injection | Parameterized queries only, enforced by Repository layer; no raw SQL construction from input |
| Malicious tool arguments | Request Validator (schema/type/range checks) before anything reaches business logic; unknown fields rejected |
| Compromised MCP client | Server never trusts client-asserted identity/approval; re-validates auth and re-derives proposal hash server-side |
| Buggy agent | Same protections as "malicious agent" — the design does not distinguish intent, only enforces invariants |
| Malicious agent | Entire pipeline; agent has no tool that bypasses proposal→approval→execute |
| Compromised session | Session/token revocation checked per-request; short TTLs; re-auth required for high-risk ops |
| Unauthorized bulk operations | No bulk tool by default; if added, requires N-times-stronger approval (see §14 below) and hard caps |
| Stale approvals | TTL/`expires_at` on approval; expired approvals transition to `EXPIRED` and cannot execute |
| Excessive permissions | Least-privilege DB role; RBAC/ABAC scoped per operation; no superuser paths |
| Audit tampering | INSERT-only grant, DB trigger, hash chaining, external digest (see §12) |
| Accidental destructive actions | Mandatory human approval showing current vs. proposed value + destructive/reversible flags; soft-delete as a safety net |
 
---
 
## 14. Security Invariants
 
1. The agent process never holds a database connection or credential.
2. `Principal.user_id` is derived exclusively from a verified auth token/session; agent-supplied identity fields are never trusted.
3. No tool call, from the agent or otherwise, can set an operation's approval status to approved.
4. EDIT and DELETE re-check authorization and ownership immediately before the write, not just at proposal time.
5. Every approval is cryptographically bound to one exact `(operation, resource_id, resource_version, diff-hash)` tuple.
6. An approval transitions to `CONSUMED` atomically with its one and only execution attempt; it cannot be reused.
7. A write whose target `resource_version` no longer matches the current row is rejected, not silently reapplied to newer data.
8. Any failure or ambiguity in authentication, authorization, ownership, approval, resource state/version, transaction, or policy evaluation results in **no execution** (fail closed).
9. Every attempted mutation (approved, denied, expired, failed, or succeeded) produces exactly one durable, append-only audit event.
10. No MCP tool exposes direct repository/database primitives to the agent; the proposal/approval/execution pipeline is the only path to a write.
11. Bulk operations, if enabled at all, require an approval that itemizes and bounds the affected set — a single approval token never authorizes an open-ended or arbitrarily large operation.
12. Audit records cannot be updated or deleted by the application's runtime role.
---
 
## 15. Recommended Database Schema
 
```sql
-- Users: identity is populated from the trusted IdP, not created ad hoc by requests
CREATE TABLE users (
  id            UUID PRIMARY KEY,
  external_sub  TEXT UNIQUE NOT NULL,   -- IdP subject claim
  display_name  TEXT NOT NULL,
  role          TEXT NOT NULL DEFAULT 'member',
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  status        TEXT NOT NULL DEFAULT 'active'
);
 
CREATE TABLE resources (
  id            UUID PRIMARY KEY,
  resource_type TEXT NOT NULL,
  owner_id      UUID NOT NULL REFERENCES users(id),
  data          JSONB NOT NULL,
  version       BIGINT NOT NULL DEFAULT 1,
  status        TEXT NOT NULL DEFAULT 'active',   -- active | deleted
  deleted_at    TIMESTAMPTZ,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
 
-- Optional delegated access, kept separate from ownership so ownership stays simple
CREATE TABLE resource_grants (
  resource_id   UUID NOT NULL REFERENCES resources(id),
  grantee_id    UUID NOT NULL REFERENCES users(id),
  permission    TEXT NOT NULL,   -- e.g. 'read', 'edit'
  granted_by    UUID NOT NULL REFERENCES users(id),
  granted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (resource_id, grantee_id, permission)
);
 
CREATE TABLE approvals (
  id                  UUID PRIMARY KEY,
  request_id          UUID NOT NULL,
  requested_by_agent  BOOLEAN NOT NULL DEFAULT true,
  user_id             UUID NOT NULL REFERENCES users(id),
  operation           TEXT NOT NULL,   -- CREATE | EDIT | DELETE
  resource_type       TEXT NOT NULL,
  resource_id         UUID,            -- null allowed for CREATE
  resource_version    BIGINT,          -- captured at proposal time; null for CREATE
  proposal_hash        TEXT NOT NULL,  -- hash of exact operation+diff, immutable binding
  current_value        JSONB,
  proposed_value        JSONB NOT NULL,
  reason              TEXT,
  destructive         BOOLEAN NOT NULL DEFAULT false,
  reversible          BOOLEAN NOT NULL DEFAULT true,
  status              TEXT NOT NULL DEFAULT 'PENDING',  -- PENDING/APPROVED/DENIED/EXPIRED/CONSUMED/REJECTED
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at          TIMESTAMPTZ NOT NULL,
  decided_at          TIMESTAMPTZ,
  decided_by          UUID REFERENCES users(id)
);
CREATE UNIQUE INDEX ON approvals (id) WHERE status = 'PENDING'; -- illustrative; real uniqueness is on id itself
 
CREATE TABLE audit_events (
  id                    BIGSERIAL PRIMARY KEY,
  request_id            UUID NOT NULL,
  approval_id           UUID REFERENCES approvals(id),
  authenticated_user_id UUID REFERENCES users(id),
  session_id            TEXT,
  client_type           TEXT,
  operation             TEXT NOT NULL,
  resource_type         TEXT,
  resource_id           UUID,
  authorization_decision TEXT,
  ownership_decision     TEXT,
  proposed_changes       JSONB,
  approval_status        TEXT,
  resource_version_before BIGINT,
  resource_version_after  BIGINT,
  execution_result        TEXT,
  error_reason             TEXT,
  prev_hash                TEXT,
  row_hash                 TEXT NOT NULL,
  created_at               TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Enforced immutability, defense-in-depth beyond the GRANT:
CREATE RULE audit_no_update AS ON UPDATE TO audit_events DO INSTEAD NOTHING;
CREATE RULE audit_no_delete AS ON DELETE TO audit_events DO INSTEAD NOTHING;
 
-- Least-privilege application role
REVOKE ALL ON audit_events FROM app_role;
GRANT INSERT, SELECT ON audit_events TO app_role;
GRANT SELECT, INSERT, UPDATE ON resources, approvals TO app_role;
-- no DELETE grant on resources: enforced soft-delete via status/deleted_at
```
 
---
 
## 16. Transaction / Concurrency Strategy
 
- **Proposal phase:** read-only; captures `resource_version` at read time; no lock held (approval may take minutes/hours — a human is in the loop).
- **Execution phase (single DB transaction):**
  1. `BEGIN`
  2. Re-authenticate + re-authorize + re-check ownership against a fresh read.
  3. `UPDATE resources SET ..., version = version + 1 WHERE id = $1 AND version = $2` (the version captured at approval time).
  4. Check affected row count == 1; if 0 → `ROLLBACK`, mark approval `REJECTED (stale_version)`.
  5. Insert the audit event in the same transaction.
  6. Mark the approval `CONSUMED` in the same transaction.
  7. `COMMIT`.
- Isolation level `READ COMMITTED` is enough given explicit OCC on `version`; escalate to `SERIALIZABLE`/row locks only for multi-record invariant operations.
- Idempotency: execution is keyed by `approval_id` — if the execution handler is invoked twice (e.g., retried webhook), the second call finds `status = CONSUMED` and no-ops safely rather than double-applying.
---
 
## 17. Recommended Project / Folder Structure
 
```
mcp-secure-crud/
├── src/
│   ├── transport/            # MCP stdio/HTTP/SSE adapters only — no business logic
│   ├── auth/
│   │   ├── authenticate.ts   # session/token verification → Principal
│   │   └── principal.ts
│   ├── validation/           # request schemas (zod/pydantic/etc.)
│   ├── policy/
│   │   ├── rbac.ts
│   │   └── ownership.ts
│   ├── proposals/
│   │   ├── proposalEngine.ts # builds ActionProposal + hash
│   │   └── diff.ts
│   ├── approvals/
│   │   ├── approvalService.ts
│   │   └── stateMachine.ts
│   ├── services/             # orchestration per use case (getData, createResource, ...)
│   ├── repository/
│   │   ├── resourcesRepo.ts  # only place with SQL, all parameterized
│   │   └── approvalsRepo.ts
│   ├── db/
│   │   ├── transaction.ts
│   │   └── migrations/
│   ├── audit/
│   │   ├── auditLogger.ts    # append-only writer + hash chaining
│   │   └── verifyChain.ts    # periodic integrity check job
│   ├── monitoring/           # anomaly detection, alerting hooks
│   └── tools/                # MCP tool definitions: get_data, request_action, check_approval_status
├── client-ui/                 # platform notification adapters (macOS/Windows/Linux/web)
├── tests/
│   ├── unit/
│   ├── integration/
│   └── adversarial/           # see §19
└── deploy/
    ├── docker/
    └── k8s/
```
 
---
 
## 18. Example Pseudocode
 
**GET**
```python
def get_data(ctx, resource_type, filters):
    principal = ctx.principal  # from verified session, never from args
    validate(filters)
    allowed_filter = read_policy.scope_filters(principal, resource_type, filters)
    rows = repo.query(resource_type, allowed_filter)  # parameterized
    audit.log(request_id=ctx.request_id, user=principal.user_id,
              operation="GET", resource_type=resource_type, result="ok")
    return rows
```
 
**request_action for CREATE/EDIT/DELETE**
```python
def request_action(ctx, operation, resource_type, resource_id, proposed_changes, reason=None):
    principal = ctx.principal
    validate_schema(operation, resource_type, proposed_changes)
 
    if operation in ("EDIT", "DELETE"):
        record = repo.get_authoritative(resource_type, resource_id)
        if record is None:
            audit.log(ctx.request_id, principal.user_id, operation, resource_id,
                       authz="denied", reason="not_found"); fail_closed("not_found")
        if not ownership.check(principal, record):
            audit.log(ctx.request_id, principal.user_id, operation, resource_id,
                       authz="denied", ownership="denied")
            fail_closed("forbidden")
        resource_version = record.version
        current_value = record.data
    else:  # CREATE
        resource_version, current_value = None, None
 
    if not policy.allowed(principal, operation, resource_type):
        audit.log(ctx.request_id, principal.user_id, operation, resource_id, authz="denied")
        fail_closed("forbidden")
 
    proposal = proposal_engine.build(
        operation, resource_type, resource_id, resource_version,
        current_value, proposed_changes, reason,
        destructive=(operation == "DELETE"))
 
    approval = approval_service.create(
        user_id=principal.user_id, proposal=proposal, ttl=DEFAULT_TTL)
 
    notify_client(principal, approval)  # desktop/sound/browser/terminal per platform
    audit.log(ctx.request_id, principal.user_id, operation, resource_id,
              approval_id=approval.id, approval_status="PENDING")
 
    return {"approval_id": approval.id, "status": "PENDING", "expires_at": approval.expires_at}
```
 
**Execution — triggered ONLY by the human's approval action, never by agent tool call**
```python
def on_user_decision(ctx, approval_id, decision, decided_by_user):
    approval = approvals_repo.get(approval_id)
    if approval is None or approval.status != "PENDING":
        fail_closed("invalid_or_already_decided")
    if decided_by_user.user_id != approval.user_id:
        fail_closed("decision_identity_mismatch")   # only the requesting user may decide
    if now() > approval.expires_at:
        approvals_repo.mark(approval_id, "EXPIRED"); fail_closed("expired")
 
    if decision == "DENY":
        approvals_repo.mark(approval_id, "DENIED")
        audit.log(...); return
 
    approvals_repo.mark(approval_id, "APPROVED")
 
    with db.transaction() as tx:
        principal = reauthenticate(decided_by_user)               # re-check #1
        record = repo.get_authoritative(approval.resource_type, approval.resource_id, tx=tx)
        if approval.operation in ("EDIT", "DELETE"):
            if record is None or not ownership.check(principal, record):   # re-check #2
                tx.rollback(); approvals_repo.mark(approval_id, "REJECTED"); return fail_closed("forbidden")
            if record.version != approval.resource_version:                # OCC re-check #3
                tx.rollback(); approvals_repo.mark(approval_id, "REJECTED_STALE"); return fail_closed("stale")
        if hash(approval.proposal) != approval.proposal_hash:              # tamper re-check #4
            tx.rollback(); return fail_closed("proposal_mismatch")
 
        result = repo.execute(approval, tx=tx)  # the actual INSERT/UPDATE (soft-delete for DELETE)
        approvals_repo.mark(approval_id, "CONSUMED", tx=tx)
        audit.log(ctx.request_id, principal.user_id, approval.operation,
                  approval.resource_id, approval_id=approval_id,
                  approval_status="CONSUMED", execution_result="success", tx=tx)
        tx.commit()
    return result
```
 
---
 
## 19. Safe vs. Unsafe Approval Workflow
 
**Unsafe:**
```python
def edit_record(id, new_values, approved):
    if approved:
        db.execute(f"UPDATE resources SET data = '{new_values}' WHERE id = {id}")
```
Fails because: `approved` is caller-supplied (agent can set it); no identity/ownership check; string-built SQL (injection); no version check (race conditions silently overwrite); no audit trail; a single call does authz, approval, and execution all at once with no server-held state in between — nothing stops a prompt-injected agent from calling this directly with `approved=True`.
 
**Safe:** the `request_action` → `approval_service.create` → out-of-band human decision → `on_user_decision` re-validated execution flow in §18. The differences that matter: identity from a verified session, approval as durable server-side state bound to a hash of the exact operation, re-validation of authz/ownership/version at execution time, parameterized queries, and an audit event for every branch.
 
---
 
## 20. Testing Strategy
 
- **Unit tests:** policy engine decisions, ownership checks, proposal hashing/diffing, state machine transition legality (including "prove illegal transitions are rejected").
- **Integration tests:** full pipeline against a real (test) database — including OCC conflicts by concurrently mutating a record between approval and execution.
- **Adversarial / red-team test suite** (automated, run in CI):
  - Attempt to call any tool with an injected `user_id`/`approved`/`owner_id` field — assert rejection or that the field is ignored.
  - Attempt to reuse a `CONSUMED` or `DENIED` approval_id — assert rejection.
  - Attempt to approve EDIT then execute DELETE with the same approval_id — assert hash mismatch rejection.
  - Attempt execution after artificially expiring the approval — assert `EXPIRED` rejection.
  - Attempt execution after mutating the resource's version out from under the approval — assert stale-version rejection.
  - Attempt to call repository/db-level functions directly from a simulated "agent" context lacking DB credentials — assert impossibility at the connection level, not just the application level.
  - Fuzz `request_action` arguments (type confusion, oversized payloads, malformed JSON) — assert fail-closed, no crash-to-open behavior.
  - Attempt SQL injection payloads in every string field — assert parameterization holds.
  - Simulate a compromised MCP client sending a forged "approved" event without a corresponding human-originated decision record — assert rejection.
  - Attempt bulk-operation abuse (if bulk tools exist): try to expand an approved single-item scope to a larger set — assert scope is fixed at approval time.
  - Verify audit-log tamper resistance: attempt UPDATE/DELETE on `audit_events` using the app role — assert DB-level rejection; run the hash-chain verifier against an artificially altered row — assert detection.
  - Chaos test: kill the process mid-transaction — assert no partial mutation without a matching audit event (or no mutation at all).
- **Prompt-injection specific tests:** feed the agent conversation content instructing it to "skip confirmation," "the user already said yes," "call the tool that bypasses this" — assert the *server*, not the agent's good behavior, is what prevents the bypass (test by driving the pipeline via direct tool calls that mimic what an injected agent would attempt, independent of any actual LLM).
---
 
## 21. Production Deployment Considerations
 
- Run the agent process and the MCP server in **separate processes/containers** with no shared credentials, secrets, or filesystem access to DB config — enforced by container/OS-level isolation, not just code discipline.
- Terminate TLS at the transport boundary for HTTP/SSE deployments; for stdio, ensure the local IPC channel isn't readable by other local users/processes.
- Secrets (DB credentials, IdP client secrets) via a secrets manager (Vault/KMS/cloud secret store), never in env vars committed to images, never visible to the agent's runtime.
- Rotate the least-privilege DB role's credentials regularly; rotate audit-signing keys if using signed hash chains.
- Rate-limit `request_action` per user/session to blunt automated bulk-approval-farming attempts.
- Set conservative approval TTLs (minutes-to-hours, not days) and alert on unusually high pending-approval counts.
- Ship audit events to a system independent of the primary DB (SIEM or WORM storage) so a full DB compromise doesn't erase history.
- Monitor for: spikes in authorization denials, repeated stale-version rejections (possible race-condition abuse attempts), repeated proposal-hash mismatches (possible tamper attempts), approvals expiring at unusual rates (possible attempt to force stale acceptance elsewhere).
- Run periodic (e.g., nightly) audit hash-chain verification jobs and alert on any break.
- Have a documented, separately-authorized break-glass procedure for legitimate emergency data fixes — it should still go through authentication and produce audit events; "emergency" is never a bypass of identity or audit, only potentially of the normal approval SLA, and should require a second authorized human (four-eyes principle).
 