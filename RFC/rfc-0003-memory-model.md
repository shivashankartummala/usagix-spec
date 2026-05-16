# RFC-0003: Memory Model & Context Virtualization

**Status**: Implemented (v1.0)  
**Authors**: USAGIX Specification Stewards  
**Created**: 2025-03-22  
**Updated**: 2025-05-16

## Summary

This RFC defines the three-tier memory virtualization system (L1/L2/L3) for USAGIX agent contexts. The model enables agents to operate with context windows larger than physical L1 memory by automatically paging inactive context to L2 (warm cache) and L3 (cold storage). This RFC formalizes the paging contracts, memory pressure semantics, and context restoration guarantees.

## 1. Semantic Definitions

### 1.1 Memory Tier Concepts

**L1 Context (Fast Memory)**:
- Active working memory in the cognitive container
- Typically: RAM allocated to agent process
- Access latency: < 1ms
- Size: Limited by available memory (e.g., 128MB–2GB)
- Content: Current token context window + inference state

**L2 Cache (Warm Cache)**:
- Intermediate storage for recently-used context segments
- Typically: Redis, Memcached, or local SSD cache
- Access latency: 1–100ms
- Size: Much larger than L1 (10GB–1TB typical)
- TTL: Minutes to hours

**L3 Cold Storage (Persistent Storage)**:
- Long-term storage for checkpoints and full context snapshots
- Typically: S3, GCS, Azure Blob Storage, or filesystem
- Access latency: 100ms–10s
- Size: Essentially unlimited
- Durability: Replicated, durable across failures

**Context Window**: The set of tokens, system prompt, and inference state available to the model at any point.

**Context Pressure**: A metric (0–100%) indicating how much of L1 memory is occupied relative to the configured L1 size limit.

**Paging Event**: The operation that moves context segments between tiers in response to pressure.

**Eviction Policy**: The algorithm determining which context segments to page to L2/L3 when space is needed (LRU, LFU, FIFO, etc.).

### 1.2 Formal Type System

```
TokenID            = uint64 (position in token stream)
ContextSegment     = {
  id: UUID,
  tokens: List<int32>,
  token_range: Range<TokenID>,
  embedding: Optional<bytes> (for semantic search),
  last_accessed: Timestamp,
  access_count: uint32,
  size_bytes: uint32,
  tier: L1 | L2 | L3
}
MemoryTier         = {
  name: "L1" | "L2" | "L3",
  capacity_bytes: uint64,
  used_bytes: uint64,
  access_latency_ms: float,
  ttl_seconds: uint64 | null,
  eviction_policy: EvictionPolicy
}
ContextSnapshot    = {
  id: UUID,
  session_id: SessionID,
  timestamp: Timestamp,
  total_tokens: uint64,
  segments: List<ContextSegment>,
  l1_segments: List<ContextSegmentID>,
  l2_segment_refs: List<(ContextSegmentID, CacheKey)>,
  l3_segment_refs: List<(ContextSegmentID, S3ObjectKey)>,
  content_hash: bytes32,
  signature: bytes
}
MemoryPressure     = {
  l1_utilization_percent: uint8,
  l2_utilization_percent: uint8,
  l3_utilization_percent: uint8,
  estimated_segment_count: uint32,
  largest_segment_bytes: uint32,
  paging_latency_ms: float
}
EvictionPolicy     = LRU | LFU | FIFO | LIFO | RR
```

## 2. Formal Contracts

### 2.1 Context Availability Guarantee

**Theorem (Transparent Context Virtualization)**: If a session has stored a context segment and has not explicitly deleted it, that segment MUST be available for restoration from any tier (L1→L2→L3) within a bounded time.

**Invariant (Segment Durability)**:
- Segment in L1: Available immediately (< 1ms)
- Segment in L2: Available within T_l2_access seconds (default: 100ms)
- Segment in L3: Available within T_l3_access seconds (default: 10s)
- Segment deleted: Permanently unavailable

