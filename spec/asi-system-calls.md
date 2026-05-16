# ASI System Calls: Detailed Specification

This document provides detailed reference material for each RPC method in the Agent Substrate Interface (ASI). See `proto/usage/v1/asi.proto` for wire format definitions.

## 1. UsageSpawn — Create an Agent Session

### Purpose
Creates a new agent session instance with initial quota, capabilities, and lifecycle state. The new session begins in `PENDING` state and must transition to `ACTIVE` before execution can begin.

### Protobuf Definition
```protobuf
service UsageSubstrate {
  rpc UsageSpawn(SpawnRequest) returns (SpawnResponse);
}

message SpawnRequest {
  string parent_session_id = 1;              // Creator session (may be empty for root)
  string session_name = 2;                   // Human-readable session name
  UsageSpec usage_spec = 3;                  // Quota, capabilities, runtime profile
  string inference_model = 4;                // Model ID (e.g., "claude-opus-4-7")
  map<string, string> environment = 5;       // Initial environment variables
  int32 max_tool_nesting_depth = 6;          // Max nested tool calls (default: 10)
  string namespace_policy = 7;               // Namespace/isolation policy
  ColdStartSchedulingHint cold_start_hint = 8; // Optional cold-start parameters
  
  message ColdStartSchedulingHint {
    google.protobuf.Timestamp target_activation_time = 1;  // When to move to ACTIVE
    string queue_reason = 2;                 // Why this session is queued
  }
}

message SpawnResponse {
  string session_id = 1;                     // Unique session identifier (UUID)
  AgentState state = 2;                      // Initial state (always PENDING)
  int64 spawn_timestamp = 3;                 // Nanosecond UTC timestamp
  bytes initial_checkpoint = 4;              // Minimal checkpoint for persistence
}
```

### Semantics

**Validation**:
- Parent session (if provided) MUST exist and caller MUST have authority to create child
- `usage_spec.max_tokens_quota` MUST be positive and <= parent's remaining quota (if parent exists)
- `inference_model` MUST be recognized by substrate
- Namespace policy MUST be valid (substrate verifies against policy engine)

**Execution**:
1. Substrate allocates unique `session_id` (UUIDv4)
2. Substrate binds `UsageSpec` to session (quota, capabilities, runtime profile)
3. Substrate creates initial checkpoint with zero tokens consumed
4. Session transitions to `PENDING` state (not yet running)
5. If `cold_start_hint` provided, session is queued for later activation

**Return**:
- `session_id`: Caller uses this in all subsequent RPC calls
- `state`: Always `PENDING` on spawn
- `spawn_timestamp`: Server-side timestamp for audit trail
- `initial_checkpoint`: Minimal checkpoint for distributed session tracking

### Example: Spawning an Agent with Cold-Start Scheduling

```python
response = substrate.UsageSpawn(SpawnRequest(
  session_name="batch_processor_001",
  usage_spec=UsageSpec(
    max_tokens_quota=500_000,
    granted_capabilities={
      "database.query": CapabilityToken(
        resource_or_tool="database.query",
        operation="SELECT",
        constraints={"table_allowlist": "users,transactions"}
      ),
      "slack.post_message": CapabilityToken(
        resource_or_tool="slack.post_message",
        operation="POST",
        constraints={"channel": "#alerts-only"}
      )
    }
  ),
  inference_model="claude-opus-4-7",
  max_tool_nesting_depth=5,
  cold_start_hint=ColdStartSchedulingHint(
    target_activation_time=Timestamp(seconds=1710432600),  # 2:30 PM
    queue_reason="Batch window opens at 2:30 PM"
  )
))
# Returns: session_id="ag-uuid-xxx", state=PENDING
```

---

## 2. UsageSetState — Transition State Explicitly

### Purpose
Requests a state transition (e.g., PENDING→ACTIVE, ACTIVE→PAUSED, THINKING→PAUSED). Substrate validates that the transition is legal and that all preconditions are met.

