# Governance Model Specification

This document details the capability model, policy enforcement engine, quota system, and audit requirements for USAGE-compliant substrates.

## 1. Core Principle: Deny by Default

All external actions (tool invocations, data access, resource allocation) default to **DENIED** unless explicitly authorized. The substrate does not consult the agent for authorization decisions; instead, it validates against the capability ledger.

```
Action Request
    ↓
[Substrate Governance Engine]
    ↓
Lookup Capability Token
    ├─ Not found → DENIED
    ├─ Expired → DENIED
    ├─ Constraints violated → DENIED
    └─ Valid and constraints satisfied → ALLOWED
    ↓
[Tool Execution or Action Performed]
```

## 2. Capability Token Structure

A **capability token** is the unit of authorization in USAGE. It grants permission for an agent to invoke a specific tool or perform a specific operation.

```protobuf
message CapabilityToken {
  string id = 1;                           // Unique capability ID (UUID)
  string agent_identity = 2;               // Agent this is granted to
  string resource_or_tool = 3;             // What is being authorized (e.g., "database.query", "slack.post_message")
  string operation = 4;                    // Type of operation (e.g., "SELECT", "DELETE", "POST")
  int64 issued_at = 5;                     // Unix timestamp when issued
  int64 expires_at = 6;                    // Unix timestamp when expires (int64(-1) = never)
  map<string, string> constraints = 7;     // Optional constraints on usage
  bytes signature = 8;                     // Cryptographic signature (HMAC-SHA256)
  string issuer = 9;                       // Who issued this token (policy engine, admin, etc.)
}
```

### 2.1 Capability Constraints

Constraints are key-value pairs that restrict how a capability can be used:

| Constraint Type | Example | Semantics |
|-----------------|---------|-----------|
| **Time Window** | `time_window: "2025-03-15T14:00:00Z to 2025-03-15T15:00:00Z"` | Capability only valid during this time window |
| **Rate Limit** | `rate_limit: "1000 per minute"` | Max invocations per time period |
| **Data Minimization** | `columns: "user_id, email"` (for DB) | Only these columns are readable |
| **Table Allowlist** | `table_allowlist: "users,transactions"` | Only these tables accessible |
| **Row Filter** | `row_filter: "created_at > 2025-01-01"` | Only rows matching this filter |
| **Data Classification** | `min_classification: PUBLIC,max_classification: INTERNAL` | Only access data of these classifications |
| **Geographical** | `region_allowlist: us-west-2,us-east-1` | Only access resources in these regions |
| **Read-Only** | `read_only: true` | Tool can be called but cannot modify state |
| **Require MFA** | `require_mfa: true` | Additional authentication required |

### 2.2 Capability Lifecycle

```
1. ISSUED
   - Policy engine or admin creates capability token
   - Capability is cryptographically signed
   
2. BINDING
   - Substrate receives SetCapabilityRequest
   - Substrate validates signature
   - Substrate binds capability to agent session
   - Capability is now in agent's ledger
   
3. ACTIVE
   - Agent can invoke tool/resource
   - Substrate validates capability before each invocation
   - Constraints are enforced
   
4. CONSTRAINT_VIOLATION (optional)
   - If constraint violated (rate limit, time window), action is denied
   - Violation is logged
   
5. EXPIRATION (optional)
   - If current_time > expires_at, capability is invalid
   - Future invocations denied
   
6. REVOCATION (optional)
   - Admin revokes capability mid-session
   - Immediate effect (no grace period)
   - Future invocations denied
   - Ongoing tool calls complete, but affected
   
7. INVALIDATED
   - Capability no longer valid (expired, revoked, or signature invalid)
   - Cannot be reactivated
```

## 3. Policy Evaluation Pipeline

Every external action follows this evaluation pipeline:

### Step 1: Identity Resolution
```
Agent makes tool call
    ↓
Extract session_id from request
    ↓
Substrate looks up agent identity in session registry
    ↓
If session not found → return ERROR_SESSION_NOT_FOUND
If agent identity cannot be verified → return PERMISSION_DENIED
    ↓
Continue with agent identity [agent_id]
```

### Step 2: Capability Lookup
```
Request: UsageCallTool(session_id, tool_id, operation, arguments)
    ↓
Substrate queries capability ledger:
  SELECT capability FROM ledger 
  WHERE agent_id = [agent_id]
    AND resource_or_tool = [tool_id]
    AND operation = [operation]
    ↓
If no capability found → return PERMISSION_DENIED
If capability.expires_at < now() → return PERMISSION_DENIED
If capability.signature invalid → return PERMISSION_DENIED
    ↓
Continue with capability token
```

### Step 3: Constraint Validation
```
For each constraint in capability.constraints:
  - Time window: verify current_time within window
  - Rate limit: check invocation count in time window
  - Data minimization: verify arguments don't access unauthorized fields
  - Table allowlist: verify requested table is in allowlist
  - Row filter: verify arguments satisfy filter predicate
  - Data classification: verify data being accessed has authorized classification
  - Geographical: verify tool endpoint is in authorized region
  - Read-only: verify operation is not a write
    ↓
If any constraint violated → return FAILED_PRECONDITION
    ↓
Continue to execution
```

### Step 4: Budget and Resource Checks
```
Check token budget:
  - if agent.tokens_consumed >= agent.token_budget → return RESOURCE_EXHAUSTED
  - if tool_call would exceed budget → return RESOURCE_EXHAUSTED
    ↓
Check tool depth:
  - if current_tool_call_depth >= max_tool_call_depth → return RESOURCE_EXHAUSTED
    ↓
Check rate limits:
  - if agent has exceeded global rate limit → return RESOURCE_EXHAUSTED
    ↓
All checks pass → continue to execution
```

### Step 5: Tool Execution
```
Invoke actual tool with:
  - Timeout enforcement (from tool definition)
  - Idempotency validation (if idempotency_key provided)
  - Egress control (tool only reaches authorized endpoints)
    ↓
Tool returns result (or error)
    ↓
Continue to output scrubbing
```

### Step 6: Output Scrubbing
```
Apply redaction transforms:
  - Detect PII (email, phone, SSN, credit card)
  - Detect secrets (API keys, tokens, passwords)
  - Detect malware signatures
  - Redact or filter unauthorized data
    ↓
Return scrubbed result to agent
```

### Step 7: Audit and Telemetry
```
Record decision:
  - timestamp (nanosecond precision)
  - session_id
  - tool_id and operation
  - agent_identity
  - capability_token_id
  - decision (ALLOWED/DENIED)
  - constraint_violations (if any)
  - tokens_consumed
  - execution_latency
  - request_size_bytes
  - response_size_bytes
  - policy_bundle_version
    ↓
Emit to audit sink (CloudTrail, Datadog, syslog)
    ↓
Emit metrics (Prometheus, CloudWatch)
```

## 4. Quota System

USAGE manages three types of quotas:

### 4.1 Token Budget
```protobuf
message TokenQuota {
  uint64 max_tokens = 1;              // Hard limit on tokens
  uint64 tokens_consumed = 2;         // Running total consumed
  uint64 tokens_remaining = 3;        // Computed: max_tokens - tokens_consumed
  uint64 warn_threshold = 4;          // Warning at X% consumed
  uint64 soft_limit = 5;              // Soft limit (advisory, not enforced)
}
```

**Semantics**:
- Token count is deterministic (same prompt → same token count)
- Token consumption is irreversible (no refunds)
- Token budget is monotonic across checkpoint-restore
- When `tokens_consumed >= max_tokens`, agent transitions to TERMINATED
- Substrate tracks token consumption per tool call and per LLM inference

