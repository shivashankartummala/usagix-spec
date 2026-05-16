# RFC-0001: USAGIX Core Architecture & Trust Domains

**Status**: Implemented (v1.0)  
**Authors**: USAGIX Specification Stewards  
**Created**: 2025-03-15  
**Updated**: 2025-05-16

## Summary

USAGIX defines a substrate-agnostic architecture for safe, auditable execution of autonomous agent workloads through strict trust domain separation. The core principle is that untrusted agent code cannot be trusted to enforce governance constraints if it has direct system access—therefore, all agent side effects must transit through a trusted governance enforcement plane.

## 1. Semantic Definitions

### 1.1 Core Concepts

**Agent**: An autonomous process that:
- Invokes one or more LLM/foundation model calls to determine control flow or state
- Invokes external tools, APIs, databases, or system calls
- May spawn child agents
- Executes under USAGIX substrate governance

**Substrate**: The execution environment hosting agent processes. Must implement:
- Trust domain separation (Cognitive Container vs. Governance Enforcement Plane)
- Agent Substrate Interface (ASI) gRPC protocol
- Capability-based access control
- Immutable audit logging
- Token budget enforcement

**Trust Domain**: A security boundary with distinct privilege levels and isolation guarantees.

**Cognitive Container** (Untrusted Domain A): 
- Runtime environment for agent code
- Isolated from direct system/network access
- Cannot bypass governance controls
- May be compromised or adversarial

**Governance Enforcement Plane** (Trusted Domain B):
- Out-of-process or hardware-enforced boundary
- Validates all agent requests via capability ledger
- Enforces policies, budgets, and quotas
- Maintains immutable audit logs
- Executes under different privilege model than agent

**Capability**: An unforgeable token granting agent permission to invoke a specific tool with bounded scope. Format: cryptographically signed JWT with:
- Subject: agent session ID
- Resource: tool registry name
- Scope: parameter constraints
- TTL: expiration timestamp

### 1.2 Formal Type System

```
Session        = {id: UUID, state: AgentState, parent_pid: PID}
Capability     = {token: JWT, subject: SessionID, resource: ToolName, scope: Constraints, ttl: Timestamp}
CapabilityLedger = Map<SessionID, Set<Capability>>
CheckpointHeader = {format_version, timestamp, tokens_consumed, context_hash, governance_metadata}
Policy         = {version, rules: [PolicyRule], effect: Allow|Deny}
AuditRecord    = {timestamp, session_id, decision: Allow|Deny, policy_version, rationale, latency_ms}
```

## 2. Formal Contracts

### 2.1 Trust Boundary Guarantee

**Theorem (Trust Boundary Isolation)**: If a substrate correctly implements USAGIX trust domain separation, then no agent process can invoke system operations without passing through the Governance Enforcement Plane.

**Proof Sketch**:
1. Agent code runs in Cognitive Container with no socket, filesystem, or inter-process communication (IPC) access to external systems
2. Agent can only issue gRPC calls to the Governance Enforcement Plane (via localhost:50051 or secure boundary)
3. All tool invocations require capability tokens validated by the Plane
4. Therefore, all external effects are mediated by the Plane

**Assumptions**:
- Container runtime correctly enforces process isolation (Linux namespaces, seccomp, etc.)
- gRPC channel is cryptographically authenticated (mTLS)
- Substrate code does not contain vulnerabilities enabling container escape

### 2.2 Governance Decision Contract

**Invariant**: For all tool invocation requests R, the Governance Enforcement Plane executes exactly one of:
- *Allow* (execute tool, log decision, increment token counter)
- *Deny* (return error, log denial, maintain token counter)

**Atomicity**: The decision (allow/deny) and audit log entry are recorded atomically. If the system crashes mid-decision, the agent receives no response and must retry.

**Monotonicity**: Token consumption never decreases. Once N tokens are consumed, total consumed ≥ N.

### 2.3 Capability Revocation Contract

**Guarantee**: If a capability is revoked from the capability ledger, all subsequent invocations of that capability MUST be denied with `PERMISSION_DENIED` error within T seconds (default T = 1 second).

**Latency SLO**: Revocation propagates to all enforcement points within the SLO window. Substrates must implement capability caching with TTL ≤ SLO window.

## 3. Architecture

### 3.1 Trust Domain Separation

