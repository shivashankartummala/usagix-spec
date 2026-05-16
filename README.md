# USAGE: Universal Substrate for Agent Governance Enforcement

## Executive Summary
USAGE is an open interface specification for executing autonomous agent processes under strict substrate governance. It standardizes the boundary between cognitive workloads and execution substrates, analogous to the role of POSIX between applications and operating systems. USAGE defines control-plane and data-plane contracts for lifecycle, signaling, memory paging, tool mediation, quota enforcement, and auditability.

Myelin-AX is the Kubernetes-native reference implementation of USAGE. It uses CRDs, an operator, mutating admission, and sidecar-based governance to enforce zero-trust execution semantics for agent processes.

## The Case for an Agent OS
<details>
<summary>Expand The Case for an Agent OS</summary>

USAGE addresses not only technical orchestration but also enterprise governance and liability requirements for autonomous digital workers.

See: [case-for-agent-os.md](spec/case-for-agent-os.md)

</details>

## Problem Statement
<details>
<summary>Expand Problem Statement</summary>

Current agent deployments exhibit five systemic failures:
- Security paradox: high-privilege agents with weak containment and broad network reach.
- Token-resource mismatch: schedulers reason about CPU/RAM while real bottlenecks are tokens, context windows, and provider quotas.
- Governance vacuum: policy, redaction, and budget logic duplicated in application code.
- Agent memory wall: prompt growth, context degradation, and no explicit paging semantics.
- Coordination chaos: recursive loops, orphaned subtasks, and undefined supervision semantics.

USAGE addresses these failures by defining an operating substrate contract instead of another application framework.

</details>

## Definitive Scope Trigger
<details>
<summary>Expand Definitive Scope Trigger</summary>

USAGE uses an operational binary for classification. A workload is classified as an AI Agent under USAGE when both are true:
- Inference Core Invocation: it invokes one or more foundation model or LLM calls to determine state or control flow.
- Peripheral Access Capabilities: it can invoke external tools, databases, web APIs, or native host system calls.

This rule applies to scripts, binaries, background services, and active workload threads regardless of complexity.

</details>

## Absolute Boundary Condition
<details>
<summary>Expand Absolute Boundary Condition</summary>

No exemptions are granted based on implementation size or framework choice. Once the scope trigger is satisfied, the workload is inside the USAGE runtime boundary and MUST:
- Relinquish direct external side-effect pathways outside substrate mediation.
- Authenticate through a distinct cryptographically verifiable workload identity.
- Route all external actions through substrate tool-proxy enforcement.
- Submit to real-time token accounting and quota enforcement.

Substrate non-compliance handling is terminal (`SIG_AGENT_TERMINATE`).

</details>

## Design Principles
<details>
<summary>Expand Design Principles</summary>

- Zero Trust by Default
- Governance Outside the Trust Boundary
- Tool Calls as System Calls
- Tokens as Schedulable Resources
- Context as Virtual Memory
- Cognitive Workloads as Processes
- Deterministic Lifecycle Management
- Portable Agent Substrates

</details>

## Protocol Stack
<details>
<summary>Expand Protocol Stack</summary>

USAGE specifies a four-layer stack:
- Layer 4: Cognitive Application Layer
- Layer 3: Governance and Aspect Layer
- Layer 2: Runtime and Execution Layer
- Layer 1: Substrate Abstraction Layer (SAL)

Detailed specification: [usage-core.md](spec/usage-core.md)

</details>

## Runtime Architecture
<details>
<summary>Expand Runtime Architecture</summary>

Myelin-AX enforces USAGE through out-of-process supervision:
- `agent-brain`: untrusted cognition container.
- `myelin-proxy`: privileged governance sidecar implementing ASI server.
- `myelin-sandbox`: ephemeral isolated compute worker for untrusted tool execution.

Kubernetes topology and flow diagrams: [kubernetes-architecture.md](spec/kubernetes-architecture.md)

</details>

## System Calls (ASI)
<details>
<summary>Expand System Calls (ASI)</summary>

