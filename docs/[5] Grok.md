**Security-first, zero-trust architecture for an MCP-based CRUD system.** The AI agent is treated as completely untrusted (malicious, buggy, prompt-injected, or compromised). No optimization for agent convenience. Every mutation requires a fresh, human-approved, exactly-bound proposal that is re-validated immediately before execution. The database remains safe even if the agent is fully rogue.

### 1. Architecture Diagram (logical)

```
[Human User / Client UI]
        │ (trusted auth channel + approval UI)
        ▼
┌───────────────────────────────────────────────────────────────┐
│ MCP Client (OS-specific UI/notifications)                     │
│ - Desktop/browser/terminal notifications                      │
│ - Approval UX (exact diff, ownership, risk flags)             │
└───────────────────────────┬───────────────────────────────────┘
                            │ MCP transport (stdio / SSE / HTTP)
                            ▼
┌───────────────────────────────────────────────────────────────┐
│ MCP Server (OS-independent, zero-trust)                       │
│                                                               │
│  MCP Transport Layer                                          │
│       │                                                       │
│  Authentication & Identity Layer  ← trusted IdP / session     │
│       │                                                       │
│  Request Validation & Schema Layer                            │
│       │                                                       │
│  Policy Engine + Authorization + Ownership                    │
│       │                                                       │
│  Action / Proposal Engine                                     │
│       │                                                       │
│  Human Approval Service (state machine)                       │
│       │ (only after explicit human approval)                  │
│  Service Layer (re-authz + concurrency)                       │
│       │                                                       │
│  Repository / Transaction Manager                             │
│       │                                                       │
│  Database (least-privilege, parameterized, optimistic lock)   │
│                                                               │
│  Cross-cutting: Audit Logger (append-only), Monitoring        │
└───────────────────────────────────────────────────────────────┘
```

Agent tools only ever call into the Proposal Engine. There is no path from agent → repository → database.

### 2. Trust Boundaries

- **Outside**: AI agent, MCP client process (if compromised), network, user machine OS.
- **Trusted boundary 1**: MCP Server process (runs with minimal privileges).
- **Trusted boundary 2**: Authentication source (IdP / SSO / OS session / short-lived token issued by a trusted client after human login). Agent-supplied `user_id` is ignored.
- **Trusted boundary 3**: Approval channel (human-controlled UI that signs or cryptographically binds an approval token to a specific proposal ID + exact operation hash).
- **Trusted boundary 4**: Database credentials (least-privilege service account; agent never sees them).
- **Audit store**: Append-only / WORM / separate write-only principal or external SIEM.

Agent arguments, previous conversation, tool call history, and claims of “user already approved” are never trusted.

### 3. Component Responsibilities

| Responsibility              | Component                          | Notes |
|-----------------------------|------------------------------------|-------|
| MCP transport               | MCP Server transport adapter       | stdio/SSE/HTTP; no business logic |
| Authentication              | Auth middleware                    | Extracts identity from trusted token/session only |
| Identity                    | Identity service                   | Maps to internal `user_id`; never from agent args |
| Request validation          | Schema + input sanitizer           | Strict typed schemas; reject unknown fields |
| Authorization / ownership   | Policy engine                      | Evaluated on every proposal and again at execution |
| Action/proposal             | Proposal engine                    | Builds immutable, exact, versioned ActionProposal |
| Human approval              | Approval service + state machine   | Human-originated only; bound to proposal hash |
| Service layer               | Domain services                    | Orchestrates re-check → execute |
| Repository / DB access      | Repository + transaction manager   | Only callable from service layer |
| Concurrency control         | Optimistic versioning              | Version check inside transaction |
| Audit logging               | Audit logger (append-only)         | Correlation ID on every event |
| Security monitoring         | Metrics + alerts                   | Failed authz, approval anomalies, rate limits |

### 4. MCP Tool / API Design

Unsafe patterns (explicitly forbidden):
- `edit_record(id, fields, approved=true)`
- `delete_record(id, confirm=true)`
- Any tool that accepts an approval flag, user_id, or direct mutation payload.

Safe tools (proposal/approval/execution model):

```text
propose_action(operation: "CREATE"|"EDIT"|"DELETE"|"GET",
               resource_type: string,
               resource_id?: string,          // required for EDIT/DELETE/GET-by-id
               payload?: object,              // CREATE or EDIT fields only
               reason?: string) → Proposal

get_proposal(proposal_id: string) → ProposalStatus

list_pending_approvals() → [ProposalSummary]   // for the authenticated user only

# Execution is NEVER a tool the agent can call.
# After human approval, a privileged internal path (or a short-lived,
# single-use execution token issued only to the trusted client) triggers:
execute_approved_proposal(proposal_id: string, approval_token: string)
```

