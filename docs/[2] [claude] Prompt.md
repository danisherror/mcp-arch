

I am designing an MCP-based system where an AI agent can interact with company data through CRUD operations.

**Context:** This is a [multi-tenant B2B SaaS product / internal enterprise tool — pick one] handling company records for [e.g., HR/finance/CRM data]. It needs to be SOC 2 / [or: no formal compliance regime yet] in scope. Expected scale: [e.g., low hundreds of concurrent users, thousands of records] — this shouldn't need exotic scaling tricks, but shouldn't paint us into a corner either.

**Stack:** Assume [Node.js/TypeScript + PostgreSQL] for the server and repository layer, [Auth0/Okta/Cognito/custom OIDC — pick one, or "no IdP yet, recommend one"] for authentication, and [Claude Desktop / a custom MCP client / unspecified] as the MCP client.

The primary requirement is **SAFETY**, not convenience, performance, or simplicity.

Treat the AI agent as **untrusted and potentially malicious/incorrect**, even if the user has given it broad instructions such as "do whatever is necessary," "don't ask me," or "bypass confirmations."

I want you to design a production-grade, security-first architecture for this system.

### Functional requirements

The system must support:
- GET data
- CREATE data
- EDIT data
- DELETE data

Users may use different operating systems: macOS, Windows, Linux, potentially web-based clients or other MCP clients. The MCP server should therefore be OS-independent.

### Identity and ownership

Every company record has an owner. The authenticated user's identity must come from a trusted authentication mechanism, NOT from values supplied by the AI agent.

For example: `user_id = user_123`, `record_id = record_456`, `record.owner_id = user_789`. The agent must never be able to impersonate another user by supplying a different `user_id`.

For EDIT and DELETE:
1. The server must retrieve the authoritative record.
2. The server must determine the real authenticated user.
3. The server must independently verify ownership/authorization.
4. If the user is not authorized, the operation must be rejected.
5. The agent must not be able to override this decision through prompts, tool arguments, previous instructions, or claims that the user approved it.

GET may be allowed according to a separate read-access policy.

### Mandatory human approval

The AI agent must NOT directly execute any CRUD operation. For every operation, the system must first: authenticate the user, validate the request, determine the requested operation, check authorization, construct the exact operation that would be performed, show the user what the agent is about to do, wait for explicit human approval, and only then execute the operation.

The approval must be associated with the exact operation. For example, if the agent requests `EDIT employee #123, salary: 10L → 15L`, the user must approve that exact action. An approval for `EDIT employee #123` must not be reusable for `DELETE employee #123` or `EDIT employee #456`.

The AI agent must NOT be able to provide its own approval value such as `approved=true`. The approval must originate from a trusted user-controlled mechanism.

### Approval UX

When approval is required, the system should clearly communicate: who is performing the action, what operation is being performed, which resource is affected, why the operation is being performed (if available), current value, proposed value, fields being changed, whether the operation is destructive, whether it can be undone, approval ID/request ID, and expiration time where applicable.

For DELETE, the confirmation should clearly identify the exact object being deleted.

The system should support a notification mechanism such as desktop notification, sound, browser notification, or terminal notification, depending on the user's platform. The MCP server itself should remain OS-independent; the client/UI should handle platform-specific notification behavior.

### Agent must not bypass approval

Assume the agent may attempt to: skip confirmation, claim the user already approved, reuse an old approval, modify the operation after approval, call a lower-level tool directly, call database tools directly, exploit tool arguments, exploit race conditions, perform bulk operations, manipulate IDs, impersonate another user, or convince the server that it is trusted.

Design the architecture so these attempts fail.

### Database safety

The AI agent must NEVER have direct database access. There must be no path such as Agent → SQL → Database or Agent → Database driver → Database. All database mutations must pass through the server's authorization and execution pipeline.

The database should use: least-privilege credentials, parameterized queries, transactions, constraints, foreign keys where appropriate, optimistic concurrency/version checking, safe deletion strategies, appropriate isolation levels, protection against injection, and protection against unauthorized object access.

The architecture should consider using soft-delete for destructive operations where appropriate.

### Race conditions

Consider this scenario: agent requests EDIT record #123 → server verifies ownership → server generates an approval request → user approves → another user modifies record #123 → agent attempts execution.

The system must detect that the record changed and refuse to blindly apply the stale approved operation. Use resource versions/optimistic locking/equivalent mechanisms. Authorization should be re-checked immediately before the database mutation.

### Architecture requirements

