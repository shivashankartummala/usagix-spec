# USAGIX Architecture Documentation

This directory contains architecture documentation, diagrams, and design rationale for the USAGIX specification.

## Directory Structure

### [diagrams/](diagrams/)
Architectural diagrams and visualizations:
- **[trust-domains.mmd](diagrams/trust-domains.mmd)** — Trust domain separation diagram
- **[protocol-stack.mmd](diagrams/protocol-stack.mmd)** — ASI protocol stack layers
- **[agent-lifecycle.mmd](diagrams/agent-lifecycle.mmd)** — Agent runtime state machine
- **[memory-model.mmd](diagrams/memory-model.mmd)** — Memory virtualization model (L1/L2/L3)
- **[governance-flow.mmd](diagrams/governance-flow.mmd)** — Governance decision flow
- **[checkpoint-flow.mmd](diagrams/checkpoint-flow.mmd)** — Checkpoint/restore process

### [documents/](documents/)
Detailed architecture documentation:
- **[trust-domains.md](documents/trust-domains.md)** — Trust domain architecture and security boundaries
- **[protocol-stack.md](documents/protocol-stack.md)** — ASI gRPC protocol stack design
- **[memory-model.md](documents/memory-model.md)** — Memory virtualization architecture
- **[capability-system.md](documents/capability-system.md)** — Capability-based access control design
- **[governance-engine.md](documents/governance-engine.md)** — Governance policy evaluation

## Architecture Overview

USAGIX defines a layered architecture separating concerns across multiple tiers:

```
┌─────────────────────────────────────────────────────────┐
│  Application Layer (Agent Code)                         │
└──────────────────────┬──────────────────────────────────┘
                       │ gRPC (ASI v1)
┌──────────────────────▼──────────────────────────────────┐
│  Cognitive Container (Untrusted Agent Runtime)          │
│  - Language Runtime (Python, JavaScript, etc.)          │
│  - Agent Checkpoint Manager                             │
│  - Memory virtualization client                         │
└──────────────────────┬──────────────────────────────────┘
                       │ gRPC (ASI v1)
                    [Trust Boundary]
                       │ gRPC (ASI v1)
┌──────────────────────▼──────────────────────────────────┐
│  Governance Enforcement Plane (Trusted)                 │
│  - Capability Ledger & Validator                        │
│  - Token Budget Enforcer                                │
│  - Policy Evaluator & Decision Engine                   │
│  - Immutable Audit Logger                               │
│  - Output Scrubber & Secrets Filter                     │
│  - Memory Manager & Checkpoint Store                    │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│  Substrate Layer (Kubernetes, Serverless, WASM, VM)     │
│  - Pod/Function/Process Management                      │
│  - Network/IPC Mediation                                │
│  - Resource Quotas & Isolation                          │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│  Infrastructure (Hardware, OS, Container Runtime)       │
└─────────────────────────────────────────────────────────┘
```

## Core Design Principles

### 1. Trust Domain Separation
- **Untrusted**: Cognitive container (agent runtime)
- **Trusted**: Governance enforcement plane
- **Mediation**: All agent side effects flow through enforcement plane
- **Guarantee**: Agent cannot bypass governance controls

### 2. Capability-Based Access Control
- **Deny-by-default**: Absence of capability = denial
- **Cryptographic tokens**: Capabilities are signed tokens
- **Ledger validation**: All RPC calls validated against capability ledger
- **Immediate revocation**: Capability revocation takes effect immediately

### 3. Memory Virtualization
- **L1 (Hot)**: Current context window (~8K tokens)
- **L2 (Warm)**: Recent history (~128K tokens)
- **L3 (Cold)**: Long-term storage (unbounded)
- **Checkpointing**: Agents periodically save state to persistent storage
- **Portability**: Checkpoints are substrate-agnostic; any compliant substrate can restore

### 4. Governance-First Design
- **Declarative policies**: Define what agents can do via policies
- **Audit trail**: Every decision logged with cryptographic signature
- **Compliance reporting**: Generate audit reports for regulatory compliance
- **Policy versioning**: Policies can be updated; changes are audited

### 5. Substrate Abstraction
- **gRPC interface**: Substrate-independent protocol
- **Modular implementation**: Substrates implement governance plane independently
- **Optimizable**: Substrates can optimize memory/I/O without changing spec
- **Testable**: Conformance test suite validates substrate compliance