**Proof Sketch**:
1. When context segment is created, it's stored in L1
2. When L1 pressure exceeds threshold, segments page to L2 (async)
3. When L2 TTL approaches expiration, segments page to L3
4. L3 is durable (replicated); segments remain available indefinitely
5. On segment access request, substrate traverses L3→L2→L1; returns first match

**Assumptions**:
- L2 cache backend (Redis, etc.) has ≥99.5% availability
- L3 storage (S3, etc.) has ≥99.99% durability
- Network connectivity is available (not in airplane mode)

### 2.2 Paging Contract

**Definition**: Automatic page-out (context → L2 or L3) is triggered when:
```
context_pressure > L1_PRESSURE_THRESHOLD (default: 80%)
```

**Page-Out Preconditions**:
- L1 utilization > threshold
- Eviction policy identifies candidate segments
- Candidate is not currently being accessed (no race condition)

**Page-Out Postconditions**:
1. Segment copied to L2 (atomically)
2. L2 cache key returned to substrate
3. Segment removed from L1 (memory freed)
4. Audit entry: `{type: PAGE_OUT, segment: segment_id, from: L1, to: L2, size: bytes}`

**Page-In Preconditions**:
- Segment reference exists (L2 key or L3 path)
- Segment not already in L1
- L1 has space for incoming segment (or space freed via eviction)

**Page-In Postconditions**:
1. Segment fetched from L2 or L3
2. Segment deserialized and placed in L1
3. Access count and timestamp updated
4. Segment available for inference < T_page_in latency (default: 100ms)
5. Audit entry: `{type: PAGE_IN, segment: segment_id, from: L2_or_L3, to: L1, latency_ms: X}`

**Paging Cost**: Each page-out or page-in operation consumes T_paging_cost tokens (default: 50 tokens).

### 2.3 Memory Pressure Feedback

**Definition**: `GetMemoryPressure(session_id) → MemoryPressure`

**Invariant (Accurate Pressure Reporting)**:
```
reported_l1_percent = (used_bytes / capacity_bytes) * 100
error <= 5% (measurement accuracy requirement)
```

**Guarantee**: Memory pressure is updated at least once per T_pressure_update seconds (default: 1 second).

**Latency SLO**: `GetMemoryPressure` returns within 10ms.

**Usage**: Agent code MAY use pressure feedback to make yielding decisions:
- When pressure > 80%, consider yielding to checkpoint
- When pressure > 95%, strongly recommended to yield
- Substrate may force yield if pressure > 99%

### 2.4 Eviction Policy Contract

**Definition**: When L1 capacity is exceeded, substrate applies the configured eviction policy to select candidates for page-out.

**Policy Definitions**:
- **LRU (Least Recently Used)**: Evict segment with oldest last_accessed timestamp
- **LFU (Least Frequently Used)**: Evict segment with lowest access_count
- **FIFO (First In First Out)**: Evict segment with oldest creation timestamp
- **LIFO (Last In First Out)**: Evict segment with newest creation timestamp
- **RR (Random Replacement)**: Randomly select segment for eviction

**Determinism Guarantee**: Given identical sequence of segment accesses and identical L1 capacity, LRU/LFU/FIFO/LIFO policies MUST evict identical segments.

**Exception**: Hot segments (accessed in last T_hot_window seconds, default: 10s) are NOT eligible for eviction.

### 2.5 L2-to-L3 Promotion Contract

**Trigger**: L2 cache entry approaches TTL expiration
```
time_until_ttl_expiration < L2_TO_L3_PROMOTION_WINDOW (default: 60 seconds)
```

**Preconditions**:
- L2 TTL set (not infinite)
- Segment has been accessed at least once since L2 entry

**Postconditions**:
1. Segment promoted to L3 storage (S3, GCS, etc.)
2. L3 object path stored in session metadata
3. L3 entry is durable (≥3 replicas or equivalent)
4. L2 entry may be dropped after promotion confirmed
5. Audit entry: `{type: L2_TO_L3_PROMOTION, segment: segment_id, s3_key: X}`

**Atomicity**: L2 drop and L3 confirmation are atomic. If system crashes mid-promotion, L2 entry is retained.

## 3. Memory Tier Semantics

### 3.1 L1 Context Allocation