### Protobuf Definition
```protobuf
message SetStateRequest {
  string session_id = 1;
  AgentState target_state = 2;               // Desired state
  string reason = 3;                         // Human-readable reason
}

message SetStateResponse {
  AgentState new_state = 1;
  int64 state_change_timestamp = 2;
  string decision_explanation = 3;           // Why state transition succeeded/failed
}

enum AgentState {
  STATE_UNSPECIFIED = 0;
  PENDING = 1;                               // Awaiting activation
  ACTIVE = 2;                                // Ready to run (waiting for THINKING request)
  THINKING = 3;                              // Currently executing LLM inference
  PAUSED = 4;                                // Suspended (checkpoint present)
  TERMINATED = 5;                            // Execution complete or failed
}
```

### Semantics

**Valid Transitions**:
```
PENDING  → ACTIVE (normal activation)
PENDING  → TERMINATED (reject without running)
ACTIVE   → THINKING (normal execution entry)
ACTIVE   → PAUSED (cold-start pause, emergency halt)
THINKING → ACTIVE (LLM call complete, awaiting next decision)
THINKING → PAUSED (yield with checkpoint)
THINKING → TERMINATED (budget exhausted, critical error)
PAUSED   → ACTIVE (resume after pause)
PAUSED   → TERMINATED (graceful shutdown from paused state)
ANY      → TERMINATED (terminal state, unreversible)
```

**Illegal Transitions** (substrate rejects):
- PAUSED → THINKING (must go PAUSED→ACTIVE→THINKING)
- ACTIVE → ACTIVE (no-op, return current state)
- TERMINATED → anything (no exit from terminal state)

**Pre-checks before transition**:
- If PENDING→ACTIVE: verify no pending exceptions, governance policies validated
- If ACTIVE→PAUSED: verify checkpoint can be created, open capabilities are revokable
- If PAUSED→ACTIVE: re-validate governance policies, verify checkpoint integrity
- If ANY→TERMINATED: graceful shutdown, close all open resources

### Example

```python
# Resume a paused agent after policy change
response = substrate.UsageSetState(SetStateRequest(
  session_id="ag-uuid-xxx",
  target_state=AgentState.ACTIVE,
  reason="Policy validation complete, batch window open"
))
# Returns: new_state=ACTIVE, decision_explanation="..."
```

---

## 3. UsageCallTool — Mediated Tool Invocation

### Purpose
Agent requests execution of an external tool (API call, database query, system command, etc.). Substrate validates capability, enforces constraints, executes tool, scrubs output, and returns result.

### Protobuf Definition
```protobuf
message ToolRequest {
  string session_id = 1;
  string tool_id = 2;                        // Tool name/identifier
  string operation = 3;                      // Operation type (e.g., "SELECT", "POST")
  oneof arguments {
    string arguments_json = 4;               // For small payloads (< 64 KB)
    string arguments_reference = 5;          // For large payloads (S3 URI, etc.)
  }
  string idempotency_key = 6;                // Optional idempotency key
  int32 timeout_seconds = 7;                 // Request timeout (default: 30s)
}

message ToolResponse {
  string tool_id = 1;
  int64 execution_start_timestamp = 2;       // When tool execution began
  int64 execution_end_timestamp = 3;         // When tool execution ended
  oneof result {
    string result_json = 4;                  // For small payloads (< 64 KB)
    string result_reference = 5;             // For large payloads (S3 URI, etc.)
  }
  bool success = 6;
  string error_message = 7;                  // If success=false
  ToolExecutionMetadata execution_metadata = 8;
  string policy_decision_trace_id = 9;       // Audit trail ID
  
  message ToolExecutionMetadata {
    string capability_token_used = 1;
    int32 tokens_consumed = 2;               // Tokens used by this tool call
    map<string, string> constraints_applied = 3;
    string policy_enforcement_layer = 4;     // Which policy rule applied
  }
}
```

### Semantics

**Step 1: Capability Validation** (before execution)
- Substrate checks agent's capability ledger
- If `tool_id` + `operation` not in ledger → return `CAPABILITY_DENIED` immediately
- If capability exists but expired → return `CAPABILITY_DENIED`
- If capability has constraints (rate limit, time window, data minimization), validate against request

