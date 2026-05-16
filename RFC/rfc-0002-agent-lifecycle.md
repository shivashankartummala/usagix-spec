# RFC-0002: Agent Lifecycle & State Machine Semantics

**Status**: Implemented (v1.0)  
**Authors**: USAGIX Specification Stewards  
**Created**: 2025-03-20  
**Updated**: 2025-05-16

## Summary

This RFC formalizes the complete lifecycle of a USAGIX agent session, from initialization through termination. It defines the session state machine, spawn/terminate contracts, checkpoint/restore mechanics, and resource cleanup guarantees. All agents operating under USAGIX governance MUST conform to this lifecycle model.

## 1. Semantic Definitions

### 1.1 Lifecycle Concepts

**Session**: A bounded execution context representing one or more agent reasoning loops. Each session has:
- Unique session ID (UUID)
- Parent session ID (nullable, for spawned child sessions)
- Session state (one of: Pending, Active, Thinking, Paused, Terminated)
- Token budget (remaining quota)
- Context snapshot (L1/L2/L3 memory state)
- Audit trail (immutable decision log)

**Spawn**: The operation that creates a new child session from a parent session. Allocates a budget partition and establishes parent-child relationship.

**Checkpoint**: The operation that suspends a session mid-reasoning and serializes its state for restoration. Produces a checkpoint record with version, timestamp, and hash.

**Restore**: The operation that deserializes a checkpoint and resumes execution at the exact point of suspension.

**Termination**: The operation that closes a session and reclaims all resources. May be:
- **Graceful**: Agent consumed its budget or completed successfully
- **Forced**: Governance Plane signaled termination (timeout, policy violation, etc.)
- **Cascading**: Parent session termination triggers child cleanup

**Budget Partition**: The token quota allocated to a child session from parent budget. Includes:
- Hard quota (max tokens for this session)
- Soft quota (warning threshold for remaining budget)
- Policy floor (minimum quota for any child)

### 1.2 Formal Type System

```
SessionID          = UUID
ParentSessionID    = UUID | null
SessionState       = Pending | Active | Thinking | Paused | Terminated
TokenBudget        = {
  allocated: uint64,
  consumed: uint64,
  remaining: uint64
}
CheckpointRecord   = {
  id: UUID,
  session_id: SessionID,
  timestamp: Timestamp,
  format_version: string,
  state_hash: bytes32,
  token_consumed_at_checkpoint: uint64,
  context_size_bytes: uint64,
  parent_session_id: SessionID | null,
  policy_version: string,
  signature: bytes
}
SessionMetadata    = {
  id: SessionID,
  parent_id: SessionID | null,
  child_ids: Set<SessionID>,
  state: SessionState,
  budget: TokenBudget,
  created_at: Timestamp,
  terminated_at: Timestamp | null,
  checkpoint_records: List<CheckpointRecord>
}
```

## 2. Formal Contracts

### 2.1 Session State Machine

**State Diagram**:
```
[*] → Pending → Active → Thinking ⇄ Paused → Terminated → [*]

Transitions:
- Pending→Active: Identity established, admission decision approved
- Active→Thinking: First inference loop begins
- Thinking→Paused: Yield signal or checkpoint request
- Paused→Thinking: Resume from checkpoint (if not terminated)
- Thinking→Pending: (invalid, caught by substrate)
- Paused→Active: (invalid, caught by substrate)
- *→Terminated: Terminal state, irreversible
```

**Invariant (State Determinism)**: For any given session ID, if:
1. Replay all input signals (spawn args, tool responses, policy decisions)
2. In identical order and timing
3. Using identical policy versions and code paths

Then the state trajectory and state duration MUST be identical.

**Proof Sketch**: Session state is a pure function of:
- Initial parameters (parent budget, model ID, system prompt)
- Input signals (tool responses, policy decisions, checkpoint versions)
- Elapsed time (bounded by timeout)

If inputs are identical, pure function outputs must be identical.

### 2.2 Spawn Contract

**Definition**: `UsagixSpawn(parent_session_id, child_config, budget_quota) → (child_session_id, error)`

**Preconditions**:
- Parent session exists and is in Active or Thinking state
- Parent has remaining budget ≥ budget_quota + policy_floor
- Child config specifies valid model ID, system prompt, timeout
- budget_quota > 0

