I am designing an MCP-based system where an AI agent can interact with company data through CRUD operations.

The primary requirement is **SAFETY**, not convenience, performance, or simplicity.

Treat the AI agent as **untrusted and potentially malicious/incorrect**, even if the user has given it broad instructions such as "do whatever is necessary", "don't ask me", or "bypass confirmations".

I want you to design a production-grade, security-first architecture for this system.

## Functional requirements

The system must support:

* GET data
* CREATE data
* EDIT data
* DELETE data

Users may use different operating systems:

* macOS
* Windows
* Linux
* potentially web-based clients or other MCP clients

The MCP server should therefore be OS-independent.

## Identity and ownership

Every company record has an owner.

The authenticated user's identity must come from a trusted authentication mechanism, NOT from values supplied by the AI agent.

For example:

* user_id = user_123
* record_id = record_456
* record.owner_id = user_789

The agent must never be able to impersonate another user by supplying a different user_id.

For EDIT and DELETE:

1. The server must retrieve the authoritative record.
2. The server must determine the real authenticated user.
3. The server must independently verify ownership/authorization.
4. If the user is not authorized, the operation must be rejected.
5. The agent must not be able to override this decision through prompts, tool arguments, previous instructions, or claims that the user approved it.

GET may be allowed according to a separate read-access policy.

## Mandatory human approval

The AI agent must NOT directly execute any CRUD operation.

For every operation, the system must first:

1. Authenticate the user.
2. Validate the request.
3. Determine the requested operation.
4. Check authorization.
5. Construct the exact operation that would be performed.
6. Show the user what the agent is about to do.
7. Wait for explicit human approval.
8. Only then execute the operation.

The approval must be associated with the exact operation.

For example, if the agent requests:

EDIT employee #123
salary: 10L → 15L

the user must approve that exact action.

An approval for:

EDIT employee #123

must not be reusable for:

DELETE employee #123

or:

EDIT employee #456.

The AI agent must NOT be able to provide its own approval value such as:

approved=true.

The approval must originate from a trusted user-controlled mechanism.

## Approval UX

When approval is required, the system should clearly communicate:

* Who is performing the action
* What operation is being performed
* Which resource is affected
* Why the operation is being performed, if available
* Current value
* Proposed value
* Fields being changed
* Whether the operation is destructive
* Whether it can be undone
* Approval ID / request ID
* Expiration time where applicable

For DELETE, the confirmation should clearly identify the exact object being deleted.

The system should support a notification mechanism such as:

* desktop notification
* sound
* browser notification
* terminal notification

depending on the user's platform.

The MCP server itself should remain OS-independent. The client/UI should handle platform-specific notification behavior.

## Agent must not bypass approval

Assume the agent may attempt:

* to skip confirmation
* to claim that the user already approved
* to reuse an old approval
* to modify the operation after approval
* to call a lower-level tool directly
* to call database tools directly
* to exploit tool arguments
* to exploit race conditions
* to perform bulk operations
* to manipulate IDs
* to impersonate another user
* to convince the server that it is trusted

Design the architecture so these attempts fail.

## Database safety

The AI agent must NEVER have direct database access.

There must be no path such as:

Agent → SQL → Database

or:

Agent → Database driver → Database.

All database mutations must pass through the server's authorization and execution pipeline.

The database should use:

* least-privilege credentials
* parameterized queries
* transactions
* constraints
* foreign keys where appropriate
* optimistic concurrency/version checking
* safe deletion strategies
* appropriate isolation levels
* protection against injection
* protection against unauthorized object access

The architecture should consider using soft-delete for destructive operations where appropriate.

## Race conditions

Consider this scenario:

1. Agent requests EDIT record #123.
2. Server verifies ownership.
3. Server generates an approval request.
4. User approves.
5. Another user modifies record #123.
6. Agent attempts execution.

The system must detect that the record changed and refuse to blindly apply the stale approved operation.

Use resource versions / optimistic locking / equivalent mechanisms.

Authorization should be re-checked immediately before the database mutation.

## Architecture requirements

Separate these responsibilities:

* MCP transport
* authentication
* identity
* request validation
* authorization
* ownership checks
* policy engine
* action/proposal engine
* human approval
* service layer
* repository/database access
* transaction management
* concurrency control
* audit logging
* security monitoring

Explain exactly where each responsibility belongs.

## Logging and auditing

Logging must be production-grade.

Every request should have a correlation/request ID.

Audit events should capture, where appropriate:

* request_id
* approval_id
* authenticated_user
* session/client identity
* operation
* resource
* authorization decision
* ownership decision
* proposed changes
* approval status
* approval timestamp
* execution timestamp
* resource version
* execution result
* error/failure reason

Do NOT log:

* passwords
* API keys
* access tokens
* session cookies
* unnecessary sensitive data

For stdio MCP transports, do not write application logs to stdout if stdout is being used by the MCP protocol.

Explain how logs should be handled across macOS, Windows, Linux, containers, and remote deployments.

## Audit integrity

Assume a compromised or buggy agent may try to hide what it did.

Design an append-only or otherwise tamper-resistant audit mechanism.

The application should not be able to silently delete or rewrite security audit records.

## Tool design

Do not simply expose:

edit_record(id, approved=true)

or:

delete_record(id, confirm=true).

Explain why these designs are unsafe.

Prefer a proposal/approval/execution model such as:

request_action()
→ authorization
→ generate exact action proposal
→ human approval
→ re-validation
→ execute

Design the MCP tools and state machine accordingly.

## Bulk operations

Consider whether tools such as:

delete_all()
update_many()
delete_many()

should exist.

If they do, design additional protections.

The agent must not be able to turn a single approval into authorization for an arbitrary bulk operation.

## Failure behavior

The default behavior must be:

**FAIL CLOSED.**

If any of the following are unavailable or ambiguous:

* authentication
* authorization
* ownership
* approval
* resource state
* resource version
* database transaction
* policy evaluation

the operation must NOT execute.

Never fail open.

## Threat model

Analyze the architecture against:

* prompt injection
* confused deputy attacks
* privilege escalation
* IDOR / BOLA
* impersonation
* replay attacks
* approval replay
* approval substitution
* TOCTOU/race conditions
* SQL injection
* malicious tool arguments
* compromised MCP client
* buggy agent
* malicious agent
* compromised session
* unauthorized bulk operations
* stale approvals
* excessive permissions
* audit tampering
* accidental destructive actions

For each threat, explain which architectural layer prevents or limits it.

## Security invariants

Define explicit invariants that must ALWAYS remain true.

For example:

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

## Deliverables

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
16. Example pseudocode for GET, CREATE, EDIT, DELETE.
17. Example of a safe approval workflow.
18. Example of an unsafe implementation and why it fails.
19. Testing strategy, especially security and adversarial testing.
20. Production deployment considerations.

## Most important constraint

Do NOT optimize the architecture for the AI agent's convenience.

Optimize it for:

**preventing unauthorized or accidental data modification even when the AI agent behaves incorrectly or maliciously.**

The final design should follow a zero-trust philosophy:

**Never trust the agent.
Never trust agent-provided identity.
Never trust agent-provided approval.
Never trust previous instructions.
Never trust stale authorization.
Never allow the agent to bypass the server's security pipeline.**

The database should remain safe even if the agent goes completely rogue.