USAGE defines the Agent Substrate Interface over gRPC:
- `UsageSpawn`
- `UsageYield`
- `UsageSignal`
- `UsageMemPageOut`
- `UsageCallTool`

Formal contracts: [asi.proto](proto/usage/v1/asi.proto)

System-call semantics: [asi-system-calls.md](spec/asi-system-calls.md)

</details>

## Security Model
<details>
<summary>Expand Security Model</summary>

Source: [security-model.md](spec/security-model.md)

## Objectives
- Constrain blast radius of compromised cognition workloads.
- Prevent direct credential and network exfiltration.
- Enforce policy on every side effect.

## Mandatory Controls
- Sidecar governance enforcement out-of-process.
- Least-privilege tool capability allowlists.
- Runtime sandboxing for dynamic code execution.
- Immutable audit trail for syscall decisions.
- Identity-bound quotas and policy snapshots.

## Trust Domains
- Domain A: Untrusted cognition (`agent-brain`).
- Domain B: Trusted governance (`myelin-proxy`).
- Domain C: Isolated executor (`myelin-sandbox`).
- Domain D: Control plane (operator, admission, policy backend).

## Security Invariants
- A cannot directly invoke D or external networks.
- A -> side effects MUST traverse B.
- C instances are ephemeral and non-reusable.

</details>

## Memory Model
<details>
<summary>Expand Memory Model</summary>

Source: [memory-model.md](spec/memory-model.md)

## Tiers
- L1 Hot Context: model-visible active window.
- L2 Warm Semantic Cache: low-latency retrieved context.
- L3 Cold Persistent Store: long-term durable memory.

## Page Semantics
- Page Unit: opaque context segment with policy labels.
- Page Metadata: `{page_id, hash, sensitivity, ttl, lineage}`.
- `UsageMemPageOut` demotes pages L1->L2/L3.

## Invariants
- Sensitive pages MUST carry policy labels across tiers.
- Page references MUST be integrity-verifiable.
- Expired pages MUST be unavailable to retrieval unless retention exception applies.

## Memory Wall Handling
- Trigger demotion by token pressure threshold.
- Maintain retrieval quality via semantic compaction.
- Avoid unbounded prompt growth by bounded L1 resident set.

</details>

## Scheduling Model
<details>
<summary>Expand Scheduling Model</summary>

Source: [scheduling-model.md](spec/scheduling-model.md)

## Problem
CPU/RAM scheduling is insufficient for cognitive workloads that are quota-bound by tokens, context windows, and provider rate limits.

## Scheduling Dimensions
- Compute: CPU, memory, accelerator
- Cognitive: token budget, token rate, context pressure
- External: provider QPS/TPM, tool concurrency

## Policies
- Token Budget Class: `small`, `medium`, `large` with hard upper bounds.
- Context Pressure Index (CPI): ratio of L1 occupancy to configured max.
- Provider Backpressure State: normal, degraded, blocked.

## Decisions
- Admit when quotas and policy permit.
- Preempt/terminate when token budget exhausted.
- Force yield when CPI exceeds threshold.
- Defer tool calls under provider backpressure.

</details>

## Governance Model
<details>
<summary>Expand Governance Model</summary>

Source: [governance-model.md](spec/governance-model.md)

## Pipeline
1. Parse syscall request.
2. Resolve identity and session policy snapshot.
3. Run OPA policy evaluation.
4. Apply redaction/scrubbing transforms.
5. Enforce budget and concurrency guards.
6. Execute or deny.
7. Emit audit and telemetry records.

## Idempotency
- Tool calls SHOULD carry idempotency keys.
- Retries MUST preserve policy context and audit correlation ids.

## Audit
Each decision MUST record:
- session id
- syscall
- policy bundle version
- allow/deny outcome
- rationale code
- latency and resource metrics

</details>

## Multi-Agent Coordination Model
<details>
<summary>Expand Multi-Agent Coordination Model</summary>

Source: [coordination-model.md](spec/coordination-model.md)