**Initial Allocation**:
- When session transitions from Active to Thinking, L1 context allocated
- Size: min(requested_context_size, available_memory, T_max_l1_context)
- Default: 100K tokens (≈400KB in compressed tokens)

**Resize Semantics**:
- If requested_context_size > available_memory, substrate pages to L2
- Paging is automatic and transparent
- Inference continues unblocked during paging

**Deallocation**:
- When session terminates, L1 immediately freed
- Async process moves any unsaved segments to L2/L3

### 3.2 L2 Cache Lifecycle

**Insertion**:
- Segment paged from L1 or restored from L3
- Key: `l2:{session_id}:{segment_id}`
- TTL: Configured per session (default: 3600 seconds / 1 hour)

**Access**:
- Segment accessed during inference
- Access count incremented
- last_accessed timestamp updated
- TTL reset (if policy = reset-on-access)

**Expiration**:
- If accessed within T_l2_to_l3_promotion_window before TTL, promote to L3
- If NOT accessed before TTL, entry discarded (garbage collected)
- Audit entry: `{type: L2_EXPIRATION, segment: segment_id, reason: NOT_ACCESSED|PROMOTED}`

**Eviction** (under pressure):
- L2 may evict old entries if capacity exceeded
- Policy: Oldest expiration timestamp first (FIFO)

### 3.3 L3 Cold Storage

**Insertion**:
- Segment promoted from L2 or explicit checkpoint
- Path: `s3://bucket/usagix/{tenant}/{session_id}/{checkpoint_id}/{segment_id}.pb`
- Durability: ≥3 replicas (built into S3 standard)

**Retention**:
- Indefinite (no TTL for L3 by default)
- May be garbage collected if session not accessed for T_l3_retention_period (default: 30 days)

**Retrieval**:
- Access: < 10s latency (typical)
- Cost: Network + S3 API calls

**Cleanup**:
- When session terminates, L3 objects marked for deletion
- Actual deletion occurs async (eventual consistency)

## 4. Context Restoration Guarantees

### 4.1 Restore Contracts

**Definition**: `UsagixRestoreContext(session_id, checkpoint_id) → (context, error)`

**Preconditions**:
- Checkpoint exists in audit log
- Checkpoint signature validates
- L1/L2/L3 data accessible

**Postconditions**:
1. All segments in checkpoint_id reconstructed
2. Segments assembled in token order
3. Context hash verified: `SHA256(segments) == checkpoint.context_hash`
4. Context loaded into L1 or L2 as needed
5. Segment access counts and timestamps reset
6. Audit entry: `{type: RESTORE_CONTEXT, checkpoint: checkpoint_id, latency_ms: X}`

**Latency SLO**:
- If all segments in L1: < 1ms
- If segments in L2: < 100ms
- If segments in L3: < 10s

**Partial Failure Handling**:
- If segment retrieval fails: Return error with list of unavailable segments
- Substrate may retry with exponential backoff

### 4.2 Context Consistency Guarantee

**Invariant (Consistency After Restore)**:
```
∀ checkpoint ∈ session.checkpoints:
  hash(restored_context) == checkpoint.context_hash
  AND segment_count(restored_context) == checkpoint.segment_count
  AND SUM(segment_sizes) == checkpoint.total_size_bytes
```

**Verification**: On restore, substrate MUST verify:
1. All segments present and deserializable
2. Total size matches expected value
3. Content hash matches checkpoint
4. Token ranges are contiguous and complete

If verification fails, restore is aborted with error.

## 5. Memory Pressure Management

### 5.1 Pressure Thresholds

| Threshold | L1 Utilization | Action |
|-----------|-----------------|--------|
| Low       | 0–50%          | No action, normal paging |
| Moderate  | 50–80%         | Proactive paging to L2 |
| High      | 80–95%         | Agent should consider yielding |
| Critical  | 95–99%         | Substrate force-yields if > 30s in Thinking |
| Overflow  | > 99%          | Emergency termination signal (SIG_MEMORY_EXHAUSTED) |

### 5.2 Pressure-Based Yielding

Agents MAY query `GetMemoryPressure()` to make intelligent yielding decisions:

