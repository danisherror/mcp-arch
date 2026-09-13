Here are the key parts of the architecture as Mermaid diagrams — trust boundaries, the approval state machine, auth/authz flow, and the race-condition sequence are the ones worth seeing visually.

**1. Trust boundaries & data flow**

```mermaid
flowchart TB
    subgraph untrusted["🔴 UNTRUSTED ZONE"]
        Agent["AI Agent (LLM)"]
        Client["MCP Client"]
        Agent -->|tool call| Client
    end

    subgraph boundary1["Trust Boundary 1: MCP Ingress"]
        Ingress["Transport handling<br/>Correlation ID<br/>Rate limiting<br/>Schema validation"]
    end

    subgraph boundary2["Trust Boundary 2: Auth"]
        Auth["Verify OIDC token<br/>Derive user_id from TOKEN only"]
    end

    subgraph app["🟡 APPLICATION LAYER (semi-trusted, server-controlled)"]
        Policy["Policy / Authorization Engine"]
        Proposal["Proposal Engine<br/>(builds exact diff)"]
        Human["🟢 Human Approval Gate<br/>(out-of-band, agent cannot satisfy)"]
        Reauth["Re-Authorization<br/>+ Version Check"]
        Service["Service Layer<br/>(transactions)"]
    end

    subgraph db["🟢 TRUSTED ZONE"]
        Repo["Repository / DB Access<br/>(parameterized, versioned)"]
        Audit["Append-only Audit Log"]
        PG[("PostgreSQL<br/>least-privilege role, RLS")]
    end

    Client -->|MCP transport| Ingress
    Ingress --> Auth
    Auth --> Policy
    Policy --> Proposal
    Proposal --> Human
    Human --> Reauth
    Reauth --> Service
    Service --> Repo
    Repo --> PG
    Service --> Audit
    Audit --> PG

    style untrusted fill:#3a1414,stroke:#c0392b
    style app fill:#3a3414,stroke:#d4ac0d
    style db fill:#143a1e,stroke:#27ae60
    style boundary1 fill:#2a2a2a,stroke:#888
    style boundary2 fill:#2a2a2a,stroke:#888
```

**2. Approval state machine**

```mermaid
stateDiagram-v2
    [*] --> PROPOSED
    PROPOSED --> REJECTED_BY_POLICY: authz fails
    PROPOSED --> PENDING_APPROVAL: authz passes,<br/>proposal persisted

    PENDING_APPROVAL --> EXPIRED: timeout
    PENDING_APPROVAL --> DENIED: human clicks Deny
    PENDING_APPROVAL --> APPROVED: human clicks Approve<br/>(trusted channel only)

    APPROVED --> STALE_REJECTED: re-authz OR version<br/>check fails at execution
    APPROVED --> EXECUTING: claim succeeds<br/>(conditional UPDATE)

    EXECUTING --> EXECUTION_FAILED: DB error / rollback
    EXECUTING --> EXECUTED: transaction commits

    REJECTED_BY_POLICY --> [*]
    EXPIRED --> [*]
    DENIED --> [*]
    STALE_REJECTED --> [*]
    EXECUTION_FAILED --> [*]
    EXECUTED --> [*]

    note right of PENDING_APPROVAL
        Only state where a human
        decision is awaited.
        No agent action moves
        this forward.
    end note

    note right of APPROVED
        NOT the same as executed.
        Still requires re-authorization
        + version match before write.
    end note
```

**3. Authentication + authorization flow (per request)**

```mermaid
sequenceDiagram
    participant A as AI Agent
    participant M as MCP Server
    participant IdP as Identity Provider
    participant DB as Database

    A->>M: propose_edit(resourceId, fieldDiffs)
    M->>M: Verify token signature, exp, aud
    alt token invalid/missing
        M-->>A: DENY (AUTH_FAILED) — fail closed
    end
    M->>IdP: (verify against JWKS)
    IdP-->>M: valid, sub = user_123
    M->>DB: Load resource by id (authoritative)
    DB-->>M: resource { owner_id, version }
    M->>M: policy.can(user_123, EDIT, resource)
    alt not authorized
        M->>DB: write audit event (DENY)
        M-->>A: DENY (NOT_AUTHORIZED)
    else authorized
        M->>DB: INSERT pending_approvals<br/>(snapshot version, exact diff)
        M->>DB: write audit event (ALLOW, PENDING)
        M->>M: notify client (out-of-band UI)
        M-->>A: { proposal_id, summary, expires_at }
        Note over A: Agent has NO tool to<br/>approve or execute this
    end
```

**4. Race condition: the scenario you flagged, and how it's stopped**

```mermaid
sequenceDiagram
    participant Agent
    participant Server
    participant OtherUser as Other User
    participant DB

    Agent->>Server: propose_edit(#123, salary→15L)
    Server->>DB: load #123, version=4
    Server->>DB: create proposal (snapshot_version=4)
    Server-->>Agent: proposal_id, pending

    Note over Server: Human reviews and approves
    Server->>DB: status → APPROVED

    OtherUser->>DB: edits #123 concurrently
    DB-->>DB: #123 version becomes 5

    Agent->>Server: (attempts execution trigger)
    Server->>DB: claim approval (WHERE status='APPROVED')
    Server->>DB: SELECT #123 FOR UPDATE
    DB-->>Server: current version = 5
    Server->>Server: compare snapshot_version(4) vs current(5)
    Note over Server: MISMATCH → abort
    Server->>DB: status → STALE_REJECTED
    Server->>DB: write audit event
    Server-->>Agent: "Record changed since approval —<br/>re-review required"
```

**5. Bulk operation containment**

```mermaid
flowchart LR
    Req["propose_bulk_edit(filter)"] --> Enum["Server enumerates<br/>matching resources NOW"]
    Enum --> Cap{"Count ≤ cap<br/>(e.g. 50)?"}
    Cap -->|No| Split["Reject / require<br/>splitting into batches"]
    Cap -->|Yes| PerRow["Authorize EACH row<br/>individually"]
    PerRow --> Excl["Rows failing authz<br/>→ EXCLUDED, shown to approver"]
    PerRow --> Items["Remaining rows →<br/>pending_approval_items<br/>(own version snapshot each)"]
    Items --> Show["Full enumerated list<br/>shown in approval UI<br/>(never just a count)"]
    Show --> Human["Human approves the<br/>exact enumerated set"]
    Human --> Exec["Execute: re-authorize +<br/>re-version-check EACH row<br/>at execution time"]
```

The throughline across all five: the agent's reach ends at "create a proposal." Everything past that — the approve click, the re-check, the write — happens in a zone the agent has no tool to touch.