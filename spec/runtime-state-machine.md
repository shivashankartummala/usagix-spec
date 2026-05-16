# Agent Runtime State Machine Specification

This document provides detailed specifications for the agent state machine, including state definitions, transition rules, and formal semantics.

## 1. State Definitions

USAGE defines exactly **5 states** in the agent lifecycle:

### 1.1 PENDING

**Meaning**: Agent session created, but not yet active. No inference or tool execution occurs.

**Duration**: Seconds to minutes (waiting for activation)

**Characteristics**:
- Session ID assigned
- Initial checkpoint created
- Quota and capabilities bound
- Governance policies loaded
- No tokens consumed
- No tool invocations possible

**Valid Transitions**:
- PENDING → ACTIVE (normal activation)
- PENDING → TERMINATED (reject without running)

**Example Sequence**:
```
2:00 PM: UsageSpawn() called
         Response: session_id="ag-xxx", state=PENDING
         
2:00:01 PM: Policy validation, infrastructure checks
            
2:00:02 PM: UsageSetState(target_state=ACTIVE)
            Response: state=ACTIVE, ready for execution
            
OR
            
2:05 PM: Emergency termination needed
         UsageSetState(target_state=TERMINATED)
         Response: state=TERMINATED (never ran)
```

### 1.2 ACTIVE

**Meaning**: Agent is ready to run. Waiting for inference engine to invoke THINKING, or waiting for tool results.

**Duration**: Seconds (between LLM calls or tool invocations)

**Characteristics**:
- Agent code not currently executing
- Awaiting input (tool results, next instruction)
- Can receive capability grants/revocations
- Governance policies can be updated
- Can transition to PAUSED for cold-start scheduling
- Token budget is decrementing (indirectly, through LLM usage)

**Valid Transitions**:
- ACTIVE → THINKING (enter LLM inference)
- ACTIVE → PAUSED (emergency pause, cold-start scheduling)
- ACTIVE → TERMINATED (graceful shutdown)

**Example Sequence**:
```
2:00 PM: Agent in ACTIVE state
         Awaiting tool results from previous LLM call
         
2:00:15 PM: Tool results arrive
            Agent processes results (still ACTIVE)
            Prepares next LLM prompt (still ACTIVE)
            
2:00:20 PM: Agent ready for inference
            Calls inference engine (transition to THINKING)
            
OR
            
2:00:15 PM: Emergency pause needed
            Governance event: credential compromised
            Substrate calls UsageSetState(target_state=PAUSED)
            Agent transitions: ACTIVE → PAUSED
            Checkpoint created
```

### 1.3 THINKING

**Meaning**: Agent is actively executing LLM inference or processing results.

**Duration**: Seconds to minutes (LLM latency + tool execution)

**Characteristics**:
- LLM inference happening (tokens being consumed)
- Agent processing tool results
- Tool invocations happening (via UsageCallTool)
- Token budget decrementing
- Governance decisions being logged
- Agent is "busy" (cannot be paused except via emergency)

**Valid Transitions**:
- THINKING → ACTIVE (inference complete, awaiting next decision)
- THINKING → PAUSED (yield with checkpoint)
- THINKING → TERMINATED (budget exhausted, critical error, emergency)

**Example Sequence**:
```
2:00:20 PM: Agent enters THINKING (LLM call starts)
            Tokens consumed: 5000
            
2:00:25 PM: LLM returns result (stop)
            Tokens consumed: 5000 + 3000 = 8000
            Agent processes result
            
2:00:26 PM: Agent calls tool:
            UsageCallTool(tool="database.query", ...)
            Substrate validates capability → allowed
            Tool executes, returns result
            (Still in THINKING state during tool execution)
            
2:00:30 PM: Tool completes
            Agent processes tool result
            Decides: "I have enough info, yield"
            Calls UsageYield()
            Substrate transitions: THINKING → PAUSED
            Checkpoint created
            
OR
            
2:00:30 PM: Agent calls tool:
            UsageCallTool(tool="database.query", ...)
            Substrate checks token budget: 500K consumed, 500K budget
            Denial: RESOURCE_EXHAUSTED
            
2:00:30 PM: Agent transitions: THINKING → TERMINATED
            (No more token budget)
```