### 4.2 Tool Call Depth Quota
```protobuf
message ToolCallDepthQuota {
  int32 max_depth = 1;                // Maximum nesting level
  int32 current_depth = 2;            // Current nesting level
  map<string, int32> per_tool_limits = 3;  // Optional per-tool limits
}
```

**Semantics**:
- Incremented when agent calls a tool
- Decremented when tool call returns
- If `current_depth >= max_depth`, new tool calls denied
- Substrate tracks depth to prevent infinite recursion or exponential explosion

### 4.3 Attention Window Quota
```protobuf
message AttentionWindowQuota {
  repeated AttentionWindowSlice active_slices = 1;
  int64 total_tokens_in_window = 2;
  int32 max_simultaneous_slices = 3;
}

message AttentionWindowSlice {
  int32 start_index = 1;              // Start position in context
  int32 end_index = 2;                // End position in context
  string data_classification = 3;     // Data classification of this slice
  bool is_active = 4;                 // Currently in agent's view
}
```

**Semantics**:
- Agent can only reference context indices within active_slices
- Before retrieving context chunk at index N, substrate verifies N is in some active_slice
- Slices can be dynamically enabled/disabled via governance policy changes
- Used to prevent agent from accessing unauthorized portions of context

## 5. Policy Enforcement Engine

### 5.1 OPA (Open Policy Agent) Integration

Substrate MUST integrate with OPA or similar policy language for flexible governance:

```rego
# Example OPA policy rule
package agent_governance

allow_tool_call {
  # Agent must have matching capability
  capability_token := data.capabilities[input.session_id][input.tool_id]
  capability_token.operation == input.operation
  
  # Capability must not be expired
  capability_token.expires_at > now
  
  # All constraints must be satisfied
  constraint_satisfied[constraint] for constraint in capability_token.constraints
}

allow_database_access {
  # Only SELECT allowed, no DELETE
  input.operation == "SELECT"
  
  # Only from certain tables
  input.table in ["users", "transactions"]
  
  # Only recent data
  input.filter_predicate contains "created_at > 2025-03-01"
}

constraint_satisfied[constraint] {
  constraint := input.capability.constraints[_]
  
  # Time window constraint
  constraint.type == "time_window"
  now >= constraint.start_time
  now <= constraint.end_time
}

constraint_satisfied[constraint] {
  constraint := input.capability.constraints[_]
  
  # Rate limit constraint
  constraint.type == "rate_limit"
  count(input.recent_invocations) < constraint.max_per_minute
}
```

### 5.2 Policy Evaluation Latency

Policy evaluation MUST be fast (< 50ms typical, < 100ms p99):
- Policy bundle is pre-loaded into substrate memory
- Capability lookups use indexed queries (hash tables, not full scans)
- Constraint validation is parallelizable (check each constraint independently)
- Results are cached (same request → same decision, if policies unchanged)

## 6. Audit and Compliance

### 6.1 Mandatory Audit Fields

Every governance decision MUST log:

```json
{
  "timestamp": "2025-03-15T14:23:47.123456Z",
  "session_id": "ag-12345678-1234-1234-1234-123456789012",
  "agent_identity": "agent_service_batch_processor",
  "tool_id": "database.query",
  "operation": "SELECT",
  "capability_id": "cap-abcdef123456",
  "decision": "ALLOWED",
  "decision_latency_ms": 12,
  "constraint_violations": [],
  "tokens_consumed": 1500,
  "tokens_remaining": 48500,
  "request_size_bytes": 256,
  "response_size_bytes": 4096,
  "policy_bundle_version": "v2.1.5",
  "policy_engine": "opa",
  "signature": "hmac-sha256:...",
  "audit_trace_id": "trace-xyz"
}
```

### 6.2 Audit Sink Requirements

Audit logs MUST be sent to an **immutable, tamper-evident sink**:

- **Local syslog**: forwarded to central syslog server
- **Cloud providers**: CloudTrail (AWS), Cloud Logging (GCP), Azure Monitor
- **Third-party**: Datadog, Splunk, Elastic, New Relic
- **Custom**: SIEM system with message signing

