# RFC-0005: Agent Substrate Interface (ASI) System Calls

**Status**: Implemented (v1.0)  
**Authors**: USAGIX Specification Stewards  
**Created**: 2025-03-28  
**Updated**: 2025-05-16

## Summary

This RFC defines the Agent Substrate Interface (ASI), a gRPC-based protocol for agent-to-substrate communication. All interactions between the cognitive container and governance enforcement plane transit through ASI system calls. Each call has formal request/response contracts, error semantics, and idempotency guarantees.

## 1. Semantic Definitions

### 1.1 ASI Concepts

**System Call**: A request-response interaction between agent code (cognitive container) and substrate infrastructure (governance enforcement plane). All system calls are:
- Gated by capability tokens
- Logged in audit trail
- Subject to policy evaluation
- Bounded by timeout

**Request**: The agent's invocation of a system call with parameters.

**Response**: The substrate's result (success/error) with output data.

**Idempotency**: A call that can be safely retried; multiple identical requests produce the same result (at most once semantics).

**Timeout**: Maximum time allowed for a system call to complete (default: 30 seconds). Exceeded timeout results in DEADLINE_EXCEEDED error.

**RPC**: Remote Procedure Call (gRPC framework). All ASI calls use gRPC with Protocol Buffers v3 serialization.

### 1.2 System Calls Overview

| System Call | Purpose | Idempotent | Timeout |
|-------------|---------|-----------|---------|
| UsagixSpawn | Create child session | Yes | 5s |
| UsagixYield | Checkpoint & suspend | Yes | 1s |
| UsagixResume | Resume from checkpoint | Yes | 5s |
| UsagixSignal | Pause/resume/terminate | Yes | 1s |
| UsagixMemPageOut | Page context to L2 | No | 500ms |
| UsagixMemPageIn | Restore context from L2/L3 | Yes | 10s |
| UsagixCallTool | Invoke external tool | Conditional | Varies |
| UsagixGetMemoryPressure | Query memory utilization | Yes | 10ms |
| UsagixCreateApprovalRequest | Request approval for action | Yes | 1s |
| UsagixQueryAuditLog | Retrieve audit records | Yes | 5s |

## 2. Formal Contracts

### 2.1 UsagixSpawn

**Purpose**: Create a new child session with a budget quota.

**Request**:
```protobuf
message SpawnRequest {
  bytes parent_session_id = 1;      // UUID (16 bytes)
  ChildConfig child_config = 2;      // See below
  uint64 budget_quota = 3;           // Tokens to allocate to child
  map<string, string> labels = 4;    // Optional metadata (domain, tier, etc.)
}

message ChildConfig {
  string model_id = 1;               // e.g., "claude-opus-4-7"
  string system_prompt = 2;          // System prompt for child
  uint32 session_timeout_seconds = 3; // Max duration
  uint32 max_concurrent_sessions = 4; // If this child spawns grandchildren
}
```

**Response**:
```protobuf
message SpawnResponse {
  bytes child_session_id = 1;        // UUID (16 bytes) of new session
  uint64 allocated_budget = 2;       // Echo of budget_quota
  string status = 3;                 // "PENDING" (will transition to ACTIVE)
}
```

**Errors**:
- `INVALID_ARGUMENT`: parent_session_id invalid or parent not in Active/Thinking state
- `PERMISSION_DENIED`: Parent lacks quota for spawn
- `RESOURCE_EXHAUSTED`: System limit on concurrent sessions exceeded
- `UNAVAILABLE`: Substrate overloaded

**Idempotency**: Idempotent. Retrying with same (parent_id, child_config, budget) returns same child_session_id.

**Implementation Notes**:
- Spawn is fast (< 5s SLO); child session transitions to Active asynchronously
- Parent budget decremented immediately (atomic)
- Child inherits parent's policy floor

### 2.2 UsagixYield

**Purpose**: Checkpoint current session state and suspend to L2/L3.

**Request**:
```protobuf
message YieldRequest {
  bytes session_id = 1;              // UUID
  CheckpointRequest checkpoint = 2;  // See below
}

message CheckpointRequest {
  bytes context_snapshot = 1;        // Serialized L1 context (protobuf)
  uint64 tokens_consumed = 2;        // Tokens consumed so far
  string format_version = 3;         // "1.0"
  map<string, string> metadata = 4;  // Optional checkpoint metadata
}
```

**Response**:
```protobuf
message YieldResponse {
  bytes checkpoint_id = 1;           // UUID of checkpoint
  bytes checkpoint_hash = 2;         // SHA256(context_snapshot)
  uint64 paging_cost_tokens = 3;     // Tokens deducted for paging
  string l2_cache_key = 4;           // Cache key if paged to L2
  string l3_storage_path = 5;        // S3 path if paged to L3 (optional)
}
```