**Step 2: Constraint Validation** (before execution)
- Rate limit: if agent has exceeded max requests/minute → return `RESOURCE_EXHAUSTED`
- Time window: if current time outside allowed window → return `INVALID_CONSTRAINT`
- Data minimization: if arguments request unauthorized data → return `CAPABILITY_DENIED`
- Tool depth: if current tool_call_depth >= max_tool_nesting_depth → return `RESOURCE_EXHAUSTED`

**Step 3: Tool Execution** (substrate calls actual tool)
- Increment `current_tool_call_depth`
- Execute tool with timeout enforcement
- Catch all exceptions and return as tool error (not substrate error)

**Step 4: Output Scrubbing** (before returning to agent)
- Detect PII (email addresses, phone numbers, SSNs, credit card numbers)
- Detect secrets (API keys, passwords, tokens)
- Detect malicious patterns (known exploit code)
- If sensitive data detected: return redacted response + audit log warning
- Substrate MUST NOT crash on malicious output (assume tools are adversarial)

**Step 5: Return Result**
- Return `ToolResponse` with execution metadata and policy trace
- `policy_decision_trace_id` links this to audit logs for compliance

### Example: Database Query with Constraints

```python
# Agent requests to query user database
response = substrate.UsageCallTool(ToolRequest(
  session_id="ag-uuid-xxx",
  tool_id="database.query",
  operation="SELECT",
  arguments_json=json.dumps({
    "query": "SELECT user_id, email FROM users WHERE created_at > '2025-03-01'",
    "limit": 1000
  }),
  timeout_seconds=30
))

# Substrate validation:
# 1. Check capability: agent has "database.query" with operation "SELECT"
# 2. Check constraints: capability has constraint "table_allowlist: users,transactions"
#    Agent requested "users" table → allowed
# 3. Check time window: current time within capability window → allowed
# 4. Execute query via actual database driver
# 5. Scrub output for sensitive data (emails detected but allowed per capability)
# 6. Return response with execution_metadata and policy_decision_trace_id

if response.success:
  print(f"Query executed in {response.execution_end_timestamp - response.execution_start_timestamp}ms")
else:
  print(f"Query failed: {response.error_message}")
```

---

## 4. UsageMemPageOut — Evict Context to Lower Tier

### Purpose
Agent requests demotion of context pages from L1 (active context) to L2/L3 (warm/cold storage). Used when context grows large and agent needs to free space. Substrate maintains references for later retrieval (PageIn).

### Protobuf Definition
```protobuf
message PageOutRequest {
  string session_id = 1;
  repeated PageReference pages_to_evict = 2;
  MemoryTier target_tier = 3;                // L2 (warm cache) or L3 (cold storage)
  
  message PageReference {
    int32 page_id = 1;
    int32 size_bytes = 2;
    string data_classification = 3;          // PUBLIC, INTERNAL, CONFIDENTIAL
    bool contains_pii = 4;
  }
}

message PageOutResponse {
  string session_id = 1;
  repeated string page_references = 2;       // URIs for later retrieval (e.g., s3://...)
  int64 eviction_timestamp = 3;
  string encryption_key_id = 4;              // If encrypted
  
  enum MemoryTier {
    TIER_UNSPECIFIED = 0;
    TIER_L1_CONTEXT_WINDOW = 1;              // Active context (in process memory)
    TIER_L2_WARM_CACHE = 2;                  // Redis, Memcached (ms latency)
    TIER_L3_COLD_STORAGE = 3;                // S3, GCS (100ms latency)
  }
}
```

### Semantics

**Pre-eviction checks**:
- Verify pages are not currently in use
- If pages contain PII/CONFIDENTIAL data: encrypt before placing in cold storage
- Log eviction for audit trail

