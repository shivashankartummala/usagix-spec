# USAGE Core Architecture Specification

## 1. Scope
USAGE standardizes the contract between autonomous cognitive processes and execution substrates. It is substrate-agnostic and does not prescribe model providers, prompting frameworks, or orchestration SDKs.

## 2. Definitive Scope Trigger (Operational Binary)
USAGE classification is binary and programmatic. Behavioral, philosophical, or framework-based definitions of agency are out of scope for classification decisions.

A workload is unconditionally classified as an AI Agent under USAGE when both conditions are true:
- Inference Core Invocation: the workload invokes one or more foundation model or LLM calls to determine state or control flow.
- Peripheral Access Capabilities: the workload can invoke external tools, databases, web APIs, or native host system calls.

Any script, binary, service, job, or active execution thread meeting both conditions is in-scope for USAGE runtime enforcement.

## 3. Absolute Boundary Condition
No exception exists for code size, framework usage, or runtime longevity.

Rule of inclusion:
- A minimal script that calls an LLM endpoint and executes shell, filesystem, or API actions is semantically equivalent (for governance scope) to a multi-agent orchestration tree.

Rule of boundary enforcement:
- Upon meeting the scope trigger, the workload MUST be enclosed by the USAGE runtime boundary.
- The workload MUST relinquish direct external network/filesystem side-effect pathways except substrate-mediated pathways.
- The workload MUST authenticate with a distinct cryptographically verifiable workload identity.
- The workload MUST route external actions through the substrate tool-proxy path.
- The workload MUST submit to real-time token accounting and quota enforcement.

Non-compliance at substrate enforcement points MUST result in terminal execution signaling (for example `SIG_AGENT_TERMINATE`).

## 4. Normative Language
The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be interpreted as described in RFC 2119.

## 5. Layered Model
- Layer 4 (Cognitive Application): intent formation, prompt orchestration, tool intent emission.
- Layer 3 (Governance and Aspect): policy checks, budget accounting, redaction, trace emission.
- Layer 2 (Runtime and Execution): session lifecycle, signal handling, checkpoint/resume.
- Layer 1 (SAL): substrate adaptation for compute, network, storage, and isolation primitives.

## 6. Trust Boundaries
- The cognitive container is untrusted by default.
- Governance and policy enforcement MUST execute outside the cognitive trust boundary.
- External side effects MUST be mediated through USAGE system calls.

## 7. Portability Contract
A USAGE-compliant cognitive workload MUST execute against any substrate implementing:
- ASI gRPC service behavior
- Lifecycle semantics
- Error model and result determinism constraints
- Minimum observability semantics

## 8. Determinism Requirements
- Lifecycle transitions MUST be explicit and auditable.
- Replayed checkpoints SHOULD reproduce equivalent policy and signal outcomes under identical policy snapshots and tool responses.

## 9. Compatibility
- Wire compatibility follows protobuf backward-compatible evolution.
- Behavioral compatibility requires preserving syscall semantics unless a major version bump is declared.

## 10. Protocol Stack Architecture

USAGE defines a 4-layer protocol stack with clear responsibility boundaries:

```
┌────────────────────────────────────────┐
│  Layer 4: Cognitive Application         │
│  (Agent code, prompt logic, tool reqs)  │
└────────────────────┬───────────────────┘
                     │ Intent/Requests
┌────────────────────▼───────────────────┐
│  Layer 3: Governance & Aspect           │
│  (Capability checks, budget, audit)     │
└────────────────────┬───────────────────┘
                     │ Validated Requests
┌────────────────────▼───────────────────┐
│  Layer 2: Runtime & Execution           │
│  (Session lifecycle, checkpoint, mem)   │
└────────────────────┬───────────────────┘
                     │ Protocol Messages
┌────────────────────▼───────────────────┐
│  Layer 1: Substrate Adaptation Layer    │
│  (Compute, network, storage, isolation) │
└────────────────────────────────────────┘
```

**Layer 4 (Cognitive Application)**: The agent code itself—LLM calls, tool selection logic, internal state management. This layer is **untrusted** and run inside a security boundary.

**Layer 3 (Governance & Aspect)**: Policy enforcement, capability validation, token/budget accounting, PII/secret detection, audit logging. This layer runs **outside** the cognitive container and medifies all requests from Layer 4.

**Layer 2 (Runtime & Execution)**: Session lifecycle management (PENDING→ACTIVE→THINKING→PAUSED→TERMINATED), checkpoint/restore mechanics, virtual memory (L1/L2/L3) management, signal handling.

