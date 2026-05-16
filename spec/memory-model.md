# Memory Virtualization Model Specification

USAGE implements a **cognitive memory tier system** that differs fundamentally from traditional virtual memory (pages, DRAM, swap). This document specifies how agent context is managed across three tiers: active window (L1), warm cache (L2), and cold storage (L3).

## 1. Core Principle: Virtual Memory for Cognitive Systems

Traditional operating systems manage **physical memory**: DRAM pages are swapped to disk when memory pressure occurs. The OS is agnostic to content; all data is equally expensive to swap.

**Cognitive memory is different**:
- Context is not homogeneous (some parts are critical, some can be compressed)
- Retrieval cost is **domain-dependent** (L1→L2 is 10ms, L1→L3 is 100ms)
- Governance constraints apply to memory (agent can't access unauthorized context)
- Memory tiers have different **cost characteristics**: L1 is in-process (free), L2 is cache hits ($0.001/request), L3 is S3 gets ($0.0001/request + transfer)

USAGE treats context as a **managed resource** with explicit tiering, not a side-effect of physical memory pressure.

## 2. Three-Tier Memory System

### 2.1 L1: Active Context Window (Hot)

**Location**: In-process memory, agent's working set
**Latency**: <1ms (no fetch required, already in memory)
**Capacity**: Configurable per agent (default: 128K tokens)
**Characteristics**:
- Agent can reference any index within this window
- Includes current conversation, relevant facts, active tools
- Governed by `attention_window_indices` in governance policy
- Subject to `context_hash` integrity verification

**Management**:
```
Agent Context Window (L1):
┌─────────────────────────────┐
│ [0-50K tokens]              │
│ Current conversation        │
├─────────────────────────────┤
│ [50K-100K tokens]           │
│ Retrieved facts, tool outputs
├─────────────────────────────┤
│ [100K-128K tokens]          │
│ Working memory, intermediate results
└─────────────────────────────┘

When context fills:
1. Agent requests PageOut for oldest conversation chunk
2. Substrate evicts to L2
3. L1 now has free space for new content
```

### 2.2 L2: Warm Semantic Cache (Cached)

**Location**: Redis, Memcached, or similar warm cache
**Latency**: 5-50ms (network round-trip to cache)
**Capacity**: Unbounded (practical limit: 10GB per agent)
**Characteristics**:
- Recent evictions from L1
- Can be quickly retrieved (sub-100ms)
- Not guaranteed to persist (cache eviction possible)
- Cost: ~$0.001 per cache hit
- TTL: 24 hours (evicted if not accessed)

**Data in L2**:
```
Example L2 cache keys:
- redis://cache.internal/ag-xxx/conversation-chunk-1
- redis://cache.internal/ag-xxx/tool-output-search-results
- redis://cache.internal/ag-xxx/fact-database-query-results
```

**Governance constraints apply to L2**:
- Before rehydrating L2→L1, substrate re-validates agent's access
- If policy changed (agent lost CONFIDENTIAL access), rehydration may be denied/degraded
- PII and secrets in L2 are encrypted at rest

### 2.3 L3: Cold Persistent Store (Durable)

**Location**: S3, GCS, Azure Blob Storage, or similar object store
**Latency**: 50-500ms (object fetch + transfer)
**Capacity**: Unbounded (practical limit: 1TB per agent)
**Characteristics**:
- Long-term memory (conversations from hours/days ago)
- Durable (survives substrate restarts)
- Encrypted at rest (mandatory)
- Cost: ~$0.0001 per retrieval + $0.000000125 per GB-month storage
- TTL: Configurable (default: 90 days for INTERNAL data, 30 days for PUBLIC, 365 days for CONFIDENTIAL)

**Data in L3**:
```
Example L3 storage paths:
- s3://agent-cold-storage/ag-xxx/conversation-2025-03-15.bin.enc
- s3://agent-cold-storage/ag-xxx/checkpoint-2025-03-15-14-23.bin.enc
- s3://agent-cold-storage/ag-xxx/facts-archive-2025-01-to-02.bin.enc
```

**Governance constraints are STRICT for L3**:
- Encryption mandatory (CTR-256 with HMAC-SHA256)
- Access logs mandatory (who retrieved what, when)
- Data classification labels required (PUBLIC/INTERNAL/CONFIDENTIAL)
- PII must be separately encrypted (can be revoked independently)
- Retention policy by classification (shorter TTL for sensitive data)

## 3. Memory Tier Transitions

### 3.1 L1 → L2: PageOut (Eviction)

**Trigger**: Agent requests PageOut (voluntary) or context window reaches capacity (involuntary)

```
Agent's L1 context fills to 128K tokens
    ↓
Agent calls UsageMemPageOut(
  pages_to_evict=[PageReference(page_id=1, size=50K)],
  target_tier=L2
)
    ↓
Substrate validates:
  - Page is marked evictable (not currently in-use)
  - Data classification permits L2 storage
  - No active locks on this page
    ↓
Serialize page content
    ↓
Apply encryption if data_classification > PUBLIC
    ↓
Write to L2 (Redis key: redis://cache.internal/ag-xxx/page-1)
    ↓
Return reference: "redis://cache.internal/ag-xxx/page-1"
    ↓
Agent stores reference in its L1 working memory
```

**PageOut Metadata**:
```protobuf
message PageOutMetadata {
  int32 page_id = 1;
  int64 eviction_timestamp = 2;
  int32 page_size_bytes = 3;
  string data_classification = 4;           // PUBLIC, INTERNAL, CONFIDENTIAL
  bool contains_pii = 5;
  string page_hash = 6;                     // SHA-256 for integrity
  int64 ttl_seconds = 7;                    // When to evict from L2
  string storage_reference = 8;             // redis:// or s3:// URI
  map<string, string> governance_labels = 9;  // Policy labels
}
```

### 3.2 L2/L3 → L1: PageIn (Rehydration)

**Trigger**: Agent requests specific page by reference, substrate retrieves and reloads into L1

```
Agent references: redis://cache.internal/ag-xxx/page-1
    ↓
Agent calls UsageMemPageIn(
  page_reference="redis://cache.internal/ag-xxx/page-1"
)
    ↓
Substrate validates:
  - Reference format is valid
  - Page exists in cache/storage
  - Page was evicted by THIS agent (not another agent)
    ↓
Governance Re-validation:
  - Check if agent still has access to this page's data_classification
  - If policy changed and agent lost access → return POLICY_DENIED
  - If policy changed but agent has lesser access → return DEGRADED (read-only)
  - If policy unchanged → return APPROVED
    ↓
If approval:
  Fetch page from L2/L3
  Decrypt if encrypted
  Validate integrity (SHA-256 hash check)
  Load into L1
  Update page access time (for L2 LRU eviction)
    ↓
Return page content
```

**PageIn Policy Re-Validation Example**:

```
3 hours ago: Agent's capability = {INTERNAL, CONFIDENTIAL access}
             Context page evicted to L3, classified as CONFIDENTIAL

Now: Agent tries to PageIn that page
     Substrate re-validates governance
     
Policy changed 1 hour ago: Agent now only has {INTERNAL} access (CONFIDENTIAL revoked)

Response: policy_revalidation_status="DEGRADED"
          page_data is returned (backward compatibility)
          but marked read-only
          audit log: "Agent accessed CONFIDENTIAL data despite revoked access"
```

## 4. Memory Tier Addressing

### 4.1 L1 Addressing

Agent references L1 content by **index** (position in context window):

```
Agent's L1 context:
[0-----50K-----100K-----128K]
Start: 0
Size:  128K tokens

References:
- Attention Window: indices [0-50K] = current conversation
- Tool Outputs:    indices [50K-100K] = recent facts
- Working Memory:  indices [100K-128K] = intermediate state

Index validation:
- Agent requests data at index N
- Substrate checks: is N in active_attention_window_indices?
- If yes → allow access
- If no → deny access (governance control)
```

### 4.2 L2/L3 Addressing

Agent references L2/L3 content by **URI** (storage reference returned by PageOut):

```
L2 format: redis://cache.internal:6379/ag-xxx/page-1
L3 format: s3://agent-cold-storage/ag-xxx/conversation-chunk-1.bin.enc

URI structure:
  scheme://host/path
  
Valid schemes: redis, s3, gcs, blob (Azure), http/https (external storage)
Validation: Substrate verifies URI is within allowed domains (no SSRF)
```

## 5. Consistency Invariants

USAGE memory system must maintain these invariants:

### 5.1 Invariant 1: Single-Writer, Multiple-Reader

Only the owning agent can write to its memory. Only the owning agent can read (via PageIn).

```
Agent-A Context ⊄ Agent-B (no cross-agent reads)
Agent-A cannot modify Agent-B's L3 checkpoints
Substrate enforces: session_id must match page_owner before allow PageIn/PageOut
```

### 5.2 Invariant 2: Monotonic Token Count

Token count across PageOut/PageIn cycles must never decrease:

```
tokens_consumed at eviction: 100,000
    ↓ (page evicted to L3, returned later)
    ↓
tokens_consumed at rehydration: >= 100,000 (never < 100,000)

Substrate enforcement:
- Log tokens_consumed at eviction time
- Log tokens_consumed at rehydration time
- Verify monotonicity; if violated → ABORT and alert
```

### 5.3 Invariant 3: Governance Consistency

Data classification and access permissions must be consistent across tiers:

```
Data classified as CONFIDENTIAL at eviction to L3
    ↓
3 hours later, policy changes: agent loses CONFIDENTIAL access
    ↓
Agent tries to PageIn same data
    ↓
Substrate detects inconsistency
    ↓
Response: POLICY_DENIED (strict) or DEGRADED (lenient, for backward compat)
    ↓
Audit log: "Governance inconsistency detected during rehydration"
```

### 5.4 Invariant 4: Integrity Verification

Pages must be verifiable for tampering (SHA-256 hash checked on rehydration):

```
Page evicted with hash: sha256:abc123def456...
    ↓ (page sits in L3 for hours)
    ↓
Agent requests PageIn
    ↓
Substrate fetches page from L3, computes SHA-256
    ↓
If computed hash != original hash:
  → return ERROR_INTEGRITY_VIOLATION
  → abort rehydration
  → alert security team
  → do NOT load corrupted data into agent context
```

### 5.5 Invariant 5: Checkpoint Alignment

When agent yields (UsageYield), checkpoint header must include memory metadata:

```protobuf
message CheckpointHeader {
  // ... other fields ...
  map<int32, string> memory_tier_locators = 7;
  // Example: {1: "redis://cache.internal/ag-xxx/page-1",
  //           2: "s3://agent-cold-storage/ag-xxx/page-2.bin.enc"}
}
```

This allows substrate to restore agent state including all evicted context.

## 6. Virtual Memory Control Loop

USAGE implements a **bidirectional control loop** for memory management:

### 6.1 PageOut Control Loop (L1 → L2/L3)

```
Agent's available L1 space < threshold?
    ↓
   YES → Agent initiates PageOut
    ↓
Substrate validates eviction request
    ↓
Evict pages to L2/L3
    ↓
Return storage references
    ↓
Agent stores references in L1
    ↓
L1 space freed; agent continues
```

### 6.2 PageIn Control Loop (L2/L3 → L1)

```
Agent wants to reference old context (not in L1)
    ↓
Agent calls UsageMemPageIn(reference)
    ↓
Substrate revalidates governance
    ↓
Fetch from L2/L3
    ↓
Load into L1
    ↓
Agent can now reference it
```

### 6.3 Full Cycle Example: Long-Running Agent

```
2:00 PM: Agent spawned, L1 = 128K token capacity
         [current_conversation (50K), working_memory (78K)]

2:30 PM: After many interactions, L1 nearly full
         Agent calls PageOut([current_conversation (50K)] → L2)
         Returns reference: redis://...page-1

2:45 PM: Need to remember conversation from 2:00 PM
         Agent calls PageIn(redis://...page-1)
         Substrate revalidates: still have access? yes
         L1 rehydrates with conversation data

3:00 PM: Token budget running low, need historical context
         Agent calls PageOut([working_memory (50K)] → L3)
         Returns reference: s3://...checkpoint-1.bin.enc

3:15 PM: Need historical fact from 3:00 PM checkpoint
         Agent calls PageIn(s3://...checkpoint-1.bin.enc)
         Substrate revalidates: still have access?
         Policy changed → some data marked read-only
         Rehydrate with degraded permissions

3:30 PM: Token budget exhausted
         Agent terminates
         Checkpoint created with all memory references
         Later, agent can be restored from checkpoint
```

## 7. Memory Tier Configuration

Each agent specifies memory tier preferences in UsageSpec:

```protobuf
message UsageMemorySpec {
  MemoryTierConfig l1_hot_context = 1;
  MemoryTierConfig l2_warm_cache = 2;
  MemoryTierConfig l3_cold_storage = 3;
}

message MemoryTierConfig {
  int32 max_size_tokens = 1;                 // Max size for this tier
  int64 ttl_seconds = 2;                     // How long to keep data
  string storage_backend = 3;                // redis, s3, gcs, etc.
  bool enable_encryption = 4;                // Encrypt sensitive data
  bool enable_compression = 5;               // Compress for storage
  map<string, string> backend_config = 6;    // Backend-specific config
}
```

Example config for long-running agent:
```yaml
memory:
  l1:
    max_size_tokens: 256000        # Large window for multi-turn
    ttl_seconds: 3600              # Keep in active memory for 1 hour
  l2:
    max_size_tokens: 5000000       # Can hold lots in warm cache
    ttl_seconds: 86400             # Keep in cache for 24 hours
    storage_backend: redis
    enable_compression: true
  l3:
    max_size_tokens: unlimited
    ttl_seconds: 2592000           # Keep in cold storage for 30 days
    storage_backend: s3
    enable_encryption: true
```

## 8. Cost Model for Memory Operations

Substrates should track and report memory costs:

| Operation | Cost | Notes |
|-----------|------|-------|
| **L1 access** | $0 | Already in process memory |
| **L1→L2 eviction** | $0 | Local operation, no external call |
| **L2 cache hit** | $0.001 per 1GB | Typical Redis/Memcached cost |
| **L2→L3 move** | Included in L3 write | Move is implemented as write + delete |
| **L3 write** | $0.000000125 per GB-month | S3 standard storage cost |
| **L3 read** | $0.0001 per 1GB | S3 request cost |
| **L3 retrieval (L3→L1)** | $0.0001 per 1GB | S3 data transfer + processing |

Substrate should emit metrics:
```
memory.l1.usage_tokens
memory.l2.cache_hits
memory.l2.cache_misses
memory.l3.retrieval_latency_ms
memory.tier_transition_count
memory.total_cost_usd
```