```
pressure = substrate.GetMemoryPressure()
if pressure.l1_utilization > 80:
  // Consider checkpointing to free L1
  checkpoint = substrate.UsagixYield(checkpoint_request)
```

**Guarantee**: If agent yields when pressure > 80%, L1 utilization drops to < 50% within 1 second.

### 5.3 Substrate-Forced Yielding

If session remains in Thinking state with L1 pressure > 95% for > 30 seconds:
1. Substrate sends `SIG_MEMORY_PRESSURE` signal
2. Agent has 5 seconds to checkpoint
3. If not checkpointed, substrate force-yields (saves context to L2, pauses session)
4. Session paused; parent notified
5. Audit entry: `{type: FORCED_YIELD, reason: MEMORY_PRESSURE}`

## 6. Mandatory Memory Controls

For a substrate to claim USAGIX v1.0 compliance:

1. **Three-Tier Architecture**: L1, L2, L3 tiers implemented as specified
2. **Automatic Paging**: Page-outs triggered at 80% L1 pressure; page-ins on demand
3. **Segment Durability**: All segments in L3 are durable (replicated)
4. **Eviction Policies**: At least LRU support required; others optional
5. **Memory Pressure Reporting**: `GetMemoryPressure()` accurate within 5%
6. **Context Hash Verification**: Restore MUST verify checksum before using context
7. **Audit Logging**: All paging events logged with segment ID, size, latency

## 7. Example: Memory Lifecycle

```
Timeline:
T0:   Session created
      L1 allocated: 100K tokens (≈400KB)
      L1 utilization: 0%

T1:   Inference begins, tokens consumed
      L1 utilized: 50%
      Memory pressure: 50%

T2:   More tokens consumed
      L1 utilized: 80%
      Memory pressure: 80%
      Substrate proactively pages LRU segment to L2
      L1 utilized: 60% (after paging)

T3:   Inference continues
      L1 utilized: 85%
      Memory pressure: 85%
      Agent queries pressure, sees high utilization

T4:   Agent yields (checkpoint)
      All L1 context paged to L2
      L1 utilized: 5%
      Audit: {type: CHECKPOINT, tokens_consumed: 50000}

T5:   [Parent provides tool response]
      Agent resumes from checkpoint
      Context restored from L2 (< 100ms)
      L1 utilized: 50%

T6:   L2 cache TTL approaches (> 1h since checkpoint)
      Segment promoted to L3 (S3)
      L2 entry dropped
      Audit: {type: L2_TO_L3_PROMOTION, segment: X}

T7:   Session terminates
      L1 freed
      L2 entries garbage collected
      L3 objects marked for deletion (eventual)
```

## 8. Backward Compatibility

USAGIX v1.0 defines L1/L2/L3 semantics. Future versions may:
- Add new eviction policies without breaking changes
- Adjust default thresholds (backward compatible)
- Add new pressure metrics (backward compatible with defaults)

Changes that require v2.0:
- Segment serialization format changes
- Paging protocol changes
- L2/L3 durability guarantees changes

## 9. Normative References

- RFC-0001: USAGIX Core Architecture & Trust Domains
- RFC-0002: Agent Lifecycle & State Machine Semantics
- RFC-0004: Governance Model & Policy Evaluation
- RFC-0005: ASI System Calls (UsagixMemPageOut, UsagixMemPageIn)

## 10. Open Questions & Future Work

- **Q1**: Should L2 cache support TTL-on-access reset, or use absolute TTL? Answer deferred to implementation guide.
- **Q2**: What happens if L3 storage becomes unavailable? Can sessions continue with L2-only? Answer: TBD in fault tolerance RFC.
- **Q3**: Should semantic embeddings be stored alongside segments for similarity-based eviction? Answer deferred to optimization RFC.
- **Q4**: Can agents migrate between substrates with L3 checkpoints? Answer: TBD in federation RFC.

---

**Document Hash**: `sha256(this_document)`  
**Approvals**:
- [ ] TSC Memory Systems Lead
- [ ] TSC Storage Lead
- [ ] Community Review (2-week period)
