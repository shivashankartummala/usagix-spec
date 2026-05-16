# USAGE Architectural Diagrams

This document contains formal architectural diagrams that visualize the USAGE specification's core concepts, boundaries, and operational flows.

---

## Diagram 1: The USAGE Protocol Stack (Abstraction Layers)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AGENT DEPLOYMENT BOUNDARY                     │
│                   (Language Model + Reasoning Engine)                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  LAYER 4: COGNITIVE APPLICATION                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  • LLM Model Inference                                         │ │
│  │  • Decision Logic & Planning                                   │ │
│  │  • Tool Invocation Requests                                    │ │
│  │  • Memory Management Calls                                     │ │
│  │  • State Serialization (UsageYield)                            │ │
│  │                                                                │ │
│  │  Responsibility: Pure reasoning and semantic decision-making   │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│                     ╔════════════════════════╗                       │
│                     ║   ASI BOUNDARY (v1.0)  ║                       │
│                     ╚════════════════════════╝                       │
│                                                                      │
│  LAYER 3: GOVERNANCE & ASPECT                                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  • Capability Gating                                           │ │
│  │  • Tool Allowlist Enforcement                                  │ │
│  │  • Token Budget Tracking                                       │ │
│  │  • Policy Compliance Checks                                    │ │
│  │  • Quota Enforcement (token limits, concurrency)               │ │
│  │  • Audit Logging                                               │ │
│  │                                                                │ │
│  │  Responsibility: Security policy & resource governance         │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│                     ╔════════════════════════╗                       │
│                     ║  CONTROL PLANE BOUNDARY║                       │
│                     ╚════════════════════════╝                       │
│                                                                      │
│  LAYER 2: RUNTIME & EXECUTION                                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  • State Machine Enforcement (PENDING → ACTIVE → THINKING...)  │ │
│  │  • Checkpoint Management (UsageYield, resumption)              │ │
│  │  • Memory Tier Virtualization (L1/L2/L3 management)            │ │
│  │  • Process Lifecycle (spawn, pause, terminate)                 │ │
│  │  • Signal Handling (SIG_AGENT_TERMINATE, SIG_AGENT_PAUSE)      │ │
│  │  • Tool Invocation Marshaling                                  │ │
│  │                                                                │ │
│  │  Responsibility: Agent execution & resource management         │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│                     ╔════════════════════════╗                       │
│                     ║  SUBSTRATE ABSTRACTION │                       │
│                     ║  LAYER (SAL) BOUNDARY  ║                       │
│                     ╚════════════════════════╝                       │
│                                                                      │
│  LAYER 1: SUBSTRATE ABSTRACTION LAYER (SAL)                         │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  • Process Spawning (OS primitives abstraction)                │ │
│  │  • Compute Allocation & Scheduling                             │ │
│  │  • Storage Layer Bindings (S3, GCS, Redis, local SSD)          │ │
│  │  • Network Isolation & Tool Access Mediation                   │ │
│  │  • OS Signal Delivery                                          │ │
│  │  • Capability Enforcement at kernel boundary                   │ │
│  │                                                                │ │
│  │  Responsibility: Unified interface to any compute substrate    │ │
│  │  (Kubernetes, AWS Lambda, bare metal, edge devices, etc.)      │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      SUBSTRATE IMPLEMENTATIONS                       │
│                 (Platform-Specific, Hidden from Agent)               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  • Kubernetes Pods (L3+ scheduler, cgroups, network policy)          │
│  • AWS Lambda (isolated containers, IAM roles, VPC binding)         │
│  • Myelin-AX (untrusted container interception, sidecar proxy)      │
│  • Azure Container Instances (ACI, managed compute)                 │
│  • Bare Metal (systemd/cgroup isolation, direct OS kernel)          │
│  • Edge Devices (process isolation, resource constraints)           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

