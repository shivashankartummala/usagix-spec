# RFC-0008: Multi-Agent Coordination & Session Hierarchies

**Status**: Implemented (v1.0)  
**Authors**: USAGIX Specification Stewards  
**Created**: 2025-04-10  
**Updated**: 2025-05-16

## Summary

This RFC formalizes the multi-agent model where parent agents spawn child agents for subtasks. It defines session hierarchies, budget partitioning guarantees, failure semantics, and deadlock prevention. Multi-agent coordination enables agents to delegate work while maintaining strict governance and resource isolation.

## 1. Semantic Definitions

### 1.1 Multi-Agent Concepts

**Agent Hierarchy**: A tree structure where parent agents spawn child agents. Root agent spawned by user/orchestrator.

**Parent Session**: The session that invoked UsagixSpawn to create a child.

**Child Session**: A session created by a parent via UsagixSpawn. Has parent_session_id set.

**Sibling Sessions**: Child sessions sharing the same parent.

**Ancestor/Descendant**: Transitive relationships in the hierarchy.

**Process Tree**: The complete hierarchy rooted at the initial agent invocation.

**Budget Partitioning**: The process of allocating parent budget to children such that total allocated ≤ parent's allocated.

**Spawn Semantics**: The rules governing when and how children can be created.

**Failure Cascade**: When a parent fails, all descendants terminate automatically.

**Deadlock**: Circular dependency where agent A waits for agent B, and B waits for A.

**Loop**: A child spawns parent (creating a cycle) or an agent spawns itself recursively.

### 1.2 Formal Type System

```
SessionTree = {
  root: SessionID,
  nodes: Map<SessionID, SessionNode>,
  edges: List<(parent_id, child_id)>
}

SessionNode = {
  id: SessionID,
  parent_id: SessionID | null,
  child_ids: Set<SessionID>,
  status: Pending | Active | Thinking | Paused | Terminated,
  budget: {
    allocated: uint64,
    burned: uint64,
    distributed_to_children: uint64
  },
  created_at: Timestamp,
  terminated_at: Timestamp | null,
  termination_reason: string | null
}

SpawnContext = {
  parent_session_id: SessionID,
  parent_budget_remaining: uint64,
  parent_policy_floor: uint64,
  requested_child_budget: uint64,
  child_config: ChildConfig
}

FailureMode = {
  session_id: SessionID,
  type: Timeout | BudgetExhausted | PolicyViolation | UnrecoverableError,
  timestamp: Timestamp,
  cascading_descendants: List<SessionID>
}

CoordinationConstraint = {
  type: MaxChildren | MaxDepth | MaxConcurrentSessions | BudgetFloor,
  limit: uint32 | uint64,
  current_value: uint32 | uint64,
  enforced_at: CoordinationLevel
}
```

## 2. Formal Contracts

### 2.1 Session Hierarchy Contract

**Theorem (Acyclic Tree Property)**: The session hierarchy MUST form a directed acyclic graph (DAG). No session can be its own ancestor.

**Invariant (No Cycles)**:
```
∀ session S:
  S ∉ ancestors(S)
```

**Proof Sketch**:
1. Only UsagixSpawn can create parent-child edges
2. UsagixSpawn requires parent_session_id ≠ child_session_id
3. UsagixSpawn verifies child_session_id is fresh (not existing)
4. Therefore, forward edges only (no cycles possible)

**Enforcement**: Substrate MUST prevent creation of cycles:
- Reject UsagixSpawn(parent, child) if parent would become descendant of child
- Check: ¬(parent ∈ descendants(child))

### 2.2 Budget Partitioning Contract

**Theorem (Budget Conservation)**: The sum of all children's allocated budgets MUST NOT exceed the parent's remaining budget.

**Invariant (No Overspending)**:
```
∀ session P (parent):
  SUM(child.allocated for child ∈ children(P)) ≤ P.remaining_budget_at_spawn_time
  
∀ session P (parent):
  P.distributed_to_children ≤ P.allocated
```