**Postconditions (on Success)**:
1. New session created with state=Pending
2. Parent budget immediately decreases by budget_quota (atomic)
3. Child session inherits parent's policy floor (cannot reduce quota further for grandchildren)
4. Audit entry created: `{type: SPAWN, parent: parent_session_id, child: child_session_id, quota: budget_quota}`
5. Child session transitions to Active within T_admission seconds (default: 5s)

**Postconditions (on Failure)**:
1. No child session created
2. Parent budget unchanged
3. Error returned with reason (insufficient budget, invalid config, policy denial)
4. Audit entry created: `{type: SPAWN_DENIED, parent: parent_session_id, reason: error_code}`

**Atomicity**: Spawn decision (allow/deny) and parent budget deduction are atomic. If system crashes mid-spawn, agent receives no response and must retry.

**Budget Inheritance Guarantee**: `child.quota ≤ parent.remaining_after_spawn`

### 2.3 Checkpoint & Restore Contract

**Definition**: `UsagixYield(session_id, checkpoint_request) → (checkpoint_record, error)`

**Checkpoint Structure**:
```
CheckpointRecord = {
  id: UUID,
  session_id: SessionID,
  timestamp: Timestamp,
  format_version: "1.0",
  state_hash: SHA256(serialized_session_state),
  token_consumed_at_checkpoint: current_tokens_consumed,
  context_size_bytes: L1_context_bytes,
  l2_cache_keys: List<CacheKey>,
  policy_version: current_policy_version,
  signature: HMAC-SHA256(record_without_signature, governance_key)
}
```

**Preconditions for Checkpoint**:
- Session is in Active or Thinking state
- Token budget ≥ T_checkpoint_cost (default: 100 tokens)
- Context state is consistent (no in-flight tool calls)

**Postconditions (on Success)**:
1. Session transitions to Paused
2. Checkpoint record created with format version 1.0
3. Session context paged to L2/L3 storage (see RFC-0003)
4. Checkpoint record signed by governance key
5. Audit entry: `{type: CHECKPOINT, session: session_id, checkpoint_id: checkpoint.id, tokens_consumed: X}`

**Restore Preconditions**:
- Checkpoint record exists and signature validates
- Current policy version matches checkpoint policy version (or has compatibility guarantee)
- L2/L3 data still accessible and matches state_hash

**Restore Postconditions**:
1. Session restored to exact state at checkpoint time
2. Session transitions from Paused to Active
3. Inference resumes at next token position (no re-execution)
4. Audit entry: `{type: RESTORE, session: session_id, checkpoint_id: checkpoint.id}`

**Format Version Guarantee**: Checkpoint format v1.0 MUST be readable by all USAGIX v1.x implementations. Breaking changes require v2.0.

**State Hash Verification**: Substrate MUST verify `state_hash` matches restored state. Mismatch is fatal (session terminated with error).

### 2.4 Termination Contract

**Definition**: `UsagixTerminate(session_id, reason, signal_type) → (ok, error)`

**Termination Signals**:
- `BUDGET_EXHAUSTED`: Token quota consumed
- `TIMEOUT`: Session exceeded maximum duration
- `POLICY_VIOLATION`: Governance policy denied operation
- `USER_CANCEL`: Parent or orchestrator requested termination
- `RESOURCE_LIMIT`: System resource exhausted (memory, file descriptors, etc.)
- `ERROR`: Unrecoverable error in session

**Graceful Termination (Budget/Timeout)**:
1. Session state → Terminated
2. Return final audit log and summary to parent
3. Resource cleanup begins (L1 context freed, L2/L3 paging stops)
4. Child sessions receive cascading termination signal

**Forced Termination (Signal)**:
1. Governance Plane sends `SIG_AGENT_TERMINATE` to cognitive container
2. Agent has T_grace_period seconds (default: 30s) to clean up
3. If not terminated after grace period, substrate kills process
4. Resource cleanup proceeds (forced)

**Cascading Termination**:
- When parent session terminates, all child sessions receive PARENT_TERMINATED signal
- Child sessions transition to Terminated within T_cascade_propagate seconds (default: 5s)
- Grandchild sessions receive signal recursively