**Errors**:
- `INVALID_ARGUMENT`: Session not in Active/Thinking state
- `RESOURCE_EXHAUSTED`: L1 or L2 storage full
- `DEADLINE_EXCEEDED`: Paging took > 1 second
- `INTERNAL`: Signature generation failed

**Idempotency**: Idempotent. Retrying with same session_id and context_snapshot within T_dedupe_window (default: 60s) returns same checkpoint_id.

**Cost**: Deducts T_checkpoint_cost tokens (default: 100 tokens).

### 2.3 UsagixResume

**Purpose**: Resume a paused session from checkpoint.

**Request**:
```protobuf
message ResumeRequest {
  bytes session_id = 1;              // UUID
  bytes checkpoint_id = 2;           // UUID of checkpoint to restore
  ToolResponse tool_response = 3;    // Optional: tool response if resuming from tool call
}

message ToolResponse {
  string tool_name = 1;
  string status = 2;                 // "success", "error", "timeout"
  bytes result = 3;                  // Tool output (JSON/binary)
  string error_message = 4;          // If status=error
}
```

**Response**:
```protobuf
message ResumeResponse {
  bytes restored_context = 1;        // Deserialized L1 context
  uint64 restore_latency_ms = 2;     // How long restore took
  string session_state = 3;          // "ACTIVE" or "THINKING"
}
```

**Errors**:
- `NOT_FOUND`: Checkpoint does not exist
- `FAILED_PRECONDITION`: Checkpoint hash mismatch
- `DEADLINE_EXCEEDED`: Restore took > 10s
- `PERMISSION_DENIED`: Session not authorized to restore this checkpoint

**Idempotency**: Idempotent. Retrying with same (session_id, checkpoint_id) restores same context.

### 2.4 UsagixSignal

**Purpose**: Send a signal to a session (pause, resume, terminate, memory pressure alert).

**Request**:
```protobuf
message SignalRequest {
  bytes session_id = 1;              // UUID
  SignalType signal = 2;             // See enum below
  uint32 grace_period_seconds = 3;   // For TERMINATE signal
  string reason = 4;                 // Human-readable reason
}

enum SignalType {
  SIGNAL_UNSPECIFIED = 0;
  PAUSE = 1;                         // Suspend without checkpoint
  RESUME = 2;                        // Resume from pause
  TERMINATE = 3;                     // Graceful shutdown
  MEMORY_PRESSURE = 4;               // Alert: memory utilization high
  POLICY_VIOLATION = 5;              // Alert: policy rule violated
}
```

**Response**:
```protobuf
message SignalResponse {
  string status = 1;                 // "ACK" (acknowledged)
  uint64 signal_latency_ms = 2;      // Delivery latency
}
```

**Errors**:
- `NOT_FOUND`: Session does not exist
- `INVALID_ARGUMENT`: Unsupported signal type
- `DEADLINE_EXCEEDED`: Signal delivery timeout

**Idempotency**: Idempotent. Retrying same signal within grace period results in same outcome.

**Behavior**:
- PAUSE: Session moves to Paused state without checkpointing
- RESUME: Resumes from Paused state (typically after pressure alert)
- TERMINATE: Initiates graceful shutdown with grace_period_seconds timeout
- MEMORY_PRESSURE: Informs agent of high memory utilization (advisory, not mandatory)

### 2.5 UsagixCallTool

**Purpose**: Invoke an external tool (API, database, file system, web service).

**Request**:
```protobuf
message CallToolRequest {
  bytes session_id = 1;              // UUID
  string tool_name = 2;              // e.g., "web.search", "database.query"
  map<string, string> parameters = 3; // Tool parameters
  bytes capability_token = 4;        // JWT capability token
  uint32 timeout_seconds = 5;        // Tool invocation timeout
}
```

**Response**:
```protobuf
message CallToolResponse {
  string status = 1;                 // "success", "error", "timeout"
  bytes result = 2;                  // Tool output (JSON/binary/text)
  string result_type = 3;            // "json", "text", "binary"
  string error_message = 4;          // If status=error
  uint64 latency_ms = 5;             // Tool execution time
  bytes redacted_result = 6;         // Output with PII/secrets redacted
}
```

**Errors**:
- `PERMISSION_DENIED`: Capability invalid or scope violated
- `INVALID_ARGUMENT`: Tool name not found or parameters invalid
- `DEADLINE_EXCEEDED`: Tool execution exceeded timeout_seconds
- `RESOURCE_EXHAUSTED`: Tool rate limit exceeded
- `UNAVAILABLE`: Tool service unavailable