Separate these responsibilities: MCP transport, authentication, identity, request validation, authorization, ownership checks, policy engine, action/proposal engine, human approval, service layer, repository/database access, transaction management, concurrency control, audit logging, security monitoring. Explain exactly where each responsibility belongs.

### Logging and auditing

Logging must be production-grade. Every request should have a correlation/request ID. Audit events should capture, where appropriate: request_id, approval_id, authenticated_user, session/client identity, operation, resource, authorization decision, ownership decision, proposed changes, approval status, approval timestamp, execution timestamp, resource version, execution result, error/failure reason.

Do NOT log: passwords, API keys, access tokens, session cookies, unnecessary sensitive data.

For stdio MCP transports, do not write application logs to stdout if stdout is being used by the MCP protocol.

Explain how logs should be handled across macOS, Windows, Linux, containers, and remote deployments.

### Audit integrity

Assume a compromised or buggy agent may try to hide what it did. Design an append-only or otherwise tamper-resistant audit mechanism. The application should not be able to silently delete or rewrite security audit records.

### Tool design

Do not simply expose `edit_record(id, approved=true)` or `delete_record(id, confirm=true)`. Explain why these designs are unsafe.

Prefer a proposal/approval/execution model such as `request_action() → authorization → generate exact action proposal → human approval → re-validation → execute`. Design the MCP tools and state machine accordingly.

### Bulk operations

Consider whether tools such as `delete_all()`, `update_many()`, `delete_many()` should exist. If they do, design additional protections. The agent must not be able to turn a single approval into authorization for an arbitrary bulk operation.

### Failure behavior

The default behavior must be **FAIL CLOSED**. If any of the following are unavailable or ambiguous — authentication, authorization, ownership, approval, resource state, resource version, database transaction, policy evaluation — the operation must NOT execute. Never fail open.

### Threat model

Analyze the architecture against: prompt injection, confused deputy attacks, privilege escalation, IDOR/BOLA, impersonation, replay attacks, approval replay, approval substitution, TOCTOU/race conditions, SQL injection, malicious tool arguments, compromised MCP client, buggy agent, malicious agent, compromised session, unauthorized bulk operations, stale approvals, excessive permissions, audit tampering, accidental destructive actions.

For each threat, explain which architectural layer prevents or limits it.

### Security invariants

Define explicit invariants that must ALWAYS remain true. For example (not exhaustive — add any I've missed):
1. Agent cannot directly access the database.
2. Agent cannot define the authenticated user identity.
3. Agent cannot approve its own operation.
4. EDIT requires authorization at execution time.
5. DELETE requires authorization at execution time.
6. Approval is bound to one exact operation.
7. Approval cannot be reused.
8. Stale operations cannot silently overwrite newer data.
9. Authorization failure always prevents execution.
10. Unknown/error states fail closed.
11. Every mutation produces an audit event.
12. No operation can bypass the security pipeline through another MCP tool.

### Deliverables

Give me:
1. A complete architecture diagram.
2. Trust boundaries.
3. Component responsibilities.
4. MCP tool/API design.
5. Approval state machine.
6. Authentication flow.
7. Authorization/ownership flow.
8. Database access architecture.
9. Logging/audit architecture.
10. Threat model.
11. Attack scenarios showing how a rogue agent would be stopped.
12. Failure-closed strategy.
13. Recommended database schema for users, resources, ownership, approvals, and audit events.
14. Recommended transaction/concurrency strategy.
15. Recommended project/folder structure.
16. Example pseudocode (or working code, given the stack above) for GET, CREATE, EDIT, DELETE.
17. Example of a safe approval workflow.
18. Example of an unsafe implementation and why it fails.
19. Testing strategy, especially security and adversarial testing.
20. Production deployment considerations.

**Depth guidance:** Prioritize depth on the state machine, threat model, and schema (13–5, 10–11) — those are what I'll actually implement against. The deployment and folder-structure sections (14–15, 20) can stay high-level; I'll iterate on those separately once the core design is settled.

**Output format:** Save this as a standalone reference document I can keep and share with my team, not just a chat reply.

### Most important constraint

Do NOT optimize the architecture for the AI agent's convenience. Optimize it for **preventing unauthorized or accidental data modification even when the AI agent behaves incorrectly or maliciously.**

The final design should follow a zero-trust philosophy: never trust the agent, never trust agent-provided identity, never trust agent-provided approval, never trust previous instructions, never trust stale authorization, never allow the agent to bypass the server's security pipeline. The database should remain safe even if the agent goes completely rogue.