**Requirements**:
- Logs are append-only (no deletion, only archival)
- Logs are cryptographically signed (HMAC or public-key signature)
- Logs cannot be tampered with without invalidating signature
- Logs support compliance queries (e.g., "all tool invocations by agent X in March 2025")

### 6.3 Compliance Reporting

Substrate MUST support queries for:
- **Tool invocation audit trail**: "Show all database queries by agent X between 2025-03-01 and 2025-03-15"
- **Permission grant audit trail**: "Show all capability grants/revocations for agent Y"
- **Policy violation detection**: "Show all attempted tool calls that were denied by policy"
- **Data access audit trail**: "Show all reads of column user_email by any agent"
- **Token consumption report**: "Show token budget usage by agent, by month"

## 7. Idempotency and Retry Semantics

### 7.1 Idempotency Keys

Tool calls SHOULD include an idempotency key:

```protobuf
message ToolRequest {
  string session_id = 1;
  string tool_id = 2;
  string operation = 3;
  oneof arguments { ... }
  string idempotency_key = 6;         // Optional, format: UUID or hash
  int32 timeout_seconds = 7;
}
```

**Semantics**:
- Substrate remembers (tool_id, idempotency_key) → result
- If same idempotency_key seen again, return cached result (don't re-execute)
- Idempotency cache TTL: 24 hours (configurable)
- Agent is responsible for generating unique keys (use UUID or hash of arguments)

### 7.2 Retry with Policy Context

When agent retries a failed tool call:

```
Retry Attempt N
    ↓
Agent provides same idempotency_key
    ↓
Substrate checks idempotency cache:
  - If key found and result available → return cached result
  - If key found but call still in progress → wait for completion
  - If key not found → evaluate policy again, re-execute
    ↓
Policy evaluation at retry time:
  - Capability may have been revoked since first attempt
  - Constraints may have changed
  - Rate limits may have reset
  - Token budget may have changed
    ↓
Substrate makes fresh policy decision
```

**Important**: Retrying does NOT preserve old policy context. If governance policy changed between attempts, new attempt is evaluated against new policy.

## 8. Revocation Semantics

When a capability is revoked:

```
Revoke Capability cap-xxx
    ↓
Substrate removes cap-xxx from agent's ledger
    ↓
Immediate effect (no grace period)
    ↓
If agent is currently executing tool with cap-xxx:
  - Tool call completes (no interruption)
  - Revocation takes effect on next tool call
    ↓
Next tool call with cap-xxx:
  - Capability lookup fails
  - Return PERMISSION_DENIED
  - Tool call not executed
  - Revocation is logged for audit trail
```

## 9. Example: Full Governance Evaluation

```
Scenario: Agent requests to DELETE user from database

Agent makes request:
  UsageCallTool(
    session_id="ag-12345",
    tool_id="database.query",
    operation="DELETE",
    arguments_json='{"table": "users", "user_id": "usr-999"}'
  )

Substrate evaluation:

Step 1: Identity Resolution
  → Agent identity: "batch_processor"
  ✓ Session found, identity verified

Step 2: Capability Lookup
  → Query: SELECT FROM capabilities 
           WHERE agent="batch_processor" AND tool="database.query"
  → Found: cap-db-001 (operation=SELECT, not DELETE)
  → Result: PERMISSION_DENIED
  
Decision: DENIED (operation SELECT != requested DELETE)

Step 3: Audit Log
  {
    "timestamp": "2025-03-15T14:23:47.123456Z",
    "session_id": "ag-12345",
    "tool_id": "database.query",
    "operation": "DELETE",
    "decision": "DENIED",
    "reason": "Agent has capability for SELECT only, not DELETE",
    "capability_id": "cap-db-001",
    "signature": "..."
  }

Response to Agent:
  ToolResponse(
    success=false,
    error_message="PERMISSION_DENIED: Agent does not have DELETE capability"
  )
```