KEY INSIGHT:
USAGE standardizes Layers 2-4, allowing agents to execute identically
on any SAL-compliant substrate, just as POSIX allows applications to
run on any POSIX-compliant kernel.
```

---

## Diagram 2: Myelin-AX Pod Topography (Secure Container Isolation)

```
┌────────────────────────────────────────────────────────────────────────┐
│                    KUBERNETES POD BOUNDARY                             │
│                      (Network Namespace)                               │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                                                                  │ │
│  │  CONTAINER 1: agent-brain (UNTRUSTED)                           │ │
│  │  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   │ │
│  │                                                                  │ │
│  │  ╔═══════════════════════════════════════════════════════════╗ │ │
│  │  ║  LLM Model Inference (Claude, GPT, Llama, etc.)           ║ │ │
│  │  ║  - Token generation & reasoning                           ║ │ │
│  │  ║  - Tool invocation calls (gRPC stubs to localhost:50051)  ║ │ │
│  │  ║  - UsageYield for checkpointing                           ║ │ │
│  │  ║  - UsageMemPageOut/In for memory management               ║ │ │
│  │  ╚═══════════════════════════════════════════════════════════╝ │ │
│  │                          │                                      │ │
│  │                          │  gRPC calls to                       │ │
│  │                          │  localhost:50051                     │ │
│  │                          ▼                                      │ │
│  │  ┌─────────────────────────────────────────────────────────┐  │ │
│  │  │  gRPC Client Stubs (UsageSubstrateCore client)           │  │ │
│  │  │  - All outbound RPC bound to 127.0.0.1:50051            │  │ │
│  │  │  - NO OTHER network access permitted                     │  │ │
│  │  │  - Loopback only; no egress to internet/cluster network  │  │ │
│  │  └─────────────────────────────────────────────────────────┘  │ │
│  │                                                                  │ │
│  │  X─────────────────────────────────────────────────────────────X │
│  │  ║                   NO EGRESS WALL                             ║ │
│  │  ║          (Container Network Policy Enforced)                ║ │
│  │  X─────────────────────────────────────────────────────────────X │
│  │                                                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│           ▲                                                           │
│           │  Only loopback gRPC tunnel (localhost:50051)             │
│           │  ALL system calls intercepted                            │
│           │                                                           │
│  ┌────────┴──────────────────────────────────────────────────────────┐ │
│  │                                                                  │ │
│  │  CONTAINER 2: myelin-proxy (TRUSTED SUBSTRATE PROXY)            │ │
│  │  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   │ │
│  │                                                                  │ │
│  │  ╔═══════════════════════════════════════════════════════════╗ │ │
│  │  ║  UsageSubstrateCore gRPC Service (Layer 2 Runtime)        ║ │ │
│  │  ║                                                           ║ │ │
│  │  ║  1. Receive gRPC RPC from agent-brain (50051)            ║ │ │
│  │  ║                                                           ║ │ │
│  │  ║  2. HALT agent computation thread (state preserved)      ║ │ │
│  │  ║                                                           ║ │ │
│  │  ║  3. Extract UsageCallTool requests                       ║ │ │
│  │  ║                                                           ║ │ │
│  │  ║  4. OUT-OF-BAND GOVERNANCE EVALUATION:                   ║ │ │
│  │  ║     ┌──────────────────────────────────────────┐          ║ │ │
│  │  ║     │ Open Policy Agent (OPA) Validation      │          ║ │ │
│  │  ║     │ - Capability gating check                │          ║ │ │
│  │  ║     │ - Tool allowlist verification            │          ║ │ │
│  │  ║     │ - Token quota enforcement                │          ║ │ │
│  │  ║     │ - Quota-exceeded rejection               │          ║ │ │
│  │  ║     └──────────────────────────────────────────┘          ║ │ │
│  │  ║                                                           ║ │ │
│  │  ║  5. SECURITY SCRUBBING:                                  ║ │ │
│  │  ║     ┌──────────────────────────────────────────┐          ║ │ │
│  │  ║     │ Content Filtering & Sterilization        │          ║ │ │
│  │  ║     │ - PII/credential detection & redaction   │          ║ │ │
│  │  ║     │ - Secret scrubbing (API keys, tokens)    │          ║ │ │
│  │  ║     │ - Regex pattern matching for malicious   │          ║ │ │
│  │  ║     │   payloads                               │          ║ │ │
│  │  ║     └──────────────────────────────────────────┘          ║ │ │
│  │  ║                                                           ║ │ │
│  │  ║  6. EXECUTE Tool or REJECT with ToolResponse.err         ║ │ │
│  │  ║                                                           ║ │ │
│  │  ║  7. RESUME agent computation thread with result          ║ │ │
│  │  ║                                                           ║ │ │
│  │  ║  Manages: UsageSpawn, UsageYield, UsageSignal,            ║ │ │
│  │  ║            UsageCallTool, UsageMemPageOut/In              ║ │ │
│  │  ╚═══════════════════════════════════════════════════════════╝ │ │
│  │                          ▲                                      │ │
│  │                          │  Egress to external services         │ │
│  │                          │  (via Kubernetes Network Policy)     │ │
│  │                          │  Example destinations:               │ │
│  │                          │  - S3 (l3-cold-storage)              │ │
│  │                          │  - Redis (l2-cache)                  │ │
│  │                          │  - Tool endpoints (web.search, etc.) │ │
│  │                          │  - Kubernetes API (logging/metrics)  │ │
│  │                          │  - OPA policy server                 │ │
│  │                          │                                      │ │
│  │  ┌─────────────────────────────────────────────────────────┐  │ │
│  │  │  Kubernetes Network Interface                           │  │ │
│  │  │  (eth0 - Pod's network access point)                    │  │ │
│  │  └─────────────────────────────────────────────────────────┘  │ │
│  │                                                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

SECURITY INVARIANT:
agent-brain can ONLY access external systems through the myelin-proxy
gRPC tunnel on localhost:50051. All tool invocations are vetted by
the governance engine (OPA), scrubbed for secrets/PII, and enforced
against the agent's capability set before execution.
```

---

## Diagram 3: The USAGE State Machine (Behavior & Transitions)

```
                           ┌──────────────────┐
                           │     PENDING      │
                           └────────┬─────────┘
                                    │
                        (identity verified,
                         spawn successful)
                                    │
                                    ▼
                           ┌──────────────────┐
                           │     ACTIVE       │
                           └────────┬─────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
         (begin      │ (cold-start   │               │ (fatal error
          inference) │  scheduling)  │               │  SIG_TERMINATE)
                    │               │               │
                    ▼               ▼               ▼
           ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
           │   THINKING   │ │   PAUSED     │ │ TERMINATED   │
           └──────┬───────┘ └──────────────┘ └──────────────┘
                  │ ▲
         (yield)  │ │ (resume)
                  │ │
                  ▼ │
           ┌──────────────────────────┐
           │      PAUSED              │
           │  (state serialized,      │
           │   awaiting action)       │
           └──────────┬───────────────┘
                      │
       ┌──────────────┼──────────────┐
       │              │              │
       │ (SIG_PAUSE)  │ (timeout)    │ (SIG_TERMINATE)
       │ (UsageYield) │              │
       ▼              ▼              ▼
      ┌─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
      │  Illegal Transitions:         │
      │  X Pending → Thinking         │
      │  X Paused → Active            │
      │  X Terminated → (any)         │
      │  (substrate rejects)           │
      └─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘


LEGAL TRANSITIONS (v2.0):
┌──────────────┬──────────────┬──────────────┬─────────────────┐
│ From State   │ Trigger      │ To State     │ Conditions      │
├──────────────┼──────────────┼──────────────┼─────────────────┤
│ PENDING      │ Identity OK  │ ACTIVE       │ Substrate auth  │
│ ACTIVE       │ Begin work   │ THINKING     │ Start inference │
│ ACTIVE       │ Cold-start   │ PAUSED       │ NEW in v2       │
│ THINKING     │ UsageYield   │ PAUSED       │ Checkpoint now  │
│ THINKING     │ Loop         │ THINKING     │ Keep reasoning  │
│ THINKING     │ Token quota  │ TERMINATED   │ Budget exceeded │
│ PAUSED       │ Resume       │ THINKING     │ Restore context │
│ PAUSED       │ Timeout      │ TERMINATED   │ Max wait time   │
│ (any)        │ SIG_TERMINATE│ TERMINATED   │ Halt & cleanup  │
└──────────────┴──────────────┴──────────────┴─────────────────┘

INVARIANT ENFORCEMENT:
• No silent state violations (substrate rejects illegal transitions)
• Token budget monotonicity (consumed ≤ budget always)
• State snapshot consistency (checkpoint header + opaque payload)
• Capability enforcement at tool boundary (UsageCallTool)
• Signal atomicity (state transition is all-or-nothing)
• No concurrent state (one session = one state at a time)
```

---

## Diagram 4: System Call Sequence: Tool Execution Flow

```
TIME ──────────────────────────────────────────────────────────────────►

Agent Process (THINKING)              Myelin Proxy (Substrate)        External Tool
       │                                    │                              │
       │  1. Model inference yields         │                              │
       │     UsageCallTool request          │                              │
       │    (tool: "web.search",            │                              │
       │     args: {query: "..."})          │                              │
       │───────────────────────────────────>│                              │
       │                                    │                              │
       │                                    │ 2. HALT agent thread         │
       │                                    │    (preserve state)          │
       │                                    │                              │
       │                                    │ 3. OUT-OF-BAND OPA CHECK    │
       │                                    │    ┌──────────────────┐      │
       │                                    │    │ Policy Decision  │      │
       │                                    │    │ - Is tool        │      │
       │                                    │    │   in allowed     │      │
       │                                    │    │   list? (ALLOW)  │      │
       │                                    │    │ - Token quota    │      │
       │                                    │    │   sufficient?    │      │
       │                                    │    │   (ALLOW)        │      │
       │                                    │    │ - Security level │      │
       │                                    │    │   OK? (ALLOW)    │      │
       │                                    │    └──────────────────┘      │
       │                                    │                              │
       │                                    │ 4. SECURITY SCRUBBING       │
       │                                    │    ┌──────────────────┐      │
       │                                    │    │ Input Sanitize   │      │
       │                                    │    │ - PII redaction  │      │
       │                                    │    │ - Secret removal │      │
       │                                    │    │ - Malware regex  │      │
       │                                    │    │   check          │      │
       │                                    │    │ → Sterilized     │      │
       │                                    │    │   args JSON      │      │
       │                                    │    └──────────────────┘      │
       │                                    │                              │
       │                                    │ 5. Invoke tool               │
       │                                    │────────────────────────────>│
       │                                    │    (sterilized arguments)   │
       │                                    │                              │
       │                                    │                        (tool
       │                                    │                       execution
       │                                    │                        in progress)
       │                                    │                              │
       │                                    │<─────────────────────────────│
       │                                    │ 6. Tool returns result      │
       │                                    │    {status: success,        │
       │                                    │     data: [...]}            │
       │                                    │                              │
       │                                    │ 7. EGRESS SCRUBBING         │
       │                                    │    ┌──────────────────┐      │
       │                                    │    │ Output Sanitize  │      │
       │                                    │    │ - PII redaction  │      │
       │                                    │    │ - Secret removal │      │
       │                                    │    │   (API keys in   │      │
       │                                    │    │    responses)    │      │
       │                                    │    │ - Malicious      │      │
       │                                    │    │   pattern check  │      │
       │                                    │    │ → Clean JSON     │      │
       │                                    │    └──────────────────┘      │
       │                                    │                              │
       │                                    │ 8. RESUME agent thread      │
       │                                    │    (return from RPC)        │
       │<───────────────────────────────────│                              │
       │ ToolResponse {success: true,       │                              │
       │  result: {data: [...]}}            │                              │
       │                                    │                              │
       │ 9. Agent continues THINKING       │                              │
       │    with tool result                │                              │
       ▼                                    ▼                              ▼


KEY SECURITY BOUNDARIES:

1. AGENT ISOLATION:
   - Agent thread is HALTED during tool execution
   - Agent cannot observe governance validation
   - Agent receives only sanitized results

2. GOVERNANCE ENFORCEMENT (OPA):
   - Capability gating (tool in allowedTools list)
   - Token quota checks (sufficient budget remaining)
   - Security level validation (sandbox restrictions)
   - Atomic decision: ALLOW or DENY (no partial execution)

3. INPUT/OUTPUT SCRUBBING:
   - PII detection & redaction (SSN, email, credit card)
   - Secret detection (API keys, auth tokens, passwords)
   - Malware pattern matching (regex-based attacks)
   - Applied to BOTH agent inputs AND tool outputs

4. TRANSPARENCY:
   - Agent sees only sterilized data
   - Substrate logs all decisions and scrubbing operations
   - Audit trail maintains compliance and accountability
```

---

## Diagram 5: Memory Virtualization & Paging Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    MEMORY TIER ARCHITECTURE (v2.0)                       │
└──────────────────────────────────────────────────────────────────────────┘

                        AGENT COGNITIVE PROCESS
                        (THINKING state)
                               │
                               │
                ┌──────────────┴──────────────┐
                │                             │
                ▼                             ▼
        ┌────────────────┐          ┌────────────────┐
        │  L1 CONTEXT    │          │  MEMORY        │
        │  WINDOW        │          │  OPERATIONS    │
        │  (8K-200K      │          │                │
        │   tokens)      │          │ • UsageMemPageOut
        │                │          │   (evict L1→L2)
        │ • Current      │          │                │
        │   inference    │          │ • UsageMemPageIn
        │   history      │          │   (hydrate L2→L1)
        │ • Active       │          │                │
        │   context      │          │ • UsageYield
        │ • Working set  │          │   (checkpoint) │
        └────────┬───────┘          └────────────────┘
                 │
                 │ (automatic eviction when
                 │  context window fills)
                 ▼
        ┌────────────────┐
        │  L2 WARM CACHE │
        │  (SSD, Redis,  │
        │   pgvector)    │
        │                │
        │ • Recent       │
        │   inference    │
        │   history      │
        │ • Embeddings   │
        │ • Vector db    │
        │ • Semantic     │
        │   indices      │
        └────────┬───────┘
                 │
                 │ (on demand page-in)
                 │ (or automatic cleanup)
                 ▼
        ┌────────────────┐
        │  L3 COLD       │
        │  STORAGE       │
        │  (S3, GCS,     │
        │   persistent)  │
        │                │
        │ • Full session │
        │   history      │
        │ • Checkpoints  │
        │ • Long-term    │
        │   artifacts    │
        │ • Memory      │
        │   observations │
        └────────────────┘


PAGEOUT FLOW (L1 → L2 or L2 → L3):

Agent (in THINKING)
    │
    ├─ Inference reaches token limit in L1
    │
    └─> UsageMemPageOut(
            source=L1,
            target=L2,
            token_payload=<serialized context>
        )
        │
        ├─ Substrate stores payload in L2 cache
        │  (Redis hash: session_id:{chunk_id}
        │   PostgreSQL: pgvector embeddings table
        │   Local SSD: /cache/session/chunk)
        │
        ├─ Compute address_reference
        │  Example: "redis://127.0.0.1:6379/session-abc-chunk-42"
        │
        └─> PageOutResponse {
                address_reference,
                tier_acknowledged: L2,
                storage_timestamp
            }

Agent records reference for later retrieval
│
└─ L1 context window is freed for new inference


PAGEIN FLOW (L2 → L1 or L3 → L1):

Agent needs historical context during THINKING
    │
    ├─ Requests: UsageMemPageIn(
            source=L2,
            target=L1,
            address_reference="redis://127.0.0.1:6379/session-abc-chunk-42"
        )
    │
    ├─ Substrate GOVERNANCE CHECK (OPA):
    │  • Is agent authorized to access this context?
    │  • Does context match agent's session_id?
    │  • Apply PII scrubbing to paged-in data?
    │  • Audit log this access
    │
    ├─ Substrate retrieves payload from L2
    │  (Redis GET, PostgreSQL SELECT, SSD read)
    │
    ├─ Decompress/decrypt if needed
    │
    └─> PageInResponse {
            token_payload=<restored context>,
            tier_retrieved_from=L2,
            retrieval_timestamp,
            retrieval_time_ms=47  // Wall-clock time
        }

Agent inserts context back into L1
│
└─ L1 context window now contains historical context
   Agent can reference/reason over it


YIELD & CHECKPOINT FLOW:

Agent exhausted reasoning capacity
    │
    └─> UsageYield(
            checkpoint={
                header: {
                    format_version: "1.0",
                    total_tokens_consumed: 8472,
                    active_attention_indices: [0,1,2,3,...],
                    current_tool_call_depth: 2,
                    open_capabilities: ["web.search", "db.query"],
                    memory_tier_locators: {
                        L2: "redis://session-abc",
                        L3: "s3://bucket/session-abc/full"
                    },
                    context_hash: "SHA256(...)"
                },
                opaque_payload=<vendor-specific binary>
            },
            tokens_consumed=8472
        )
        │
        ├─ Substrate VALIDATES header
        │  • Version compatible? ✓
        │  • Tokens monotonic? ✓
        │  • Governance metadata present? ✓
        │  • Hash matches payload? ✓
        │
        ├─ Substrate STORES checkpoint durably
        │  (PostgreSQL JSONB, S3, DynamoDB)
        │
        ├─ Substrate MANAGES memory tiers
        │  • Can evict L1 to L2 (safe, header has locators)
        │  • Can garbage-collect old L2 entries (policy-driven)
        │
        └─ Agent → PAUSED state
           (awaits governance decision, tool results, or resumption signal)


CONSISTENCY INVARIANTS:

1. Token Monotonicity: consumed(t) always ≤ quota
2. Address Validity: All address_references from PageOut
   must be retrievable via PageIn (substrate guarantees)
3. Header Integrity: SHA256(opaque_payload) == header.context_hash
   prevents corruption
4. No Data Loss: All paged-out context remains accessible
   (no spontaneous garbage collection of active session data)
5. Governance Enforcement: All PageIn requests validated
   against capability set and audit-logged
```

---

## Summary

These diagrams formalize the USAGE specification's:

1. **Abstraction Layers**: Clean separation of concerns (cognitive, governance, runtime, substrate)
2. **Isolation Boundaries**: Explicit walls showing what is trusted vs. untrusted
3. **State Transitions**: Legal and illegal transitions with enforcement rules
4. **Security Flows**: Step-by-step tool execution with governance & scrubbing
5. **Memory Management**: Three-tier virtualization with demand paging

Each diagram emphasizes the standardization boundary (USAGE ASI) that enables agents to execute identically across heterogeneous substrates while maintaining strict security and governance guarantees.
