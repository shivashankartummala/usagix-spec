# USAGE: Universal Substrate for Agent Governance Enforcement

## Executive Summary
USAGE is an open interface specification for executing autonomous agent processes under strict substrate governance. It standardizes the boundary between cognitive workloads and execution substrates, analogous to the role of POSIX between applications and operating systems. USAGE defines control-plane and data-plane contracts for lifecycle, signaling, memory paging, tool mediation, quota enforcement, and auditability.

Myelin-AX is the Kubernetes-native reference implementation of USAGE. It uses CRDs, an operator, mutating admission, and sidecar-based governance to enforce zero-trust execution semantics for agent processes.

## The Case for an Agent OS
USAGE addresses not only technical orchestration but also enterprise governance and liability requirements for autonomous digital workers.

See: [case-for-agent-os.md](spec/case-for-agent-os.md)

## Problem Statement
Current agent deployments exhibit five systemic failures:
- Security paradox: high-privilege agents with weak containment and broad network reach.
- Token-resource mismatch: schedulers reason about CPU/RAM while real bottlenecks are tokens, context windows, and provider quotas.
- Governance vacuum: policy, redaction, and budget logic duplicated in application code.
- Agent memory wall: prompt growth, context degradation, and no explicit paging semantics.
- Coordination chaos: recursive loops, orphaned subtasks, and undefined supervision semantics.

USAGE addresses these failures by defining an operating substrate contract instead of another application framework.

## Definitive Scope Trigger
USAGE uses an operational binary for classification. A workload is classified as an AI Agent under USAGE when both are true:
- Inference Core Invocation: it invokes one or more foundation model or LLM calls to determine state or control flow.
- Peripheral Access Capabilities: it can invoke external tools, databases, web APIs, or native host system calls.

This rule applies to scripts, binaries, background services, and active workload threads regardless of complexity.

## Absolute Boundary Condition
No exemptions are granted based on implementation size or framework choice. Once the scope trigger is satisfied, the workload is inside the USAGE runtime boundary and MUST:
- Relinquish direct external side-effect pathways outside substrate mediation.
- Authenticate through a distinct cryptographically verifiable workload identity.
- Route all external actions through substrate tool-proxy enforcement.
- Submit to real-time token accounting and quota enforcement.

Substrate non-compliance handling is terminal (`SIG_AGENT_TERMINATE`).

## Design Principles
- Zero Trust by Default
- Governance Outside the Trust Boundary
- Tool Calls as System Calls
- Tokens as Schedulable Resources
- Context as Virtual Memory
- Cognitive Workloads as Processes
- Deterministic Lifecycle Management
- Portable Agent Substrates

## Protocol Stack
USAGE specifies a four-layer stack:
- Layer 4: Cognitive Application Layer
- Layer 3: Governance and Aspect Layer
- Layer 2: Runtime and Execution Layer
- Layer 1: Substrate Abstraction Layer (SAL)

Detailed specification: [usage-core.md](spec/usage-core.md)

## Runtime Architecture
Myelin-AX enforces USAGE through out-of-process supervision:
- `agent-brain`: untrusted cognition container.
- `myelin-proxy`: privileged governance sidecar implementing ASI server.
- `myelin-sandbox`: ephemeral isolated compute worker for untrusted tool execution.

Kubernetes topology and flow diagrams: [kubernetes-architecture.md](spec/kubernetes-architecture.md)

## System Calls (ASI)
USAGE defines the Agent Substrate Interface over gRPC:
- `UsageSpawn`
- `UsageYield`
- `UsageSignal`
- `UsageMemPageOut`
- `UsageCallTool`

Formal contracts: [asi.proto](proto/usage/v1/asi.proto)

System-call semantics: [asi-system-calls.md](spec/asi-system-calls.md)

## Security Model
Security boundaries and mandatory controls are defined in:
- [security-model.md](spec/security-model.md)
- [threat-model.md](spec/threat-model.md)

## Memory Model
USAGE defines hierarchical memory tiers (L1/L2/L3), eviction semantics, and page-out contracts:
- [memory-model.md](spec/memory-model.md)

## Scheduling Model
USAGE introduces inference-aware scheduling primitives (token budget classes, context pressure, provider quota backpressure):
- [scheduling-model.md](spec/scheduling-model.md)

## Governance Model
USAGE governance pipeline externalizes policy, redaction, retries, idempotency, and audit semantics:
- [governance-model.md](spec/governance-model.md)

## Multi-Agent Coordination Model
USAGE models supervised process trees, parent-child ownership, escalation, and deadlock handling:
- [coordination-model.md](spec/coordination-model.md)

## Kubernetes Integration
Reference CRDs and lifecycle mappings:
- [sovereignagent.example.yaml](crds/sovereignagent.example.yaml)
- [agentsession.example.yaml](crds/agentsession.example.yaml)
- [rfc-001-lifecycle.md](spec/rfc-001-lifecycle.md)

## Compliance Suite
USAGE conformance is profile-based:
- Core ASI Conformance
- Governance Conformance
- Isolation Conformance
- Observability Conformance

Suite design and test matrix:
- [compliance-suite.md](spec/compliance-suite.md)
- [asi-compliance-tests.md](compliance-tests/asi-compliance-tests.md)

## OpenTelemetry Semantic Conventions Proposal
USAGE proposes agent-runtime semantic attributes and events:
- [otel-semconv-proposal.md](spec/otel-semconv-proposal.md)

## CNCF Positioning and Roadmap
- CNCF standardization path: [cncf-positioning.md](spec/cncf-positioning.md)
- Standards-track roadmap: [standardization-roadmap.md](spec/standardization-roadmap.md)


## Architecture Diagrams

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

## Appendix
- Protobuf contracts: [asi.proto](proto/usage/v1/asi.proto)
- JSON schema: [agent_manifest.schema.json](schemas/agent_manifest.schema.json)
- Examples: [agent_manifest_basic.yaml](examples/agent_manifest_basic.yaml), [agent_manifest_advanced.yaml](examples/agent_manifest_advanced.yaml)

## Status
- Version: `v0.2-draft`
- Maturity: Draft for implementer review
- Intended process: public specification -> reference implementation hardening -> conformance publication