## Process Tree
USAGE models agent orchestration as a supervision tree.
- Parent sessions own child sessions.
- Ownership includes budget partitioning and termination semantics.

## Spawn Semantics
- Parent MAY allocate sub-budget to child at `UsageSpawn`.
- Child MUST inherit policy floor from parent; narrowing is allowed, widening is denied.

## Failure Semantics
- Child failure can be `isolated` or `escalating` based on parent policy.
- Cascading termination behavior is explicit (`NONE`, `CHILDREN`, `SUBTREE`).

## Deadlock and Loop Control
- Maximum recursion depth MUST be bounded.
- Repeated identical tool-call signatures SHOULD trigger circuit-breaker policy.

</details>

## Kubernetes Integration
<details>
<summary>Expand Kubernetes Integration</summary>

Reference CRDs and lifecycle mappings:
- [sovereignagent.example.yaml](crds/sovereignagent.example.yaml)
- [agentsession.example.yaml](crds/agentsession.example.yaml)
- [rfc-001-lifecycle.md](spec/rfc-001-lifecycle.md)

</details>

## Compliance Suite
<details>
<summary>Expand Compliance Suite</summary>

Source: [compliance-suite.md](spec/compliance-suite.md)

## Profiles
- Core Profile: ASI syscall semantics and lifecycle state machine.
- Governance Profile: policy enforcement, redaction, idempotency.
- Isolation Profile: network isolation and sandbox integrity.
- Observability Profile: required events, attributes, and traces.

## Test Categories
- Protocol tests: request/response compatibility and error codes.
- Behavioral tests: legal/illegal state transitions.
- Security tests: deny-path guarantees and boundary bypass attempts.
- Performance tests: control-plane overhead and signal latency.

## Pass Criteria
Implementation is compliant when all mandatory tests pass for declared profile.

</details>

## OpenTelemetry Semantic Conventions Proposal
<details>
<summary>Expand OpenTelemetry Semantic Conventions Proposal</summary>

Source: [otel-semconv-proposal.md](spec/otel-semconv-proposal.md)

## Scope
Defines telemetry attributes and events for USAGE substrates.

## Resource Attributes
- `usage.substrate.name`
- `usage.substrate.version`
- `usage.session.id`
- `usage.agent.blueprint`

## Span Attributes
- `usage.syscall.name`
- `usage.syscall.result`
- `usage.policy.decision`
- `usage.policy.bundle.version`
- `usage.token.consumed`
- `usage.token.remaining`
- `usage.memory.tier.target`
- `usage.tool.name`

## Events
- `usage.state.transition`
- `usage.signal.delivered`
- `usage.pageout.completed`
- `usage.tool.denied`
- `usage.quota.exhausted`

## Metric Suggestions
- `usage_syscall_latency_ms`
- `usage_policy_denials_total`
- `usage_tokens_consumed_total`
- `usage_active_sessions`
- `usage_pageout_operations_total`

</details>

## CNCF Positioning and Roadmap
<details>
<summary>Expand CNCF Positioning and Roadmap</summary>

- CNCF standardization path: [cncf-positioning.md](spec/cncf-positioning.md)
- Standards-track roadmap: [standardization-roadmap.md](spec/standardization-roadmap.md)

</details>

## Threat Model
<details>
<summary>Expand Threat Model</summary>

Source: [threat-model.md](spec/threat-model.md)

## Threat Classes
- Prompt injection
- Tool hijacking
- Credential exfiltration
- Lateral movement
- Sandbox escape
- Poisoned retrieval memory
- Recursive execution loops
- Denial-of-wallet
- Covert prompt exfiltration

## STRIDE Mapping (Summary)
- Spoofing: forged session or tool identity
- Tampering: checkpoint/page mutation
- Repudiation: missing immutable audit records
- Information Disclosure: secret leakage via outputs/tools
- Denial of Service: token or tool saturation
- Elevation of Privilege: bypassing tool/policy boundary