**Proof Sketch**:
1. When parent spawns child with budget Q:
   - Check: P.remaining ≥ Q + policy_floor
   - If check passes: P.remaining -= Q (atomic deduction)
2. Budget deduction is atomic; no race conditions
3. Sum of deductions ≤ initial allocated

**Cascading Policy Floor**:
```
∀ parent P, child C:
  C.policy_floor ≥ P.policy_floor
```

This ensures grandchildren inherit at least the same policy floor as their parent (prevents starvation).

### 2.3 Spawn Contract (Detailed)

**Definition**: `UsagixSpawn(parent_session_id, child_config, budget_quota) → (child_session_id, error)`

**Preconditions**:
1. parent_session_id exists and is in Active or Thinking state
2. parent.remaining_budget ≥ budget_quota + parent.policy_floor
3. child_config is valid (model_id, system_prompt, timeout)
4. parent is not at MaxChildren limit
5. Process tree depth ≤ MaxDepth (default: 10)
6. Total concurrent sessions ≤ MaxConcurrentSessions (default: 1000)
7. No cycle would be created: parent ∉ descendants(new_child)

**Postconditions (Success)**:
1. New session created: child_session_id = UUID()
2. SessionNode created: {id, parent_id, status=Pending, budget}
3. Parent's edge: parent.child_ids.add(child_session_id)
4. Parent's budget: remaining -= budget_quota (atomic)
5. Audit: {type: SPAWN, parent: parent_id, child: child_id, budget: quota}
6. Child transitions to Active within T_admission (5s)

**Postconditions (Failure)**:
1. No session created
2. Parent budget unchanged
3. Error returned: {code, reason}
4. Audit: {type: SPAWN_DENIED, parent: parent_id, reason: error_code}

**Errors**:
- `PERMISSION_DENIED`: Budget check failed
- `RESOURCE_EXHAUSTED`: Process tree depth or concurrency limit exceeded
- `FAILED_PRECONDITION`: Parent not in valid state for spawning

**MaxChildren Limit**: Default = 100 (per parent). Prevents explosion of process tree.

**MaxDepth Limit**: Default = 10 levels. Prevents deep recursion-like structures.

### 2.4 Failure Cascade Contract

**Definition**: When a session terminates (gracefully or forcibly), all descendant sessions receive cascading termination.

**Cascade Process**:
```
1. Parent session terminates (reason: timeout, budget, policy, user cancel)
   → status = Terminated

2. For each child in parent.child_ids:
   → Send PARENT_TERMINATED signal
   → Child has T_cascade_grace_period (5s) to clean up
   → If not terminated: Force terminate

3. Recursively cascade to grandchildren, great-grandchildren, etc.

4. All resources reclaimed in cascade order (bottom-up)
```

**Invariant (Guaranteed Cascade)**:
```
If parent.status == Terminated at time T:
  ∀ child ∈ descendants(parent):
    child.status == Terminated by time T + T_cascade_grace_period
```

**Cascade Timeout**: If cascade takes > T_cascade_timeout (default: 30s), escalate to operator.

**Audit Trail**: Each cascade termination logged:
```
{
  type: CASCADE_TERMINATION,
  parent_session: P,
  child_session: C,
  reason: parent_terminated,
  timestamp: T
}
```

### 2.5 Parent-Child Communication Contract

**Definition**: Parent and child communicate via tool invocations and tool responses.

**Parent Perspective**:
- Invokes child session (spawn)
- Sends tool responses to child (resume from checkpoint)
- Receives child results via audit log queries
- Monitors child status and budget consumption

**Child Perspective**:
- Receives parent's tool responses (inputs to next inference step)
- Can query parent's remaining budget (for resource planning)
- Cannot directly access parent's context

**Message Passing**:
- Asynchronous: Parent spawns child; child runs independently
- Synchronous (optional): Parent yields, waiting for child to complete
  
**Example (Asynchronous)**:
```
T0: Parent spawns Child-A with budget 10K tokens
T1: Parent continues own work
T2: Child-A runs in parallel, consumes budget
T3: Parent checks Child-A status (optional)
T4: Parent and Child-A both complete independently
```