**Eviction process**:
1. Serialize pages (agent's representation)
2. Apply encryption if data_classification requires it
3. Write to target_tier (L2 or L3)
4. Return references (e.g., Redis key, S3 path)
5. Agent can now store these references in its context

**Return references**:
- L2 references are keys (e.g., `redis://cache.internal:6379/agent-ag-xxx/page-123`)
- L3 references are URIs (e.g., `s3://agent-checkpoints/ag-xxx/page-123.bin`)
- References are opaque to agent; agent treats them as black-box handles

### Example

```python
# Agent's context is getting full, evict old conversation history to L2
response = substrate.UsageMemPageOut(PageOutRequest(
  session_id="ag-uuid-xxx",
  pages_to_evict=[
    PageReference(
      page_id=1,
      size_bytes=512_000,
      data_classification="INTERNAL",
      contains_pii=False
    )
  ],
  target_tier=PageOutResponse.MemoryTier.TIER_L2_WARM_CACHE
))

# Returns page_references=["redis://cache.internal:6379/ag-xxx/page-1"]
# Agent can now reference this later via UsageMemPageIn
```

---

## 5. UsageMemPageIn — Retrieve Context from Lower Tier

### Purpose
Agent requests retrieval of a previously evicted context page from L2/L3 back into L1. Substrate validates governance compliance before rehydrating.

### Protobuf Definition
```protobuf
message PageInRequest {
  string session_id = 1;
  string page_reference = 2;                 // URI returned by PageOut
  bool decryption_required = 3;
  string decryption_key_id = 4;              // If encrypted
}

message PageInResponse {
  string session_id = 1;
  bytes page_data = 2;                       // Full page content
  int64 retrieval_timestamp = 3;
  string policy_revalidation_status = 4;     // APPROVED, DENIED, DEGRADED
  string governance_decision_trace_id = 5;   // Audit trail
}
```

### Semantics

**Pre-retrieval validation**:
- Check that page_reference is valid (exists in cache/storage)
- Check that page was evicted by this session (prevent cross-session access)
- Verify current governance policy permits agent to access this data

**Governance re-validation**:
When retrieving from cold storage, substrate MUST:
1. Check if policies have changed since eviction
2. If data was classified as CONFIDENTIAL and agent no longer has CONFIDENTIAL access → return DEGRADED (allow retrieval but mark as read-only)
3. If data was classified as CONFIDENTIAL and agent lost access entirely → return DENIED (refuse retrieval)
4. Otherwise return APPROVED

**Retrieval process**:
1. Fetch page_data from L2/L3
2. Decrypt if encryption_key_id provided
3. Validate integrity (check SHA-256 hash if available)
4. Return full page_data to agent
5. Log retrieval for audit trail

### Example

```python
# Agent retrieves previously evicted context
response = substrate.UsageMemPageIn(PageInRequest(
  session_id="ag-uuid-xxx",
  page_reference="redis://cache.internal:6379/ag-xxx/page-1",
  decryption_required=False
))

# Substrate revalidates policies:
# - Page was evicted 3 hours ago when agent had INTERNAL access
# - Governance policies changed 1 hour ago: agent lost INTERNAL access
# - Response: policy_revalidation_status="DENIED"
# - Error explanation: "Data classification changed; agent no longer authorized"

if response.policy_revalidation_status == "APPROVED":
  context_page = response.page_data
elif response.policy_revalidation_status == "DENIED":
  raise GovernanceError("Cannot rehydrate: policies revoked access")
elif response.policy_revalidation_status == "DEGRADED":
  # Can retrieve but data is now read-only
  context_page = response.page_data
```

---

## 6. UsageYield — Cooperative Pause with Checkpoint

### Purpose
Agent explicitly yields control and requests pause with checkpoint. Used for long-running operations that need to suspend and resume later.

### Protobuf Definition
```protobuf
message YieldRequest {
  string session_id = 1;
  string checkpoint_hint = 2;                // Reason for yielding
  map<string, string> checkpoint_metadata = 3; // Optional metadata
}

message YieldResponse {
  string session_id = 1;
  AgentState new_state = 2;                  // Usually PAUSED
  string checkpoint_reference = 3;           // Reference for later restore
  int64 yield_timestamp = 4;
}
```

### Semantics

**Pre-yield validation**:
- Agent MUST be in THINKING state
- Substrate creates checkpoint of current agent state
- Checkpoint header includes token count, attention windows, open capabilities

**Checkpoint creation**:
1. Serialize agent's internal state → opaque_payload
2. Create CheckpointHeader with metadata
3. Compute SHA-256 hash of full payload
4. Store checkpoint reference (e.g., S3 path, database ID)
5. Return checkpoint_reference to agent

**State transition**:
- THINKING → PAUSED (successful yield)
- If checkpoint creation fails → return error, agent remains THINKING

**Token accounting**:
- Tokens already consumed remain counted
- Yielding does NOT reverse token count
- When resumed, agent continues with same total_tokens_consumed

### Example

```python
# Agent has made progress on long task, yields before budget exhausted
response = substrate.UsageYield(YieldRequest(
  session_id="ag-uuid-xxx",
  checkpoint_hint="Completed first phase of analysis, pausing for 1 hour",
  checkpoint_metadata={
    "phase": "analysis_complete",
    "tokens_used_so_far": "150000"
  }
))

# Returns: new_state=PAUSED, checkpoint_reference="s3://checkpoints/ag-xxx/cp-001"
# Agent can now safely exit
# 1 hour later, substrate can restore from checkpoint_reference
```

---

## 7. UsageSetCapability — Grant or Update Capability

### Purpose
Dynamically grant or update a capability token to an agent. Does not require checkpoint restore.

### Protobuf Definition
```protobuf
message SetCapabilityRequest {
  string session_id = 1;
  CapabilityToken capability = 2;
}

message SetCapabilityResponse {
  bool success = 1;
  string capability_id = 2;
  string decision_explanation = 3;
}

message CapabilityToken {
  string id = 1;
  string agent_identity = 2;
  string resource_or_tool = 3;
  string operation = 4;
  int64 issued_at = 5;
  int64 expires_at = 6;
  map<string, string> constraints = 7;
  string signature = 8;
}
```

### Semantics
- Parent agent (if any) must have GRANT_CAPABILITY capability
- Capability is immediately active
- Substrate logs capability grant for audit trail
- Agent does NOT need to be in any particular state

---

## 8. UsageRevokeCapability — Revoke Capability Token

### Purpose
Immediately revoke a capability from an agent, even mid-execution.

### Protobuf Definition
```protobuf
message RevokeCapabilityRequest {
  string session_id = 1;
  string capability_id = 2;
  string revocation_reason = 3;
}

message RevokeCapabilityResponse {
  bool success = 1;
  int64 revocation_timestamp = 2;
  string decision_explanation = 3;
}
```

### Semantics
- Revocation is immediate and irrevocable
- If agent is currently executing tool with revoked capability: tool call completes, but next tool call using revoked capability is denied
- Substrate logs revocation with reason for audit trail

---

## 9. Error Semantics

All errors returned by ASI follow gRPC standard error codes:

| Code | Meaning | Retryable |
|------|---------|-----------|
| **CANCELLED** | Operation cancelled by client | Maybe |
| **UNKNOWN** | Unknown error (substrate bug) | No |
| **INVALID_ARGUMENT** | Invalid request (wrong session_id, etc.) | No |
| **DEADLINE_EXCEEDED** | Request timeout (tool took too long) | Maybe |
| **NOT_FOUND** | Session/checkpoint not found | No |
| **ALREADY_EXISTS** | Duplicate spawn with same name | No |
| **PERMISSION_DENIED** | Capability check failed, tool denied | No |
| **RESOURCE_EXHAUSTED** | Token/budget exhausted, quota limit hit | No |
| **FAILED_PRECONDITION** | State transition illegal, policy violation | No |
| **ABORTED** | Session terminated, cannot proceed | No |
| **UNAVAILABLE** | Substrate service temporarily down | Maybe |
| **INTERNAL** | Substrate internal error | No |

---

## 10. Supervision and Cascading Termination

When a parent agent terminates:

```protobuf
message TerminationCascadePolicy {
  enum CascadeMode {
    CASCADE_UNSPECIFIED = 0;
    CASCADE_NONE = 1;           // Children continue running
    CASCADE_PAUSE = 2;          // Children transition to PAUSED
    CASCADE_TERMINATE = 3;      // Children transition to TERMINATED
  }
}
```

- `CASCADE_NONE`: Parent termination does not affect children
- `CASCADE_PAUSE`: Children pause automatically (checkpoint created)
- `CASCADE_TERMINATE`: Children terminate automatically

Default: `CASCADE_PAUSE` (safe, preserves work)