## Mitigation Matrix
- Prompt injection -> policy-typed tool arguments + deny-by-default tool scopes
- Tool hijacking -> signed tool registry + strict name/version pinning
- Credential exfiltration -> no static creds in cognition domain, proxy-issued ephemeral credentials
- Lateral movement -> egress-deny network policy and namespace segmentation
- Sandbox escape -> hardened runtimeclass, seccomp, read-only FS
- Denial-of-wallet -> token budgets, recursion limits, per-session circuit breakers
- Recursive loops -> supervision depth limits and mandatory yield checkpoints

</details>

## ASI Compliance Tests
<details>
<summary>Expand ASI Compliance Tests</summary>

Source: [asi-compliance-tests.md](compliance-tests/asi-compliance-tests.md)

## Core Lifecycle
- Spawn returns `PENDING`.
- Illegal transitions are rejected.
- Terminated sessions reject further syscalls.

## Syscall Behavior
- `UsageSignal` idempotency by `(session_id, sequence)`.
- `UsageCallTool` deny path includes policy decision metadata.
- `UsageMemPageOut` returns integrity-reference per page.

## Governance
- Capability violation yields `PERMISSION_DENIED`.
- Token exhaustion yields terminal state and `RESOURCE_EXHAUSTED` semantics.

## Isolation
- Attempted direct egress from cognition container fails.
- Dynamic code execution occurs only in sandbox profile.

## Observability
- Required state-transition events emitted.
- Required attributes present on syscall spans.

</details>

## Architecture Diagrams
<details>
<summary>Expand Architecture Diagrams</summary>

### 1) USAGE Protocol Stack
Source: [usage-protocol-stack.mmd](diagrams/usage-protocol-stack.mmd)

```mermaid
graph TD
  %% USAGE Protocol Stack
  subgraph L4["Layer 4 - Cognitive Application Layer"]
    L4P["Prompts"]
    L4M["LLMs"]
    L4F["CrewAI / LangChain Frameworks"]
    L4I["Intent Engine"]
  end

  subgraph L3["Layer 3 - Governance and Aspect Layer"]
    L3O["OPA Guardrails"]
    L3T["Token Quota Budgeting"]
    L3R["PII and Secret Redaction Proxy"]
  end

  subgraph L2["Layer 2 - Runtime and Execution Layer"]
    L2S["State Machine Tracking"]
    L2G["Asynchronous Signaling (SIG_PAUSE)"]
    L2I["IPC Engine"]
  end

  subgraph L1["Layer 1 - Substrate Abstraction Layer (SAL)"]
    L1V["Virtual Memory Paging"]
    L1D["Sandboxed Drivers"]
    L1K["Kubernetes / Bare-Metal Infrastructure"]
  end

  L4P --> L4I
  L4M --> L4I
  L4F --> L4I

  L4I --> L3O
  L4I --> L3T
  L4I --> L3R

  L3O --> L2S
  L3T --> L2G
  L3R --> L2I

  L2S --> L1V
  L2G --> L1D
  L2I --> L1K

  classDef layer4 fill:#EAF2FF,stroke:#1D4ED8,stroke-width:2px,color:#0B1F44;
  classDef layer3 fill:#E8FFF4,stroke:#047857,stroke-width:2px,color:#052E26;
  classDef layer2 fill:#FFF7E6,stroke:#B45309,stroke-width:2px,color:#3B1D00;
  classDef layer1 fill:#F3F4F6,stroke:#374151,stroke-width:2px,color:#111827;

  class L4P,L4M,L4F,L4I layer4;
  class L3O,L3T,L3R layer3;
  class L2S,L2G,L2I layer2;
  class L1V,L1D,L1K layer1;
```

### 2) Myelin-AX Kubernetes Pod Topography
Source: [myelin-ax-pod-topography.mmd](diagrams/myelin-ax-pod-topography.mmd)