**Layer 1 (SAL - Substrate Adaptation Layer)**: Actual hardware/cloud primitives—Kubernetes pods, container isolation, persistent storage, network access, sidecar networking. Maps USAGE abstractions to concrete infrastructure.

## 11. Security Model Overview

### 11.1 Defense Layers

USAGE security operates via **multiple defense layers**, each enforcing constraints at different points:

**Layer A: Identity & Authentication**
- Agent authenticates with unique cryptographic identity (not inherited from container/process)
- Substrate verifies identity before any capability is granted
- Identity immutable during session lifetime

**Layer B: Capability-Based Mediation**
- Before ANY external action (tool call, data access, resource allocation), substrate checks capability ledger
- Capability grant is explicit, granular, and time-bounded
- Deny-by-default: absence of explicit grant = denial

**Layer C: Budget & Resource Enforcement**
- Token budget is hard limit, enforced at call boundary
- Tool depth bounded by max_tool_call_depth counter
- Attention window indices validated before context retrieval
- Budget deductions are atomic and irreversible

**Layer D: Attestation & Audit**
- Every capability check result logged with timestamp, identity, decision metadata
- Logs cryptographically signed by substrate
- Logs immutable and tamper-evident

### 11.2 Threat Model

USAGE assumes the following threats:

| Threat | Agent Can... | Substrate Prevents |
|--------|-------------|-------------------|
| **Prompt Injection** | Craft tool requests to bypass policy | Capability ledger checked before tool execution |
| **Jailbreak** | Convince LLM to ignore constraints | Token budget and tool depth hard limits enforced by substrate |
| **Data Exfiltration** | Request data outside authorized scope | Attention window indices validated; access denied for unauthorized indices |
| **Privilege Escalation** | Claim higher capability than granted | Capability ledger is source of truth; substrate does not consult agent |
| **Cost Overrun** | Retry loops or deep nesting | Token budget and tool depth hard limits prevent unbounded consumption |
| **Denial of Service** | Saturate substrate resources | Rate limits, quota enforcement, per-agent resource isolation |
| **Lateral Movement** | Use one tool to access another's resources | Tool authorization checked per-tool; no transitive trust |
| **Time-Decay Attack** | Assume policies unchanged since checkpoint | Policies re-validated on resume; governance compliance checked |

USAGE **does not defend against**:
- Inference engine vulnerabilities (e.g., backdoored model weights)
- Tool implementation bugs (substrate assumes tools are correctly written)
- Physical infrastructure compromise (substrate assumes substrate code is trustworthy)

## 12. Governance Model Overview

### 12.1 Capability Tokens

A **capability token** is a cryptographically signed grant of permission to invoke a specific tool or access a specific resource:

```protobuf
message CapabilityToken {
  string id = 1;                          // Unique capability ID
  string agent_identity = 2;              // Who can use this capability
  string resource_or_tool = 3;            // What tool or resource
  string operation = 4;                   // What operation (e.g., SELECT, DELETE)
  google.protobuf.Timestamp issued_at = 5;
  google.protobuf.Timestamp expires_at = 6;
  map<string, string> constraints = 7;    // Optional constraints
  string signature = 8;                   // Cryptographic signature
}
```

Example constraints:
- `data_minimization: "SELECT only columns: user_id, email"`
- `time_window: "2025-03-15T14:00:00Z to 2025-03-15T15:00:00Z"`
- `rate_limit: "max 1000 requests per minute"`
- `data_classification: "handle only PUBLIC and INTERNAL, not CONFIDENTIAL"`

### 12.2 Capability Lifecycle

1. **Issuance**: Administrator or policy engine creates capability token
2. **Binding**: Substrate binds capability to agent identity
3. **Validation**: Before tool invocation, substrate checks:
   - Token exists in agent's ledger
   - Token is not expired
   - Constraints are satisfied (time window, data minimization, rate limits, etc.)
4. **Revocation**: Administrator revokes token (immediate effect)
5. **Re-validation on Resume**: After checkpoint restore, substrate re-validates all open capabilities

## 13. Token Accounting Model

Token accounting is a **mandatory, first-class subsystem**:

```
When Agent Created:
  agent.token_budget = quota.max_tokens
  agent.tokens_consumed = 0

When LLM Call Made:
  prompt_tokens = len(tokenize(prompt))
  completion_tokens = len(tokenize(completion))
  agent.tokens_consumed += prompt_tokens + completion_tokens
  if agent.tokens_consumed >= agent.token_budget:
    agent.state = TERMINATED
    reason = "Token budget exhausted"
```

