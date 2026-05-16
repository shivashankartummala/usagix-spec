# RFC-0007: Inference Scheduler & Token Budget Management

**Status**: Implemented (v1.0)  
**Authors**: USAGIX Specification Stewards  
**Created**: 2025-04-05  
**Updated**: 2025-05-16

## Summary

This RFC defines the inference scheduler, which manages agent token consumption, predicts token burn rate, enforces rate limits, and optimizes scheduling across competing sessions. The scheduler ensures fair allocation of compute resources and prevents token budget surprises through predictive budgeting.

## 1. Semantic Definitions

### 1.1 Scheduling Concepts

**Token Budget**: The total number of tokens (input + output) allocated to a session for its lifetime.

**Token Burn Rate**: The average number of tokens consumed per second during inference. Estimated from history or model calibration.

**Burned Tokens**: Tokens consumed from budget by completed API calls and inference steps.

**Remaining Budget**: allocated_budget - burned_tokens.

**Budget Runway**: remaining_budget / estimated_burn_rate (in seconds). How long until budget exhausted.

**Scheduling Horizon**: The time window into the future for which the scheduler makes decisions (default: 60 seconds).

**Context Pressure Index (CPI)**: A metric (0–100%) combining:
- Memory pressure (L1 utilization)
- Token burn rate (how fast tokens consumed)
- Budget runway (how long until exhausted)

**Rate Limit**: A constraint on requests per time unit (e.g., 100 requests per second to external API).

**Provider Rate Limit**: Rate limits imposed by external tool providers (e.g., OpenAI API: 3,500 RPM).

**Scheduling Decision**: Substrate's choice to allow, queue, or deny an operation based on resource constraints.

### 1.2 Formal Type System

```
TokenBudget = {
  allocated: uint64,
  burned: uint64,
  remaining: uint64,
  burn_rate_estimate: float (tokens/sec),
  runway_seconds: float,
  last_updated: Timestamp
}

BurnRateEstimate = {
  method: Calibration | Historical | Constant,
  tokens_per_inference: uint64,
  tokens_per_tool_invocation: uint64,
  average_tokens_per_second: float,
  confidence: float (0–1),
  last_updated: Timestamp
}

ContextPressureIndex = {
  l1_memory_percent: uint8,
  burn_rate_percent: uint8,  // (current_burn_rate / max_burn_rate) * 100
  budget_runway_percent: uint8, // (runway / scheduling_horizon) * 100
  combined_pressure: uint8,  // weighted average
  recommendation: YieldNow | YieldSoon | Monitor | Relaxed
}

SchedulingDecision = {
  session_id: SessionID,
  operation: Inference | ToolInvocation | Spawn,
  decision: Allow | Queue | Deny,
  reason: string,
  estimated_cost_tokens: uint64,
  estimated_latency_ms: uint32,
  queued_position: uint32 | null
}

ProviderRateLimit = {
  provider: string (e.g., "openai", "anthropic", "google"),
  endpoint: string,
  requests_per_minute: uint32,
  requests_per_second: float,
  concurrent_request_limit: uint32,
  backoff_strategy: Exponential | Linear | Fixed,
  current_usage: UsageSnapshot
}

UsageSnapshot = {
  requests_in_last_minute: uint32,
  requests_in_last_second: uint32,
  concurrent_requests: uint32,
  time_until_limit_reset: uint32 (seconds)
}
```

## 2. Formal Contracts

### 2.1 Token Budget Tracking Contract

**Definition**: `GetTokenBudget(session_id) → TokenBudget`

**Preconditions**:
- Session exists and is active

**Postconditions**:
1. Budget snapshot returned with current state
2. remaining = allocated - burned (invariant maintained)
3. burn_rate_estimate updated using recent history
4. runway_seconds = remaining / (burn_rate_estimate + epsilon)
5. Snapshot accurate within T_budget_accuracy seconds (default: 1 second)

**Accuracy Guarantee**:
```
|reported_burned - actual_burned| ≤ 10 tokens (or 0.1%, whichever is larger)
```

**Atomicity**: Budget read is atomic (no partial updates visible).

### 2.2 Token Deduction Contract

**Definition**: `DeductTokens(session_id, amount: uint64) → (ok: bool, error: string)`

**Preconditions**:
- Session exists
- amount > 0
- amount ≤ remaining_budget

**Postconditions (Success)**:
1. burned_tokens += amount (atomic)
2. remaining_tokens -= amount (atomic)
3. Token counter monotonically increases (never decreases)
4. Audit entry: `{type: TOKEN_DEDUCTION, session: session_id, amount: amount}`

**Postconditions (Failure - Insufficient Budget)**:
1. No tokens deducted
2. Error returned: `{code: BUDGET_EXHAUSTED, remaining: X}`
3. Session transitions to Terminated state
4. Audit entry: `{type: BUDGET_EXHAUSTED, session: session_id, attempted: amount, remaining: X}`

**Monotonicity Guarantee**: burned_tokens never decreases. Once N tokens consumed, total consumed ≥ N forever.