```mermaid
graph LR
  subgraph POD["Kubernetes Pod (The Myelin Sheath)"]
    subgraph AB["Container: agent-brain (User Space)"]
      AB1["Runs Unprivileged"]
      AB2["No Egress Network Access"]
    end

    subgraph MP["Container: myelin-proxy (Kernel Space / Sidecar)"]
      MP1["USAGE gRPC Server"]
      MP2["Governance Enforcement"]
    end

    subgraph SB["Container: myelin-sandbox (Ephemeral Tool Worker)"]
      SB1["Zero-Trust Tool Execution"]
      SB2["RuntimeClass: gvisor / firecracker"]
    end

    AB1 <--> |"localhost:50051 (gRPC System Calls)"| MP1
    MP2 -->|"Control Link: Spawn / Reap Tool Task"| SB1
  end

  LLM["External LLM Providers"]
  OPA["Central OPA Gatekeeper Engine"]

  MP2 -->|"Policy-Mediated Egress"| LLM
  MP2 -->|"ValidateAction / Policy Decisions"| OPA
```

### 3) USAGE Process Lifecycle State Machine
Source: [usage-lifecycle-state-machine.mmd](diagrams/usage-lifecycle-state-machine.mmd)

```mermaid
stateDiagram-v2
  [*] --> Pending

  Pending --> Active: Quota Approved & Identity Issued
  Active --> Thinking: Inference / Tool Invocation Triggered
  Thinking --> Paused: UsageYield Call / HITL Interrupt
  Paused --> Active: Context PageIn / Token Resume

  Active --> Terminated: Intent Met / Normal Execution End
  Thinking --> Terminated: Quota Breach / End of Context Window
  Paused --> Terminated: System TTL Expired / Manual Eviction

  state Pending {
    [*] --> PendingReady
    PendingReady: Awaiting admission checks,\nquota reservation,\nand workload identity binding.
  }

  state Active {
    [*] --> ActiveReady
    ActiveReady: Session admitted and resumable.\nEligible for inference scheduling.
  }

  state Thinking {
    [*] --> ThinkingLoop
    ThinkingLoop: Model inference and tool orchestration loop.\nBudget and policy continuously enforced.
  }

  state Paused {
    [*] --> PausedCheckpoint
    PausedCheckpoint: Context serialized.\nAwaiting operator/human/runtime resume decision.
  }

  state Terminated {
    [*] --> Finalized
    Finalized: Session closed.\nResources reclaimed and audit finalized.
  }
```

### 4) UML System Call Sequence (Tool Execution Flow)
Source: [usage-tool-execution-sequence.mmd](diagrams/usage-tool-execution-sequence.mmd)

```mermaid
sequenceDiagram
  participant AB as Agent Brain (User Space)
  participant MP as Myelin Proxy (Sidecar Kernel)
  participant PE as Policy Engine (OPA / Gatekeeper)
  participant SD as Sandboxed Tool Driver

  AB->>MP: UsageCallTool(tool_name, json_args)
  Note right of MP: Pause brain thread and lock session context
  MP->>PE: ValidateAction(payload)
  PE-->>MP: AccessGranted: True

  Note right of MP: Apply regex scrubbing for PII and secrets
  MP->>SD: ExecuteTool(clean_payload)
  Note right of SD: Execute inside RuntimeClass isolation (gVisor)
  SD-->>MP: raw_stdout_string

  Note right of MP: Compute token/resource metrics\nand persist context metadata
  MP-->>AB: ToolResponse(safe_payload, metrics, status)
```

</details>

## Appendix
<details>
<summary>Expand Appendix</summary>

- Protobuf contracts: [asi.proto](proto/usage/v1/asi.proto)
- JSON schema: [agent_manifest.schema.json](schemas/agent_manifest.schema.json)
- Examples: [agent_manifest_basic.yaml](examples/agent_manifest_basic.yaml), [agent_manifest_advanced.yaml](examples/agent_manifest_advanced.yaml)

</details>

## Status
- Version: `v0.2-draft`
- Maturity: Draft for implementer review
- Intended process: public specification -> reference implementation hardening -> conformance publication