**Semantics**:
- Token counting is deterministic (same prompt → same token count every time)
- Tokens consumed are irreversible (cannot "unspend" tokens)
- Token budget is monotonic across checkpoint-restore (tokens_consumed never decreases)
- Budget exhaustion triggers graceful termination, not abrupt crash

## 14. Implementation Patterns for Substrate Builders

### 14.1 Sidecar Pattern (Recommended)

For untrusted cognitive containers, use a **sidecar architecture**:

```
┌─────────────────────────────┐
│  Untrusted Agent Container  │
│  (connects to localhost only)│
│  ↓                          │
│  gRPC Client                │
│  (speaks USAGE ASI)         │
└─────────────────────────────┘
         ↕ localhost:50051
┌─────────────────────────────┐
│  Sidecar Proxy Container    │
│  (USAGE Substrate Impl)     │
│  ✓ Capability validation    │
│  ✓ Token accounting         │
│  ✓ Audit logging            │
│  ✓ Tool mediation           │
└────────────┬────────────────┘
         ↕ full network
┌─────────────────────────────┐
│  External Tools/Infrastructure │
└─────────────────────────────┘
```

Agent container cannot directly reach external network (no egress). All external access routes through sidecar, which enforces USAGE constraints.

### 14.2 Checkpoint Implementation

Every checkpoint MUST include:

1. **Standardized Header**: Format version, timestamps, token counts, attention indices, capability tokens, memory tier locators, context hash
2. **Opaque Payload**: Vendor-specific agent state serialization (reference as URI if > 10MB)
3. **Integrity Proof**: SHA-256 hash of payload, signed with substrate key

```protobuf
message CheckpointHeaderAndPayload {
  CheckpointHeader header = 1;
  bytes opaque_payload = 2;
}

message CheckpointHeader {
  string format_version = 1;              // e.g., "usage/v2"
  int64 checkpoint_timestamp = 2;         // When checkpoint was created
  uint64 total_tokens_consumed = 3;       // Cumulative token count
  repeated int32 active_attention_window_indices = 4;
  int32 current_tool_call_depth = 5;
  repeated string open_capability_tokens = 6;
  map<int32, string> memory_tier_locators = 7;
  string context_hash = 8;                // SHA-256 of full context
  string governance_compliance_metadata = 9;
}
```

### 14.3 Memory Tier Management

Implement bidirectional paging (PageOut and PageIn):

**PageOut (Eviction from L1 to L2/L3)**:
- Triggered when context window reaches capacity
- Eviction is policy-compliant (PII/secrets scrubbed before cold storage)
- Address reference (not full copy) returned to agent
- Metadata logged for audit

**PageIn (Rehydration from L2/L3 to L1)**:
- Agent requests context chunk by address_reference
- Substrate validates governance compliance before retrieval
- Substrate verifies checkpoint integrity
- Full payload fetched and loaded into agent's L1

## 15. Observability and Compliance

USAGE requires the following observability:

### 15.1 Mandatory Logs

Every decision point MUST log:
- Timestamp (nanosecond precision, UTC)
- Agent identity
- Operation attempted (tool name, operation, parameters)
- Capability check result (allowed/denied)
- Decision metadata (which constraint matched, which policy rule applied)
- Signature (cryptographic proof of log integrity)

### 15.2 Audit Trail

All logs flow to an **immutable audit sink** (e.g., CloudTrail, Datadog, syslog with signing):
- Logs are append-only
- Logs are cryptographically signed
- Logs cannot be deleted (only archived)
- Logs support compliance queries (e.g., "show all DELETE operations by Agent-X in March 2025")

## 16. Error Model

USAGE defines standardized error responses:

| Code | Meaning | Agent Recourse |
|------|---------|---|
| **CAPABILITY_DENIED** | Tool/operation not in capability ledger | Impossible; agent cannot retry |
| **BUDGET_EXHAUSTED** | Token budget exhausted | Impossible; agent must terminate |
| **POLICY_VIOLATION** | Operation violates governance policy | Impossible; request denied at substrate |
| **INVALID_CONSTRAINT** | Constraint validation failed (rate limit, time window) | Maybe; depends on constraint type |
| **INFRASTRUCTURE_ERROR** | Tool execution failed (network, database error) | Maybe; substrate provides error details |
| **GOVERNANCE_ERROR** | Policy system error (authorization service down) | No; substrate blocks operation until policy service recovers |

Errors returned to agent MUST NOT leak sensitive information (policy details, other agents' capabilities, etc.).