### 2.3 Burn Rate Estimation Contract

**Definition**: `EstimateBurnRate(session_id) → BurnRateEstimate`

**Estimation Methods** (in order of preference):

**1. Calibration Method** (Highest Confidence):
- Query model calibration data for same model_id
- Example: Claude-opus-4-7 averages 2000 tokens per inference
- confidence = 0.95

**2. Historical Method** (Medium Confidence):
- Use this session's recent history (last 10 inferences)
- average_burn_rate = sum(tokens_consumed_in_last_10) / 10
- confidence = 0.7 (varies with history length)

**3. Conservative Method** (Fallback):
- Use 50th percentile from model's typical usage
- confidence = 0.5

**Postconditions**:
1. BurnRateEstimate returned with method and confidence
2. estimate_valid_until = now + T_estimate_ttl (default: 60 seconds)
3. After TTL, re-estimate

**Guarantees**:
- Underestimation is safe (agent will exhaust budget sooner; yields more often)
- Overestimation is risky (agent may not yield in time)
- Substrate errs on side of underestimation (conservative)

### 2.4 Context Pressure Index (CPI) Contract

**Definition**: `ComputeContextPressure(session_id) → ContextPressureIndex`

**CPI Computation**:
```
l1_memory_percent = (l1_used / l1_capacity) * 100

burn_rate_percent = (current_burn_rate / max_sustainable_burn_rate) * 100
  // max_sustainable = min(provider_rate_limit, substrate_capacity)

budget_runway_percent = (runway_seconds / scheduling_horizon_seconds) * 100
  // scheduling_horizon = 60 seconds
  // runway = remaining_tokens / burn_rate

combined_pressure = weighted_average([
  l1_memory_percent * 0.3,
  burn_rate_percent * 0.4,
  budget_runway_percent * 0.3
])
```

**Pressure Thresholds**:
```
combined_pressure ≤ 50%    → "Relaxed"      (no action)
combined_pressure 50–75%   → "Monitor"      (consider yielding)
combined_pressure 75–90%   → "YieldSoon"    (yield within next minute)
combined_pressure > 90%    → "YieldNow"     (yield immediately)
```

**Latency SLO**: CPI computed within 10ms.

**Postconditions**:
1. CPI value returned (0–100%)
2. Recommendation provided (YieldNow/YieldSoon/Monitor/Relaxed)
3. CPI used to make scheduling decisions

### 2.5 Scheduling Decision Contract

**Definition**: `MakeSchedulingDecision(session_id, operation: Inference | ToolInvocation | Spawn, estimated_cost: uint64) → SchedulingDecision`

**Decision Logic**:

```
1. Can operation fit in remaining budget?
   if estimated_cost > remaining_budget:
     return Deny(BUDGET_EXHAUSTED)

2. Are provider rate limits exceeded?
   current_rps = current_requests_per_second
   provider_limit = provider_rate_limit_rps
   if current_rps ≥ provider_limit:
     return Queue(RATE_LIMIT_EXCEEDED, position=queue_length)

3. Is memory pressure critical?
   cpi = ComputeContextPressure(session_id)
   if cpi > 95%:
     return Deny(MEMORY_PRESSURE_CRITICAL)

4. Is budget runway below threshold?
   runway = remaining / burn_rate
   if runway < 5 seconds AND cpi > 75%:
     return Deny(BUDGET_RUNWAY_TOO_SHORT)

5. Default:
   return Allow(latency_estimate)
```

**Postconditions**:
1. SchedulingDecision returned with Allow/Queue/Deny
2. If Allow: Operation proceeds immediately
3. If Queue: Operation added to queue with estimated wait time
4. If Deny: Operation rejected with reason

## 3. Budget Management Strategies

### 3.1 Estimated Budget Allocation

When spawning a child session:
```
parent_remaining = 100000 tokens
child_quota_requested = 50000 tokens

Allocation checks:
1. Is parent_remaining ≥ child_quota + policy_floor?
   100000 ≥ 50000 + 5000? YES

2. Allocate:
   parent_remaining -= 50000 → 50000 remaining
   child_allocated = 50000

3. Child can partition for grandchildren:
   policy_floor inherited (5000 minimum per grandchild)
   If child spawns grandchild with 40000:
     child remaining = 10000
     grandchild allocated = 40000
```

### 3.2 Burn Rate-Based Yielding

Agent can use CPI to decide when to checkpoint:

```
loop {
  cpi = substrate.ComputeContextPressure()
  
  if cpi.recommendation == "YieldNow":
    checkpoint = substrate.UsagixYield(...)
    // Free L1 memory, reset burn rate tracking
  
  else if cpi.recommendation == "YieldSoon":
    continue_reasoning_but_be_aware()
    // Yield within next minute
  
  perform_inference_step()
}
```

### 3.3 Provider Rate Limit Management

When substrate detects provider rate limit:

```
1. Current state: 95 requests/sec; OpenAI limit = 100 requests/min (1.67 req/sec)

2. Scheduler queues excess requests:
   allowed_per_session = rate_limit / num_active_sessions
   
3. Backoff strategy: Exponential with jitter
   retry_delay = base_delay * (2 ^ attempt) + random_jitter
   Example: 10ms, 20ms, 40ms, 80ms, 160ms, 320ms (cap: 30s)

4. Once rate drops below limit:
   drain queue in FIFO order
   resume normal scheduling
```

## 4. Scheduling Dimensions

The scheduler considers multiple dimensions when making decisions:

### 4.1 Temporal Dimension

**Time Horizon**: 60-second window for scheduling decisions.

**Predictions**:
- Will budget run out in next 60 seconds?
- Will memory pressure exceed threshold in next 60 seconds?
- What is the average wait time for queued operations?

### 4.2 Resource Dimension

**Competing Resources**:
- Token budget (per-session, zero-sum across children)
- Memory (L1 pressure, paging latency)
- Compute capacity (substrate CPU/GPU available)
- External API quota (provider rate limits)

**Fair Allocation**:
- Sessions weighted by parent's priority
- Children inherit parent's weight
- Starvation prevention: min allocation per session

### 4.3 Quality Dimension

**Throughput vs. Latency Tradeoff**:
- Allow operation immediately: High throughput, possible queue buildup
- Queue operation: Lower latency variance, potentially higher overall latency
- Deny operation: Prevent overload, request yielding

**Recommendation**: Dynamic policy based on queue depth and provider state.

## 5. Mandatory Scheduling Controls

For a substrate to claim USAGIX v1.0 compliance:

1. **Token Tracking**: Accurate token accounting (within 10 tokens or 0.1%)
2. **Monotonic Counter**: Token consumption never decreases
3. **Budget Enforcement**: Hard limit at substrate level
4. **Burn Rate Estimation**: Reasonable estimates with confidence intervals
5. **CPI Computation**: Accurate context pressure metric
6. **Rate Limit Enforcement**: Provider limits respected
7. **Queue Management**: FIFO ordering for rate-limited operations
8. **Budget Runway Tracking**: Accurate runway estimates
9. **Scheduling Transparency**: Decision reasons logged in audit trail

## 6. Example: Scheduling in Action

```
Timeline:
T0:   Session spawned
      allocated: 100000 tokens
      burn_rate: 500 tokens/sec (estimated from calibration)
      runway: 200 seconds

T5:   Inference running
      burned: 2500 tokens
      remaining: 97500 tokens
      current_burn_rate: 600 tokens/sec (higher than calibration)
      runway: 162 seconds
      CPI: 30% (relaxed)
      Decision: Allow

T10:  cpi = ComputeContextPressure()
      result: CPI=75% (monitor)
      Agent notices and considers checkpointing

T15:  Agent continues inference
      burned: 10000 tokens
      remaining: 90000 tokens
      runway: 150 seconds
      l1_memory: 85% (high)
      CPI: 80% (yield soon)
      
T20:  Agent invokes tool
      estimated_cost: 50 tokens
      scheduling decision:
      - remaining >= cost? YES
      - rate_limit_ok? YES
      - memory_pressure? 80% (ok for now)
      Decision: Allow
      Tool executes, consumes 48 tokens

T25:  burned: 10048 tokens
      remaining: 89952 tokens
      cpi: 82% (yield soon)
      Substrate sends MEMORY_PRESSURE signal
      Agent yields (checkpoint)
      
T26:  Checkpoint created
      L1 freed, memory drops to 20%
      New burn_rate_estimate: 400 tokens/sec
      runway: 225 seconds
      CPI: 15% (relaxed)
      
T30:  [Parent provides tool response]
      Agent resumes from checkpoint
      Inference continues
      CPI: 25% (relaxed)
```

## 7. Backward Compatibility

USAGIX v1.0 defines scheduling model. Future versions may:
- Add new scheduling dimensions (priority levels, QoS classes) without breaking
- Adjust CPI weights (backward compatible)
- Add new burn rate estimation methods

Changes requiring v2.0:
- Breaking changes to budget tracking
- Changes to rate limit semantics
- Changes to scheduling decision semantics

## 8. Normative References

- RFC-0001: USAGIX Core Architecture & Trust Domains
- RFC-0002: Agent Lifecycle & State Machine Semantics
- RFC-0003: Memory Model & Context Virtualization
- RFC-0005: ASI System Calls

## 9. Open Questions & Future Work

- **Q1**: Should the scheduler support multiple priority classes (premium/standard/best-effort)? Answer deferred to QoS RFC.
- **Q2**: Can agents request budget increase mid-session (borrow from pool)? Answer deferred to budget delegation RFC.
- **Q3**: Should burn rate estimation learn from user preferences (aggressive vs. conservative)? Answer deferred to machine learning RFC.
- **Q4**: Can substrate preempt low-priority sessions to free budget for high-priority? Answer: TBD in preemption RFC.

---

**Document Hash**: `sha256(this_document)`  
**Approvals**:
- [ ] TSC Scheduling Lead
- [ ] TSC Resource Management Lead
- [ ] Community Review (2-week period)