### 1.4 PAUSED

**Meaning**: Agent execution suspended with checkpoint saved. Can resume later or shutdown gracefully.

**Duration**: Minutes to hours (whatever the pause is for)

**Characteristics**:
- Full checkpoint saved (context, state, budget, capabilities)
- No tokens being consumed
- No tool execution happening
- Agent process can be stopped/hibernated
- Governance policies can be updated
- Checkpoint can be inspected/validated
- No LLM inference occurring

**Valid Transitions**:
- PAUSED → ACTIVE (resume execution)
- PAUSED → TERMINATED (graceful shutdown, destroy checkpoint)

**When Entering PAUSED**:
- Checkpoint header created with:
  - Format version
  - Timestamp
  - total_tokens_consumed (cumulative)
  - active_attention_window_indices
  - current_tool_call_depth
  - open_capability_tokens
  - memory_tier_locators
  - context_hash (SHA-256)
  - governance_compliance_metadata
- Checkpoint signature verified
- Checkpoint reference returned to control plane

**Example Sequence**:
```
2:00:30 PM: Agent yields (THINKING → PAUSED)
            Checkpoint created: cp-001.bin
            checkpoint.header.total_tokens_consumed = 8000
            checkpoint.header.expires_at = 2025-03-15T15:00:00Z
            
2:00:31 PM: Governance policies updated
            New policies applied to open_capability_tokens
            Existing capabilities validated
            
2:30 PM: Batch window opens
         Substrate calls UsageSetState(target_state=ACTIVE)
         Checkpoint loaded: cp-001.bin
         Governance policies re-validated
         Token count: 8000 (preserved from before)
         
2:30:05 PM: Agent resumes in ACTIVE state
            Continues execution with same context
            
OR
            
2:15 PM: Operator decides to terminate this agent
         Calls UsageSetState(target_state=TERMINATED)
         Checkpoint is discarded
         Session ends
```

### 1.5 TERMINATED

**Meaning**: Agent execution finished. Session is closed and cannot be resumed.

**Duration**: Terminal state (no transitions out)

**Characteristics**:
- All resources released
- Checkpoint discarded (unless explicitly archived)
- Session cannot be resumed
- Tokens remain consumed (not refunded)
- Audit logs immutable
- Final audit entry logged with termination reason

**Valid Transitions**:
- None (TERMINATED is terminal)

**Termination Reasons**:
```
REASON_UNSPECIFIED
REASON_COMPLETED               // Normal completion
REASON_BUDGET_EXHAUSTED        // Token budget consumed
REASON_TOOL_DEPTH_EXCEEDED     // Nesting limit reached
REASON_POLICY_VIOLATION        // Governance violation
REASON_CAPABILITY_REVOKED      // Critical capability revoked
REASON_TIMEOUT                 // Execution timeout
REASON_OPERATOR_REQUEST        // Admin termination
REASON_CASCADING              // Parent agent terminated
REASON_INFRASTRUCTURE_ERROR    // Substrate error
```

**Example Sequence**:
```
2:00:30 PM: Agent in THINKING state
            Token budget: 500K
            Tokens consumed: 499K
            
2:00:31 PM: LLM call completes
            Tokens consumed: 499K + 2K = 501K
            
Substrate check: tokens_consumed (501K) > max_tokens (500K)?
Answer: Yes
Action: Transition THINKING → TERMINATED
Reason: REASON_BUDGET_EXHAUSTED

Response to agent: "Token budget exhausted"

2:00:32 PM: Audit log entry:
            {
              timestamp: 2025-03-15T14:00:32Z,
              session_id: ag-12345,
              old_state: THINKING,
              new_state: TERMINATED,
              reason: REASON_BUDGET_EXHAUSTED,
              tokens_consumed: 501000,
              tokens_budget: 500000
            }
```