- `propose_action` only creates a proposal record after authn + authz + ownership check. It never mutates data.
- The agent never receives an executable mutation primitive.
- Bulk tools (`delete_many`, `update_many`) either do not exist or require a separate multi-resource proposal that still needs one human approval per resource (or a high-risk multi-approval workflow). A single approval can never authorize an arbitrary set.

### 5. Approval State Machine

```
PENDING_VALIDATION
    → (authn + authz + ownership + schema) → PROPOSED
    → (human notified) → AWAITING_APPROVAL
        → (human rejects / expires) → REJECTED / EXPIRED
        → (human approves with exact hash match) → APPROVED
            → (re-authz + version check + transaction) → EXECUTED
            → (any failure) → FAILED (fail-closed)
```

- Each proposal carries a cryptographic hash of `(operation, resource_type, resource_id, exact payload, expected_version, requester)`.
- Approval token is bound to that hash + proposal_id + expiry + authenticated user.
- Re-use, substitution, or modification after approval is rejected.
- Approvals are single-use and short-lived (e.g., 5–15 minutes).

### 6. Authentication Flow

1. Human authenticates via trusted mechanism (SSO, OAuth, OS keychain, short-lived JWT issued by a trusted client after login).
2. MCP client presents a short-lived, audience-restricted token to the MCP server on every request (or uses a secure session).
3. Server validates token signature, expiry, audience, and issuer. Extracts `authenticated_user_id`.
4. Agent-supplied identity fields are discarded.
5. Session is bound to the MCP connection or request; no long-lived agent credentials.

### 7. Authorization / Ownership Flow

For every proposal and again at execution:

1. Load authoritative record (or parent for CREATE).
2. Determine real `authenticated_user_id`.
3. Evaluate policy: ownership (`record.owner_id == user_id` or role-based exception) + any additional ACLs.
4. For EDIT/DELETE: reject if not owner (or not authorized).
5. For GET: apply separate read policy.
6. Record decision in audit log.
7. At execution time the same checks are repeated inside the transaction (TOCTOU protection).

### 8. Database Access Architecture

- Agent never obtains DB credentials or a connection.
- Only the service layer obtains a connection from a pool using a least-privilege DB role that can only call specific stored procedures or has row-level security + column privileges.
- All queries are parameterized.
- Mutations occur inside transactions with appropriate isolation (e.g., REPEATABLE READ or SERIALIZABLE for critical paths).
- Soft-delete preferred for DELETE (`deleted_at`, `deleted_by`); hard delete only under stricter controls.
- Optimistic concurrency: every mutable row has a `version` (or `updated_at` + checksum) column. UPDATE/DELETE includes `WHERE id = ? AND version = expected_version`. Mismatch aborts.
- Foreign keys, check constraints, and triggers enforce integrity.
- No dynamic SQL constructed from agent input.

### 9. Logging / Audit Architecture

- Every request receives a correlation / request ID (propagated).
- Audit events are written to an append-only store (separate table with insert-only grants, or external WORM/SIEM, or hash-chained log).
- Application process cannot UPDATE or DELETE audit rows.
- For stdio MCP: application logs go to stderr or a dedicated file/socket; never to stdout (MCP protocol uses stdout).
- Cross-platform: structured JSON logs → file / syslog / Windows Event Log / container stdout (captured by orchestrator) / remote collector.
- Fields (when present): request_id, proposal_id, approval_id, authenticated_user, client_identity, operation, resource, authz decision, ownership decision, proposed diff, approval status/timestamp, execution timestamp, resource version before/after, result, error reason.
- Never log secrets, tokens, passwords, or full PII beyond necessity.

### 10. Threat Model (selected mappings)

| Threat                        | Mitigations |
|-------------------------------|-------------|
| Prompt injection / malicious agent | Agent has no mutation primitives; only proposes |
| Confused deputy               | Server never trusts agent-supplied identity or approval |
| Privilege escalation / IDOR / BOLA | Authz + ownership re-checked at execution; policy engine |
| Impersonation                 | Identity only from trusted token |
| Approval replay / substitution | Single-use, hash-bound, short-lived tokens |
| TOCTOU / race                 | Version check + re-authz inside transaction |
| SQL injection                 | Parameterized queries only; no agent SQL |
| Malicious tool arguments      | Strict schema validation; ignore extra fields |
| Compromised MCP client        | Approval still requires human action; tokens short-lived |
| Bulk / unauthorized ops       | No unbounded bulk tools; each resource needs its own proposal/approval |
| Audit tampering               | Append-only / separate principal / external store |
| Stale approvals               | Version + expiry + re-validation |
| Fail-open                     | Explicit fail-closed on any ambiguity |