**Invariant (Guaranteed Cleanup)**: All resources allocated to a session MUST be reclaimed within T_grace_period + T_cleanup_overhead (default: 30s + 10s).

**Invariant (Audit Completeness)**: A termination audit entry MUST be created before resources are freed:
```
{
  type: TERMINATE,
  session_id: SessionID,
  parent_id: SessionID | null,
  termination_reason: TerminationReason,
  tokens_consumed: uint64,
  tokens_allocated: uint64,
  duration_seconds: float,
  checkpoint_count: uint32,
  error_count: uint32,
  timestamp: Timestamp
}
```

### 2.5 Parent-Child Budget Relationship

**Budget Partition Guarantee**:
```
∀ child ∈ children(parent):
  child.allocated ≤ parent.remaining_at_spawn_time
  AND parent.policy_floor ≤ child.policy_floor
  AND SUM(child.allocated for all children) ≤ parent.allocated
```

**Policy Floor Guarantee**: If parent has policy_floor=1000, all children MUST have policy_floor ≥ 1000. This ensures no child can starve grandchildren.

## 3. Lifecycle Guarantees

### 3.1 Pending → Active Transition

**SLO**: Session transitions from Pending to Active within T_admission seconds (default: 5 seconds).

**Preconditions**:
- Session ID created by UsagixSpawn
- Parent has identity and policy floor established
- Admission policy evaluated

**Entry Conditions**:
- Model initialized
- System prompt loaded
- Initial context window allocated (L1)

**Exit Conditions**:
- Ready to begin inference
- Audit entry created: `{type: STATE_TRANSITION, session: session_id, from: Pending, to: Active, timestamp: T}`

### 3.2 Active → Thinking Transition

**Trigger**: First UsagixInference call or LLM invocation

**Preconditions**:
- Session in Active state
- Context window allocated and ready
- Token counter at 0

**Entry Conditions**:
- Model inference begins
- Token consumption starts

**Exit Conditions**:
- Tool invocation or checkpoint requested
- Session transitions to Paused or remains in Thinking

### 3.3 Thinking ⇄ Paused Transitions

**Thinking → Paused**:
- Trigger: UsagixYield (explicit checkpoint) or UsagixSignal(PAUSE)
- Session suspends mid-inference
- L1 context paged to L2/L3 (see RFC-0003)

**Paused → Thinking**:
- Trigger: UsagixResume or parent provides tool response
- Context restored from checkpoint
- Inference resumes

**Cycle Guarantee**: A session may transition between Thinking and Paused multiple times within budget. Each cycle MUST be logged.

### 3.4 Any State → Terminated Transition

**Trigger**: Budget exhausted, timeout, policy violation, or forced signal

**Preconditions**:
- No preconditions; termination is always allowed

**Entry Conditions**:
- Session state set to Terminated
- Resource cleanup initiated
- Child sessions receive cascading termination

**Exit Conditions**:
- Session deleted (after audit retention period)
- Resources fully freed
- Termination audit entry finalized

## 4. Checkpoint Format & Versioning

### 4.1 Checkpoint Header

```
CheckpointHeader = {
  format_version: "1.0",
  timestamp: uint64 (milliseconds since epoch),
  tokens_consumed_at_checkpoint: uint64,
  tokens_allocated: uint64,
  context_hash: bytes32 (SHA256 of serialized L1 context),
  governance_metadata: {
    policy_version: string,
    capabilities_active: List<CapabilityID>,
    audit_log_offset: uint64
  },
  session_id: UUID,
  checkpoint_id: UUID,
  parent_session_id: UUID | null,
  signature: bytes (HMAC-SHA256)
}
```

### 4.2 Format Version Semantics

**v1.0 Compatibility**:
- All USAGIX v1.0, v1.1, v1.2, ... implementations MUST read v1.0 checkpoints
- New fields may be added in v1.1 with default values for backward compat
- Removed fields or wire format changes require v2.0

**Version Negotiation**:
- Restore operation checks checkpoint.format_version
- If version > implementation_max_version, restore fails with CHECKPOINT_INCOMPATIBLE
- If version ≤ implementation_max_version, restore proceeds

## 5. Resource Cleanup Semantics

### 5.1 Termination Resource Reclamation