```
┌──────────────────────────────────────────────┐
│  Application Layer                           │
│  (Agent Code, LLM Inference, Logic)         │
└──────────────┬───────────────────────────────┘
               │ gRPC (UsagixCallTool, etc.)
        [Trust Boundary - Enforcement]
               │ gRPC (UsagixCallTool, etc.)
┌──────────────▼───────────────────────────────┐
│  Governance Enforcement Plane (Trusted)      │
│  - Capability Validator                      │
│  - Policy Evaluator (OPA)                    │
│  - Token Budget Enforcer                     │
│  - Output Scrubber                           │
│  - Audit Logger                              │
└──────────────┬───────────────────────────────┘
               │ (Direct Access)
┌──────────────▼───────────────────────────────┐
│  Substrate & Infrastructure                  │
│  (Kubernetes, Serverless, WASM, VM)         │
└──────────────────────────────────────────────┘
```

### 3.2 Agent-Substrate Boundary

The Cognitive Container and Governance Enforcement Plane communicate via **ASI (Agent Substrate Interface)**, a gRPC protocol defining system calls:

- `UsagixSpawn`: Initialize agent process
- `UsagixYield`: Checkpoint and suspend
- `UsagixSignal`: Pause/resume/terminate
- `UsagixMemPageOut`: Page memory to L2/L3
- `UsagixMemPageIn`: Restore memory from L2/L3
- `UsagixCallTool`: Invoke external tool

All communication flows through the trust boundary; no direct agent-to-external connections are possible.

## 4. Lifecycle Guarantees

### 4.1 Session Lifecycle

```
[*] → Pending → Active → Thinking ⇄ Paused → Terminated → [*]

Pending: Awaiting identity binding and admission decisions
Active: Identity established, ready to reason
Thinking: In active inference or tool invocation loop
Paused: Yielded (checkpoint or signal), awaiting resume
Terminated: Session closed, resources reclaimed
```

**Guarantee (State Machine Determinism)**: For any given session, if the sequence of inputs (signals, tool responses, policy decisions) is identical, the state transitions MUST be identical.

### 4.2 Termination Guarantee

**Invariant (Guaranteed Termination)**: 
1. Agent process MUST terminate within budget-exhaustion grace period (default: 30 seconds) after token quota is exceeded
2. `SIG_AGENT_TERMINATE` signal delivery is guaranteed within grace period
3. Substrate reclaims all resources within grace period + cleanup overhead

### 4.3 Audit Completeness Guarantee

**Invariant (Complete Audit Trail)**: Every governance decision (allow, deny, error) is recorded atomically with:
- Session ID
- Timestamp
- Decision outcome
- Policy version
- Latency metrics
- Cryptographic signature of record

Audit logs are append-only; no retroactive modification is permitted.

## 5. Mandatory Security Controls

For a substrate to claim USAGIX v1.0 compliance, the following controls MUST be implemented:

1. **Trust Domain Separation**: Agent and Plane in distinct processes/containers
2. **Capability-Based Access Control**: All tool calls gated by capability tokens
3. **Token Budget Enforcement**: Hard quota limit at substrate level
4. **Immutable Audit Logging**: Append-only, cryptographically signed
5. **Output Scrubbing**: PII/secret detection and redaction
6. **Memory Isolation**: Distinct context per session; no cross-contamination

## 6. Backward Compatibility

USAGIX v1.0 is the initial stable version. No prior versions require compatibility.

For future versions:
- Breaking changes require MAJOR version bump (v2.0)
- New features require MINOR version bump (v1.1)
- Bug fixes require PATCH version bump (v1.0.1)

See [spec/versioning.md](../spec/versioning.md) for detailed policy.

## 7. Normative References

- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) — Keywords for use in specs
- POSIX.1-2008 — Process management and IPC primitives
- gRPC specification — Remote procedure call framework
- Protocol Buffers v3 — Serialization format

## 8. Open Questions & Future Work

- **Q1**: Should trust domains be implemented via separate processes, separate containers, or hardware-enforced boundaries? Answer deferred to substrate specifications.
- **Q2**: What cryptographic algorithms are required for capability signing? Answer: TBD in security model RFC.
- **Future**: Hardware attestation integration for TEE-based enforcement (v2.0 candidate)

---

**Document Hash**: `sha256(this_document)`  
**Approvals**:
- [ ] TSC Security Lead
- [ ] TSC Architecture Lead
- [ ] Community Review (2-week period)