### 11. Attack Scenarios (how a rogue agent is stopped)

- Agent calls `propose_action(..., approved=true)` → ignored; only creates a proposal.
- Agent supplies `user_id=admin` → discarded; real identity used.
- Agent re-uses an old approval_id → rejected (single-use + hash mismatch).
- Agent changes payload after approval → hash mismatch; execution refused.
- Agent tries to call a hypothetical low-level `execute_sql` tool → tool does not exist.
- Between approval and execution another user edits the record → version mismatch → abort.
- Agent requests DELETE of a record it does not own → ownership check fails at proposal and again at execution.
- Agent floods proposals → rate limits + human must still approve each one.

### 12. Failure-Closed Strategy

Any of the following causes immediate rejection (no mutation):
- Authentication missing / invalid / expired
- Authorization or ownership check fails or is indeterminate
- Proposal not in APPROVED state
- Approval token invalid, expired, already used, or hash mismatch
- Resource version mismatch
- Transaction cannot be started or committed
- Policy engine error
- Audit write failure (optional hard requirement)

Default: do nothing.

### 13. Recommended Database Schema (core)

```sql
users (
  id UUID PRIMARY KEY,
  ...
)

resources (          -- example generic; specialize per domain
  id UUID PRIMARY KEY,
  owner_id UUID NOT NULL REFERENCES users(id),
  version BIGINT NOT NULL DEFAULT 1,
  data JSONB NOT NULL,          -- or normalized columns
  created_at, updated_at,
  deleted_at TIMESTAMPTZ,       -- soft delete
  deleted_by UUID
)

action_proposals (
  id UUID PRIMARY KEY,
  requester_id UUID NOT NULL,
  operation TEXT NOT NULL,      -- CREATE|EDIT|DELETE|GET
  resource_type TEXT NOT NULL,
  resource_id UUID,
  payload JSONB,
  expected_version BIGINT,
  proposal_hash TEXT NOT NULL,  -- deterministic hash of exact op
  status TEXT NOT NULL,         -- PROPOSED|AWAITING_APPROVAL|APPROVED|...
  expires_at TIMESTAMPTZ,
  created_at
)

approvals (
  id UUID PRIMARY KEY,
  proposal_id UUID NOT NULL REFERENCES action_proposals(id),
  approver_id UUID NOT NULL,
  approval_token_hash TEXT NOT NULL,  -- store hash only
  bound_hash TEXT NOT NULL,           -- must match proposal_hash
  used_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ
)

audit_events (          -- insert-only grants
  id BIGSERIAL PRIMARY KEY,
  request_id UUID,
  proposal_id UUID,
  approval_id UUID,
  authenticated_user UUID,
  event_type TEXT,
  details JSONB,        -- redacted
  created_at TIMESTAMPTZ DEFAULT now()
)
```

Indexes on owner_id, status, expires_at, proposal_hash, etc. Row-level security optional extra layer.

### 14. Transaction / Concurrency Strategy

- Proposal creation: short read transaction + insert.
- Execution: 
  1. BEGIN
  2. SELECT … FOR UPDATE (or equivalent) of the resource
  3. Re-evaluate ownership + policy
  4. Check version == expected_version
  5. Apply change (or soft-delete)
  6. Increment version
  7. Mark proposal EXECUTED
  8. Insert audit event
  9. COMMIT
- On any check failure → ROLLBACK.
- Isolation: at least REPEATABLE READ; SERIALIZABLE for highest-risk resources.

### 15. Recommended Project / Folder Structure

```
mcp-secure-server/
├── cmd/server/                 # entrypoint
├── internal/
│   ├── transport/              # MCP adapters
│   ├── auth/                   # token validation, identity
│   ├── validation/
│   ├── policy/                 # authorization + ownership
│   ├── proposal/               # action engine + state machine
│   ├── approval/
│   ├── service/                # domain orchestration
│   ├── repository/             # DB access only
│   ├── audit/
│   └── config/
├── pkg/                        # shared types, hashing
├── migrations/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── adversarial/            # malicious agent simulations
└── deploy/                     # least-privilege configs, secrets
```

### 16. Example Pseudocode (EDIT path)