When a session terminates:
1. **Immediate** (< 100ms): L1 context freed (memory deallocated)
2. **Fast** (< 1s): L2 cache entries invalidated (Redis/Memcached)
3. **Eventual** (< 30s): L3 cold storage deleted (S3 objects removed)
4. **Async** (< 5 min): Audit logs moved to archival storage

**Cleanup Failures**: If cleanup fails (e.g., S3 delete timeout), substrate MUST:
1. Retry with exponential backoff (1s, 2s, 4s, 8s, 16s, 30s cap)
2. Log cleanup errors to audit trail
3. Eventually succeed or escalate to operator

### 5.2 Child Session Cleanup

When parent terminates, child cleanup is cascading:
```
for each child in children(parent):
  send PARENT_TERMINATED signal
  wait up to T_cascade_propagate for graceful shutdown
  if not terminated:
    force terminate child
  recursively cleanup grandchildren
```

## 6. Mandatory Lifecycle Controls

For a substrate to claim USAGIX v1.0 compliance:

1. **State Machine Enforcement**: Session state transitions MUST follow the state machine diagram
2. **Spawn Atomicity**: Budget deduction and child creation are atomic
3. **Checkpoint Integrity**: Checkpoint signatures MUST be validated on restore
4. **Termination Guarantee**: All sessions MUST terminate within grace period + cleanup overhead
5. **Budget Monotonicity**: Token consumption never decreases
6. **Audit Completeness**: All state transitions logged atomically with session ID, timestamp, and reason
7. **Cascading Cleanup**: Parent termination triggers child cleanup within SLO

## 7. Example: Agent Lifecycle

```
Timeline:
T0:   UsagixSpawn(parent_id, {model: claude-opus}, budget: 100000)
      → child_session_id = UUID-child
      → state = Pending
      → audit: {type: SPAWN, parent: parent_id, child: UUID-child}

T1:   [Admission policy approved]
      → state = Active
      → audit: {type: STATE_TRANSITION, from: Pending, to: Active}

T2:   First inference call
      → state = Thinking
      → token_consumed starts incrementing

T3:   (tokens: 5000/100000)
      → UsagixYield(checkpoint_request)
      → checkpoint created, signed
      → state = Paused
      → audit: {type: CHECKPOINT, tokens_consumed: 5000}

T4:   [Parent provides tool response]
      → UsagixResume(UUID-checkpoint)
      → context restored, state = Active → Thinking
      → inference continues from checkpoint

T5:   (tokens: 95000/100000)
      → Token budget exhausted
      → UsagixTerminate(BUDGET_EXHAUSTED)
      → state = Terminated
      → cleanup proceeds (L1→L2→L3)
      → audit: {type: TERMINATE, reason: BUDGET_EXHAUSTED, tokens: 100000}

T6:   (T5 + cleanup_overhead)
      → All resources freed
      → session deleted from active registry
```

## 8. Backward Compatibility

USAGIX v1.0 is the initial stable version. Checkpoint format v1.0 is stable.

For future versions:
- v1.1+: May add new checkpoint fields with defaults; MUST read v1.0 checkpoints
- v2.0: May break checkpoint format; old checkpoints NOT readable

Migration path: Implementations MUST support both v1.0 and v1.1 checkpoint reading. No in-place upgrade of checkpoints is required.

## 9. Normative References

- RFC-0001: USAGIX Core Architecture & Trust Domains
- RFC-0003: Memory Model & Context Management
- RFC-0004: Governance Model & Policy Evaluation
- RFC-0005: ASI System Calls
- RFC 2119 — Keywords for use in specs

## 10. Open Questions & Future Work

- **Q1**: Should child sessions inherit parent's system prompt or specify their own? Answer deferred to governance model.
- **Q2**: What is the maximum depth of session hierarchy (parent→child→grandchild→...)? Answer deferred to implementation guide.
- **Q3**: Can a checkpoint span multiple model invocations, or must each inference call be a separate checkpoint? Answer deferred to inference scheduler RFC.

---

**Document Hash**: `sha256(this_document)`  
**Approvals**:
- [ ] TSC Architecture Lead
- [ ] TSC Agent Lifecycle Lead
- [ ] Community Review (2-week period)