## 2. State Transition Rules

### 2.1 Legal Transitions

```
PENDING     → ACTIVE          ✓ Normal activation
PENDING     → TERMINATED      ✓ Reject without running

ACTIVE      → THINKING        ✓ Enter LLM inference
ACTIVE      → PAUSED          ✓ Cold-start pause (NEW in v2)
ACTIVE      → TERMINATED      ✓ Graceful shutdown

THINKING    → ACTIVE          ✓ Inference complete
THINKING    → PAUSED          ✓ Yield with checkpoint
THINKING    → TERMINATED      ✓ Budget exhausted or error

PAUSED      → ACTIVE          ✓ Resume execution
PAUSED      → TERMINATED      ✓ Graceful shutdown

TERMINATED  → <any>           ✗ Terminal state, no transitions
```

### 2.2 Illegal Transitions (Substrate Rejects)

```
PAUSED      → THINKING        ✗ Must go through ACTIVE first
TERMINATED  → ACTIVE          ✗ Cannot resurrect terminated agent
TERMINATED  → PAUSED          ✗ Cannot pause terminated agent
ANY         → PENDING         ✗ Cannot return to PENDING

Same State  → Same State      ✗ No-op, return current state
ACTIVE      → ACTIVE          ✗ Already active, no transition
THINKING    → THINKING        ✗ Already thinking, no transition
```

### 2.3 Preconditions for Transitions

#### PENDING → ACTIVE

**Preconditions**:
1. Session MUST be in PENDING state
2. No pending exceptions or errors
3. Governance policies successfully loaded
4. Quota and capabilities successfully bound
5. Infrastructure health check passed (if required)

**Post-actions**:
1. Transition to ACTIVE
2. Log state change with timestamp
3. Emit metric: `agent.state_transitions{from=PENDING,to=ACTIVE}`

#### ACTIVE → THINKING

**Preconditions**:
1. Session MUST be in ACTIVE state
2. Token budget available (tokens_consumed < max_tokens)
3. Tool call depth available (current_depth < max_depth)

**Post-actions**:
1. Transition to THINKING
2. Start token tracking for this LLM call
3. Log state change

#### THINKING → PAUSED (Yield)

**Preconditions**:
1. Session MUST be in THINKING state
2. Checkpoint creation must succeed
3. Open capabilities must be revocable (no permanent grants)

**Post-actions**:
1. Create checkpoint with header + opaque payload
2. Sign checkpoint
3. Store checkpoint reference
4. Transition to PAUSED
5. Log state change with checkpoint reference

#### PAUSED → ACTIVE (Resume)

**Preconditions**:
1. Session MUST be in PAUSED state
2. Checkpoint MUST be valid (signature checks, integrity verified)
3. Governance policies MUST be re-validated:
   - open_capability_tokens MUST be checked against current policies
   - If capability revoked → fail resume with POLICY_VIOLATION
   - If policy constraints changed → log warning but allow resume
4. Token monotonicity verified:
   - checkpoint.total_tokens_consumed MUST equal current total_tokens_consumed

**Post-actions**:
1. Restore agent state from checkpoint
2. Re-validate all governance policies
3. Transition to ACTIVE
4. Log state change with checkpoint restoration details

#### ANY → TERMINATED

**Preconditions**:
1. Session MUST not already be TERMINATED
2. Termination reason MUST be valid

**Post-actions**:
1. Release all resources (network, storage, memory)
2. Close checkpoint (archive or discard)
3. Transition to TERMINATED
4. Log final state change with reason and final metrics
5. Emit metric: `agent.terminated{reason=<reason>}`

## 3. Formal State Machine Definition

### 3.1 States as Enumeration

```
S = {PENDING, ACTIVE, THINKING, PAUSED, TERMINATED}
```