**Idempotency**: Conditional. Depends on tool:
- Read-only tools (web.search, database.query): Idempotent
- Write tools (file.write, database.insert): Not idempotent; deduplication key must be provided

**Policy Gating**:
1. Verify capability_token (signature, expiration, scope)
2. Evaluate governance policy (OPA)
3. If policy denies: return PERMISSION_DENIED
4. If policy requires approval: return AWAITING_APPROVAL
5. If approved: invoke tool and return result

**Output Handling**:
- Raw result returned in `result` field
- Redacted version (PII removed) in `redacted_result` field
- Agent receives both; caller receives redacted version

### 2.6 UsagixMemPageOut

**Purpose**: Manually page context segment from L1 to L2.

**Request**:
```protobuf
message PageOutRequest {
  bytes session_id = 1;              // UUID
  string segment_id = 2;             // Segment identifier
  bytes segment_data = 3;            // Segment bytes to page
  uint32 segment_size_bytes = 4;     // For accounting
}
```

**Response**:
```protobuf
message PageOutResponse {
  string l2_cache_key = 1;           // Key to retrieve segment
  uint64 paging_cost_tokens = 2;     // Tokens deducted
  uint64 paging_latency_ms = 3;      // How long paging took
}
```

**Errors**:
- `RESOURCE_EXHAUSTED`: L2 storage full
- `DEADLINE_EXCEEDED`: Paging > 500ms