**Example (Synchronous)**:
```
T0: Parent spawns Child-B with budget 5K tokens
T1: Parent yields (checkpoint), waiting for Child-B result
T2: Child-B runs, produces result, terminates
T3: Parent resumes from checkpoint, receives result
T4: Parent continues work using Child-B's result
```

### 2.6 Deadlock Prevention Contract

**Potential Deadlock Scenario**:
```
Agent A spawns Agent B
Agent B tries to invoke Parent (A) via UsagixCallTool
Agent A waiting for B to complete
Agent B waiting for A to respond
→ Circular dependency → Deadlock
```

**Prevention Strategy**:

**1. No Direct Parent Calls**:
- Child cannot invoke parent via UsagixCallTool
- Only parent can invoke child (asymmetric)

**2. Tool-Based Communication**:
- If child needs parent's state, child invokes external tool
- Tool queries parent's state asynchronously
- Avoids direct RPC cycle

**3. Timeout Enforcement**:
- All operations have hard timeouts
- If child yields waiting for parent response > T_parent_response_timeout (30s):
  - Child terminates with error
  - Prevents indefinite waiting

**4. Audit Detection**:
- Substrate monitors for circular dependencies in audit logs
- If detected: Escalate alert; consider forcible termination

**Invariant (No Deadlock)**:
```
If parent P spawned child C:
  C cannot directly invoke P via UsagixCallTool
  C can only receive inputs via UsagixResume
```

### 2.7 Loop Prevention Contract

**Potential Loop Scenario 1**: Recursive self-spawn
```
Agent A spawns Agent A (same agent logic)
→ Infinite recursion → Runaway process tree
```

**Potential Loop Scenario 2**: Indirect recursion
```
Agent A spawns Agent B
Agent B spawns Agent A (same logic)
→ A→B→A→B... → Cycle
```

**Prevention**:

**1. Tree Structure Enforcement** (Acyclic):
- Substrate prevents cycles during spawn
- Check: new_child ∉ ancestors(parent)

**2. Budget Depletion**:
- Each spawn deducts budget from parent
- Recursive spawns quickly exhaust budget
- Eventually spawn fails: BUDGET_EXHAUSTED

**3. Depth Limit** (MaxDepth = 10):
- After 10 levels of spawning, further spawns denied
- Prevents deep recursion

**4. Logical Loop Detection** (Future):
- Substrate could analyze agent logic (code/prompts) to detect intended loops
- Defer to future RFC

**Residual Loop Risk**: Agent could spawn many children and never terminate them (zombie processes). Mitigation: Parent failure cascade terminates all children.

## 3. Coordination Patterns

### 3.1 Work Distribution Pattern

Parent spawns multiple children for parallel work:

```
Parent
├─ Child-1: Process subset A
├─ Child-2: Process subset B
├─ Child-3: Process subset C
└─ Child-4: Aggregate results

Budget allocation:
- Parent: 100K tokens
- Child-1: 25K tokens
- Child-2: 25K tokens
- Child-3: 25K tokens
- Child-4: 20K tokens (for aggregation)
```

### 3.2 Hierarchical Refinement Pattern

Multi-level hierarchy for successive refinement:

```
Level 0: Root Agent (budget: 100K)
├─ Level 1: Sub-task A (budget: 50K)
│  ├─ Level 2: Detail A1 (budget: 25K)
│  └─ Level 2: Detail A2 (budget: 25K)
└─ Level 1: Sub-task B (budget: 50K)
   └─ Level 2: Detail B1 (budget: 50K)
```

### 3.3 Escalation Pattern

Child hits policy violation → escalates to parent for approval:

```
Child tries to invoke database.write
Policy requires approval
Child fails → reports to parent
Parent submits approval request (human)
Parent resumes child with approval grant
Child retries database.write → success
```

## 4. Mandatory Multi-Agent Controls

For a substrate to claim USAGIX v1.0 compliance:

1. **Acyclic Hierarchy**: Tree structure enforced; no cycles allowed
2. **Budget Partitioning**: Sum of children's budgets ≤ parent's allocated
3. **Policy Floor Inheritance**: Children ≥ parent's policy floor
4. **Failure Cascade**: Parent termination triggers child termination
5. **Spawn Limits**: MaxChildren, MaxDepth, MaxConcurrentSessions enforced
6. **Timeout Enforcement**: All operations have hard timeouts
7. **Deadlock Prevention**: No direct parent-child RPC cycles
8. **Loop Prevention**: Cycles prevented at spawn time
9. **Audit Completeness**: All spawn/cascade events logged
10. **Resource Cleanup**: Cascaded terminations reclaim resources

## 5. Example: Multi-Agent Workflow

```
Timeline:
T0:   User invokes Root Agent
      Session: Root (budget: 1M tokens)
      Status: Pending → Active

T1:   Root analyzes task
      Decision: Parallelize into 3 subtasks
      Root spawns 3 children:
      - Child-1: Budget 300K tokens
      - Child-2: Budget 300K tokens
      - Child-3: Budget 300K tokens
      Root remaining: 100K tokens (for aggregation)
      Audit: {type: SPAWN_CHILD_1}, {type: SPAWN_CHILD_2}, {type: SPAWN_CHILD_3}

T2:   Children execute in parallel
      Child-1: 50K tokens consumed (250K remaining)
      Child-2: 100K tokens consumed (200K remaining)
      Child-3: 75K tokens consumed (225K remaining)
      Root: 10K tokens consumed (90K remaining)

T3:   Child-2 hits policy violation (unauthorized API call)
      Child-2 yields, requests approval
      Root receives approval request
      Root human-in-loop: Denies request
      Child-2 receives denial
      Child-2 terminates with error
      Audit: {type: POLICY_DENIED, session: Child-2, action: api_call}

T4:   Child-1 and Child-3 complete successfully
      Child-1: Total consumed 280K tokens, result produced
      Child-3: Total consumed 300K tokens, result produced
      Both children terminate gracefully

T5:   Root receives results from Child-1 and Child-3
      Root aggregates results
      Root spawns Child-4 (budget: 50K) for final aggregation
      Child-4 executes and completes

T6:   Root produces final output
      Root terminates
      Audit: {type: TERMINATE, session: Root, reason: complete, total_tokens: 540K}
      
T7:   Cascade cleanup
      All children already terminated
      Resources freed
```

## 6. Backward Compatibility

USAGIX v1.0 defines multi-agent model. Future versions may:
- Add new spawn constraints (per-region limits, etc.) without breaking
- Add new communication patterns (direct RPC, message queues) while supporting v1.0
- Add new coordination features (transactions, consensus)

Changes requiring v2.0:
- Breaking changes to session tree structure
- Changes to budget partitioning guarantees
- Changes to failure cascade semantics

## 7. Normative References

- RFC-0001: USAGIX Core Architecture & Trust Domains
- RFC-0002: Agent Lifecycle & State Machine Semantics
- RFC-0004: Governance Model & Policy Evaluation
- RFC-0007: Inference Scheduler & Token Budget Management
- OWASP Distributed Systems Security — Design patterns

## 8. Open Questions & Future Work

- **Q1**: Should child agents run on different substrates (federated)? Answer deferred to federation RFC.
- **Q2**: Should children be able to request budget increases from parent? Answer deferred to budget negotiation RFC.
- **Q3**: Can parents and children transact (atomic multi-agent operations)? Answer deferred to transactions RFC.
- **Q4**: Should agents be able to directly RPC siblings (not through parent)? Answer: TBD in peer-to-peer coordination RFC.
- **Q5**: What happens if parent is in Paused state when child is spawned? Answer deferred to advanced lifecycle RFC.

---

**Document Hash**: `sha256(this_document)`  
**Approvals**:
- [ ] TSC Architecture Lead
- [ ] TSC Coordination Lead
- [ ] Community Review (2-week period)