### 3.2 Transition Function

```
δ: S × Event → S

where Event = {
  SPAWN,
  ACTIVATE,
  BEGIN_THINKING,
  YIELD,
  RESUME,
  TERMINATE,
  INTERRUPT,
  BUDGET_EXHAUSTED,
  TOOL_DEPTH_EXCEEDED,
  POLICY_VIOLATION
}

Examples:
δ(PENDING, ACTIVATE) = ACTIVE
δ(ACTIVE, BEGIN_THINKING) = THINKING
δ(THINKING, YIELD) = PAUSED
δ(PAUSED, RESUME) = ACTIVE
δ(THINKING, BUDGET_EXHAUSTED) = TERMINATED
δ(TERMINATED, ANY) = TERMINATED  // Absorbing state
```

### 3.3 Formal Semantics

```
State S has properties:
  - session_id: unique identifier
  - tokens_consumed: cumulative tokens (monotonic)
  - tokens_budget: hard limit
  - checkpoint: optional saved state
  - open_capabilities: set of granted capabilities
  - creation_timestamp: when agent was spawned
  - state_change_timestamp: when state last changed

Transition δ(S, event) → S' satisfies:
  
  1. Determinism: 
     Same state + same event → same next state
     
  2. Token Monotonicity:
     tokens_consumed' ≥ tokens_consumed
     (never decrease token count)
     
  3. Capability Invariant:
     Revoked capabilities are not in open_capabilities'
     (revocation is immediate)
     
  4. Checkpoint Integrity:
     If THINKING → PAUSED:
       checkpoint' ≠ null AND
       checkpoint'.signature is valid AND
       checkpoint'.total_tokens_consumed = tokens_consumed
     
  5. Terminal Absorbing:
     If S = TERMINATED:
       δ(S, event) = TERMINATED for all events
```

## 4. Cold-Start Scheduling Pattern

### 4.1 Motivation

Enterprise needs to spawn agents proactively and queue them for later activation:

```
Use Case: Batch processing at scheduled times
2:00 PM: System spawns 100 agents for batch job
         No tokens consumed yet (no inference)
2:30 PM: Batch window opens
         All 100 agents activated simultaneously
         Inference begins
```

### 4.2 State Machine Support

USAGE v2 authorizes: **ACTIVE → PAUSED transition**

```
Timeline:
2:00 PM: UsageSpawn()
         Response: state=PENDING
         
2:00:01 PM: UsageSetState(target_state=ACTIVE)
            Response: state=ACTIVE
            
2:00:02 PM: UsageSetState(target_state=PAUSED, reason="cold_start")
            Response: state=PAUSED
            Checkpoint created (almost empty)
            
2:30 PM: UsageSetState(target_state=ACTIVE)
         Checkpoint loaded (empty or minimal)
         Governance re-validated
         Token budget preserved
         Inference can begin
```

**Key Properties**:
- No tokens consumed until ACTIVE→THINKING transition
- Checkpoint is created immediately (minimal overhead)
- Policies can be updated while agent is PAUSED
- Agent resumes with fresh policy snapshot

### 4.3 Cost Savings

```
Traditional approach (without cold-start):
  - Spawn agent at 2:00 PM
  - Agent enters ACTIVE
  - Agent enters THINKING (begins inference)
  - Agent waits for batch window (30 minutes)
  - Tokens consumed: ~5-10K (inference overhead, polling)
  - Cost: $0.05-$0.10 per agent

Cold-start approach (with USAGE):
  - Spawn agent at 2:00 PM
  - Agent remains PAUSED (no tokens)
  - Resume at 2:30 PM when batch window opens
  - Tokens consumed: 0 (no inference while waiting)
  - Cost: $0 per agent
  
Savings for 100 agents: $5-$10 per batch cycle
Annual savings: $520-$1040 (assuming 2 batch cycles per day)
```

## 5. State Machine in Examples