```text
function propose_edit(resource_id, payload, reason):
  user = authenticate()                    // fail-closed
  validate_schema(payload)
  record = repo.get(resource_id)           // authoritative
  assert ownership_or_policy(user, record) // fail-closed
  proposal = create_proposal(
    op="EDIT", resource_id, payload,
    expected_version=record.version,
    hash=hash(exact_op)
  )
  audit("PROPOSED", ...)
  notify_human(proposal)                   // OS-specific client
  return proposal_id

# Human approves via trusted UI → approval token issued bound to hash

function execute_approved(proposal_id, approval_token):
  BEGIN
    proposal = load_for_update(proposal_id)
    assert proposal.status == APPROVED
    assert approval_token_valid_and_unused_and_hash_matches(proposal)
    user = authenticate()                  // again
    record = repo.get_for_update(proposal.resource_id)
    assert ownership_or_policy(user, record)
    assert record.version == proposal.expected_version
    apply_edit(record, proposal.payload)
    record.version += 1
    mark_proposal_executed()
    mark_approval_used()
    audit("EXECUTED", before=..., after=...)
  COMMIT
  // any assert failure → ROLLBACK + audit failure
```

CREATE / DELETE follow the same pattern (DELETE sets `deleted_at` + version check). GET may short-circuit to a read-only path with its own policy, still authenticated.

### 17. Safe Approval Workflow Example

1. Agent: `propose_action(EDIT, "employee", "123", {salary: 1500000}, "raise")`
2. Server authenticates user_A, loads employee #123 (owner=user_A, version=7), creates proposal P with hash H, notifies user.
3. Client UI shows exact diff, current vs proposed, risk flags, approval ID, expiry.
4. User clicks “Approve this exact change”.
5. Client obtains short-lived approval token T bound to (P, H, user_A).
6. Trusted path calls `execute_approved(P, T)`.
7. Server re-checks everything inside a transaction; succeeds or fails closed.
8. Audit trail records the full chain.

### 18. Unsafe Implementation (and why it fails)

```text
// BAD
tool edit_record(id, fields, approved=false):
  if approved:
    db.update(id, fields)   // agent can set approved=true
```

Failures: agent forges approval, supplies wrong identity, no version check, no re-authz, direct DB path, no exact-operation binding, no audit integrity, fails open on missing checks.

### 19. Testing Strategy

- Unit: policy engine, hash binding, state machine transitions, fail-closed paths.
- Integration: full propose → approve → execute with real DB (optimistic lock races).
- Adversarial / red-team:
  - Agent tries every bypass (forged approval, wrong user_id, payload mutation, bulk, replay, concurrent modification).
  - Compromised client simulation.
  - Prompt-injection style tool argument fuzzing.
  - Audit immutability tests (attempt UPDATE/DELETE on audit table must fail).
- Chaos: kill DB mid-transaction, expired tokens, clock skew, network partitions → must fail closed.
- Property-based: “no mutation occurs unless a valid human approval for that exact hash exists and version matches”.

### 20. Production Deployment Considerations

- Run MCP server with least privilege (no root, minimal capabilities, read-only filesystem where possible).
- Secrets via vault / cloud secret manager; never in env for long-lived credentials.
- Network: mTLS or equivalent between client and server if remote; restrict egress.
- Rate limiting and anomaly detection on proposal volume / failed authz.
- Separate audit sink (immutable storage or SIEM).
- Monitoring: alert on approval anomalies, version conflicts, high rejection rates.
- Container / k8s: non-root, seccomp, read-only rootfs, resource limits.
- Multi-OS clients: each platform implements its own notification + approval UI; server remains pure.
- Backup / recovery: point-in-time recovery; soft-deletes aid recovery.
- Key rotation for any signing keys used in approval tokens.
- Regular adversarial exercises against the live system.

**Core invariants that must always hold**

1. Agent has no direct database access path.
2. Authenticated identity is never taken from agent input.
3. Agent cannot approve its own operations.
4. Every mutation is preceded by a human approval of the exact operation hash.
5. Authorization and ownership are re-evaluated at execution time.
6. Approvals are single-use and bound to one exact proposal.
7. Version mismatch or any check failure aborts the mutation.
8. Unknown / error states fail closed.
9. Every mutation attempt (success or failure) produces an immutable audit event.
10. No MCP tool exists that can bypass the proposal → human-approval → re-validation → execute pipeline.

This design deliberately makes the agent’s life harder in exchange for making unauthorized or accidental modification extremely difficult. The database stays protected even when the agent is fully adversarial.