**Idempotency**: Not idempotent (distinct segments can have same ID; don't retry with different data).

### 2.7 UsagixMemPageIn

**Purpose**: Restore context segment from L2/L3 to L1.

**Request**:
```protobuf
message PageInRequest {
  bytes session_id = 1;              // UUID
  string cache_key = 2;              // L2 cache key or L3 path
  string source = 3;                 // "L2" or "L3"
}
```

**Response**:
```protobuf
message PageInResponse {
  bytes segment_data = 1;            // Restored segment
  uint64 paging_latency_ms = 2;      // Restore latency
}
```

**Errors**:
- `NOT_FOUND`: Segment not found in cache/storage
- `DEADLINE_EXCEEDED`: Restore > 10s

**Idempotency**: Idempotent.

### 2.8 UsagixGetMemoryPressure

**Purpose**: Query current memory utilization.

**Request**:
```protobuf
message MemoryPressureRequest {
  bytes session_id = 1;              // UUID
}
```

**Response**:
```protobuf
message MemoryPressureResponse {
  uint32 l1_utilization_percent = 1;  // 0–100
  uint32 l2_utilization_percent = 2;  // 0–100
  uint32 l3_utilization_percent = 3;  // 0–100
  uint32 segment_count = 4;           // Total segments in all tiers
  uint64 largest_segment_bytes = 5;   // Size of largest segment
  float paging_latency_ms = 6;        // Recent paging latency
}
```

**Errors**: None (always succeeds)

**Idempotency**: Idempotent.

**Latency SLO**: Returns within 10ms.

### 2.9 UsagixCreateApprovalRequest

**Purpose**: Request human approval for an action that requires it.

**Request**:
```protobuf
message ApprovalRequest {
  bytes session_id = 1;              // UUID
  string action_description = 2;     // Human-readable action
  string reason = 3;                 // Why agent is requesting approval
  uint32 ttl_seconds = 4;            // Approval expires after this time
}
```

**Response**:
```protobuf
message ApprovalResponse {
  bytes approval_request_id = 1;     // UUID of request
  string status = 2;                 // "PENDING"
  uint64 created_at = 3;             // Timestamp (ms since epoch)
}
```

**Errors**:
- `INVALID_ARGUMENT`: action_description empty or > 1000 chars

**Idempotency**: Idempotent within T_dedupe_window (60s).

### 2.10 UsagixQueryAuditLog

**Purpose**: Retrieve audit records for this session.

**Request**:
```protobuf
message AuditLogQuery {
  bytes session_id = 1;              // UUID
  uint64 start_time_ms = 2;          // Timestamp range (inclusive)
  uint64 end_time_ms = 3;            // Timestamp range (exclusive)
  repeated string decision_filter = 4; // ["Allow", "Deny", "Error"]
  string tool_filter = 5;            // Optional: filter by tool name
  uint32 limit = 6;                  // Max records to return (default: 1000)
}
```

**Response**:
```protobuf
message AuditLogResponse {
  repeated AuditRecord records = 1;
  uint32 total_matching = 2;         // Total matching records (may be > limit)
  string next_page_token = 3;        // For pagination
}

message AuditRecord {
  bytes id = 1;
  uint64 timestamp_ms = 2;
  bytes session_id = 3;
  string decision = 4;               // "Allow", "Deny", "Error"
  string policy_version = 5;
  string tool_name = 6;
  string rule_id = 7;
  string rationale = 8;
  float latency_ms = 9;
  bytes signature = 10;              // HMAC-SHA256
}
```

**Errors**: None (returns empty list if no matches)

**Idempotency**: Idempotent.

## 3. Error Semantics

All ASI calls return gRPC status codes:

| Code | Meaning | Retry? |
|------|---------|--------|
| OK (0) | Success | No |
| CANCELLED (1) | Call cancelled by client | Maybe |
| UNKNOWN (2) | Unknown error | Retry with backoff |
| INVALID_ARGUMENT (3) | Bad input | No (fix and retry) |
| DEADLINE_EXCEEDED (4) | Timeout | Retry with backoff |
| NOT_FOUND (5) | Resource missing | No (resource does not exist) |
| ALREADY_EXISTS (6) | Resource already exists | No (already created) |
| PERMISSION_DENIED (7) | Policy or capability denial | No (retry without help) |
| RESOURCE_EXHAUSTED (8) | Quota/limit exceeded | Retry with backoff |
| FAILED_PRECONDITION (9) | State machine violation | No |
| ABORTED (10) | Call aborted (race condition) | Retry with backoff |
| INTERNAL (13) | Internal substrate error | Retry with backoff |
| UNAVAILABLE (14) | Service unavailable | Retry with backoff |
| DEADLINE_EXCEEDED (4) | Timeout exceeded | Retry with backoff |

**Retry Strategy**: Exponential backoff with jitter: 100ms, 200ms, 400ms, 800ms, 1.6s, 3.2s, max 30s.

## 4. Transport & Security

### 4.1 gRPC Channel

- **Protocol**: gRPC with HTTP/2
- **Address**: localhost:50051 (or secure boundary endpoint)
- **Authentication**: mTLS (mutual TLS with client certificate)
- **Encryption**: TLS 1.3 or higher

### 4.2 Message Serialization

- **Format**: Protocol Buffers v3
- **Compression**: Optional (gzip)

### 4.3 Capability Token Format

Capability tokens are JWTs:
```
Header: {
  "alg": "HS256",
  "typ": "JWT"
}

Payload: {
  "sub": "session-id",
  "resource": "tool-name",
  "scope": {
    "param1": ["value1", "value2"],
    "rate_limit": 10
  },
  "exp": 1234567890,
  "iat": 1234567800
}

Signature: HMAC-SHA256(header.payload, governance_secret_key)
```

## 5. Mandatory ASI Controls

For a substrate to claim USAGIX v1.0 compliance:

1. **All System Calls**: Implement all 10 system calls as specified
2. **gRPC Protocol**: Use gRPC v1.50+ with Protocol Buffers v3
3. **mTLS Authentication**: Mutual TLS on all ASI channels
4. **Request-Response Contracts**: Enforce pre/post conditions
5. **Error Handling**: Return correct gRPC status codes
6. **Idempotency**: Support idempotency as specified (or make safe retries)
7. **Timeout Enforcement**: Hard timeout limits as specified
8. **Audit Logging**: All calls logged in audit trail
9. **Capability Validation**: Verify JWT signatures before allowing calls

## 6. Backward Compatibility

USAGIX v1.0 defines ASI v1.0. Future versions may:
- Add new system calls (v1.1+)
- Add optional fields to requests/responses (backward compatible)

Changes requiring v2.0:
- Removing existing system calls
- Changing request/response structure (breaking changes)
- Changing error semantics

## 7. Normative References

- RFC-0001: USAGIX Core Architecture & Trust Domains
- RFC-0002: Agent Lifecycle & State Machine Semantics
- RFC-0004: Governance Model & Policy Evaluation
- gRPC Specification v1.50+
- Protocol Buffers Language Guide v3
- RFC 7519 — JSON Web Token (JWT)

## 8. Open Questions & Future Work

- **Q1**: Should ASI support HTTP/REST in addition to gRPC? Answer deferred to API standardization RFC.
- **Q2**: Should capability tokens support expiration refresh without new governance decision? Answer: TBD in capability management RFC.
- **Q3**: Can agents batch multiple UsagixCallTool requests for throughput? Answer deferred to performance optimization RFC.

---

**Document Hash**: `sha256(this_document)`  
**Approvals**:
- [ ] TSC API Lead
- [ ] TSC Protocol Lead
- [ ] Community Review (2-week period)