### 5.1 Example 1: Normal Completion

```
2:00 PM: UsageSpawn()
         ↓ PENDING

2:00:01 PM: UsageSetState(ACTIVE)
            ↓ ACTIVE

2:00:05 PM: Agent begins work
            UsageCallTool(tool="database.query", ...)
            (Still in ACTIVE, awaiting tool result)
            ↓ ACTIVE

2:00:10 PM: Tool returns result
            Agent processes
            (Still in ACTIVE)
            ↓ ACTIVE

2:00:15 PM: Agent needs LLM inference
            Begins LLM call
            ↓ THINKING

2:00:20 PM: LLM call completes
            Agent determines: task complete
            Transitions back to ACTIVE for cleanup
            ↓ ACTIVE

2:00:21 PM: Agent calls:
            UsageSetState(ACTIVE → TERMINATED)
            ↓ TERMINATED

Final state: TERMINATED
Reason: Normal completion
Tokens consumed: 50,000
```

### 5.2 Example 2: Budget Exhaustion

```
2:00 PM: Agent spawned
         Token budget: 100,000
         ↓ PENDING

2:00:01 PM: Activated
            ↓ ACTIVE

2:00:05 PM: Multiple LLM calls
            Tokens consumed: 95,000
            ↓ THINKING

2:00:10 PM: LLM call returns (5000 tokens)
            Tokens consumed: 100,000
            Substrate check: tokens_consumed ≥ max_tokens?
            Answer: Yes
            ↓ TERMINATED

Final state: TERMINATED
Reason: Budget exhausted
Tokens consumed: 100,000
```

### 5.3 Example 3: Paused and Resumed

```
2:00 PM: Agent spawned
         ↓ PENDING

2:00:01 PM: Activated
            ↓ ACTIVE

2:00:05 PM: Work progresses
            ↓ THINKING

2:00:10 PM: Agent yields
            checkpoint created
            ↓ PAUSED (checkpoint reference: cp-001)

2:00:30 PM: Batch window opens
            Policies re-validated
            Checkpoint restored
            ↓ ACTIVE

2:00:35 PM: Resume work
            ↓ THINKING

2:00:40 PM: Task complete
            ↓ ACTIVE → TERMINATED

Final state: TERMINATED
Reason: Completion after resume
Total execution: 40 minutes (10 min active, 20 min paused, 10 min active)
Tokens consumed: 75,000
```

## 6. Implementation Notes for Substrate Builders

### 6.1 State Storage

Store agent state in substrate with:
```
Session Record:
  - session_id (UUID)
  - current_state (enum: PENDING|ACTIVE|THINKING|PAUSED|TERMINATED)
  - state_change_timestamp (nanoseconds)
  - tokens_consumed (uint64)
  - token_budget (uint64)
  - checkpoint_reference (optional, for PAUSED)
  - governance_policy_snapshot (for re-validation on PAUSED→ACTIVE)
  - creation_timestamp
  - termination_timestamp (if TERMINATED)
  - termination_reason
```

### 6.2 Event Handling

Implement state transitions as event handlers:
```
on_activate_request(session_id):
  session = lookup_session(session_id)
  if session.state != PENDING:
    return ERROR_INVALID_STATE
  session.state = ACTIVE
  log_state_change(session_id, PENDING, ACTIVE)
  return OK

on_begin_thinking_request(session_id):
  session = lookup_session(session_id)
  if session.state != ACTIVE:
    return ERROR_INVALID_STATE
  if session.tokens_consumed >= session.token_budget:
    return ERROR_BUDGET_EXHAUSTED
  session.state = THINKING
  log_state_change(session_id, ACTIVE, THINKING)
  return OK
```

### 6.3 Concurrency

Use locks/transactions to prevent race conditions:
```
All state transitions must be atomic:
  - Read current state
  - Validate transition is legal
  - Update state
  - Log change
  
Use database transaction or distributed lock to ensure atomicity
```