## Key Architectural Decisions

### Why Trust Domain Separation?
The key security insight is that untrusted agent code cannot be trusted to respect governance policies if it has direct access to the system. By placing the governance enforcement plane in a trusted domain with exclusive control over:

- Network access (agents cannot reach external services directly)
- File I/O (agents cannot read arbitrary files)
- Tool invocation (agents cannot call tools not approved by governance)
- Memory management (agents cannot expand memory beyond quota)

...we can guarantee that policies are enforced even against a compromised agent.

### Why Capability-Based Access Control?
Capabilities (unforgeable tokens) enable fine-grained, revocable access control without requiring a central policy repository to be consulted on every access. Capabilities can be:

- **Delegated**: Passed to other agents or services
- **Attenuated**: Reduced scope (e.g., "read but not write")
- **Revoked**: Central revocation list checked on every use
- **Signed**: Cryptographically authenticated; substrate cannot forge

### Why Multi-Tier Memory?
Agents process information at different time scales:

- **L1 (immediate)**: Working memory for current task (tokens needed now)
- **L2 (recent)**: Context for recent decisions (last N interactions)
- **L3 (historical)**: Long-term knowledge (facts, learned patterns)

Different tiers have different performance/cost tradeoffs:

- L1 in fast memory (GPU/host memory); expensive but fast
- L2 in persistent storage with caching; moderate cost/speed
- L3 in durable storage; cheap but slower to retrieve

Agents benefit from all three tiers without managing the complexity.

### Why gRPC Over REST?
- **Streaming**: Needed for real-time I/O (stdout, token streaming)
- **Multiplexing**: Efficient for many concurrent requests
- **Strongly typed**: Protocol buffers ensure type safety
- **Language-agnostic**: Works with Python, Go, JavaScript, C++, etc.

## Reference Implementation Mappings

See [../reference/myelin-ax/ARCHITECTURE.md](../reference/myelin-ax/ARCHITECTURE.md) for how abstract USAGIX concepts map to Kubernetes primitives:

- **Cognitive Container** → Kubernetes pod with LLM runtime sidecar
- **Governance Enforcement Plane** → Kubernetes pod with myelin-proxy sidecar
- **Capability Tokens** → Kubernetes ServiceAccount with RBAC rules
- **Immutable Audit Logging** → CloudTrail or GKE audit logs
- **Memory Management** → Kubernetes PVC for persistent checkpoints

## Design Document Index

| Document | Focus | Audience |
|----------|-------|----------|
| [trust-domains.md](documents/trust-domains.md) | Security model and trust boundaries | Security engineers, implementers |
| [protocol-stack.md](documents/protocol-stack.md) | ASI gRPC protocol design | Implementers, protocol designers |
| [memory-model.md](documents/memory-model.md) | Memory virtualization strategy | Implementers, architects |
| [capability-system.md](documents/capability-system.md) | Capability-based access control | Security engineers, policy authors |
| [governance-engine.md](documents/governance-engine.md) | Policy evaluation and decisions | Policy authors, operators |

---

## For Different Roles

### Substrate Implementers
1. Read this README for overview
2. Study [documents/trust-domains.md](documents/trust-domains.md) for security requirements
3. Review [documents/protocol-stack.md](documents/protocol-stack.md) for protocol details
4. Reference [../reference/myelin-ax/ARCHITECTURE.md](../reference/myelin-ax/ARCHITECTURE.md) for implementation patterns

### Security Engineers
1. Start with [documents/trust-domains.md](documents/trust-domains.md)
2. Review [../spec/security-model.md](../spec/security-model.md) for threat analysis
3. Check [documents/capability-system.md](documents/capability-system.md) for access control design
4. See [../SECURITY.md](../SECURITY.md) for security requirements

### Architects & Designers
1. This README for high-level overview
2. Review diagrams in [diagrams/](diagrams/) for visual understanding
3. Deep dive into individual architecture documents as needed
4. Compare with [../reference/myelin-ax/ARCHITECTURE.md](../reference/myelin-ax/ARCHITECTURE.md)

---

**Last Updated**: 2025-03-15  
**Maintained By**: USAGIX Specification Stewards
