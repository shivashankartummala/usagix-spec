# USAGE Specification Index

Complete reference for the Universal Substrate for Agent Governance Enforcement (USAGE) specification suite.

## Quick Start

- **New to USAGE?** Start with [README.md](README.md)
- **Business pitch?** Read [spec/case-for-agent-os.md](spec/case-for-agent-os.md)
- **Implementing substrate?** Read [spec/usage-core.md](spec/usage-core.md) then [spec/asi-system-calls.md](spec/asi-system-calls.md)
- **Compliance requirement?** Check [spec/security-model.md](spec/security-model.md) and [spec/governance-model.md](spec/governance-model.md)

---

## Core Specifications

### Architecture & Design

| Document | Purpose | Audience |
|----------|---------|----------|
| [spec/usage-core.md](spec/usage-core.md) | Core architecture, layered protocol stack, design principles | Substrate builders, architects |
| [spec/DIAGRAMS.md](spec/DIAGRAMS.md) | Visual formalization: protocol stack, pod topography, state machine, tool execution, memory tiers | Technical leads, operations |
| [spec/case-for-agent-os.md](spec/case-for-agent-os.md) | Business case: why traditional OS fail, why USAGE is necessary, market imperatives | Business leaders, decision makers |
| [spec/runtime-state-machine.md](spec/runtime-state-machine.md) | Formal agent lifecycle: states, transitions, semantics, cold-start scheduling | Implementers, QA |

### API & Protocol Specifications

| Document | Purpose | Audience |
|----------|---------|----------|
| [proto/usage/v1/asi.proto](proto/usage/v1/asi.proto) | gRPC service definition, message types, RPC methods | Substrate builders, client library authors |
| [spec/asi-system-calls.md](spec/asi-system-calls.md) | Detailed RPC reference: UsageSpawn, UsageCallTool, UsageMemPageOut/In, UsageYield | Substrate builders, client developers |
| [schemas/agent_manifest.schema.json](schemas/agent_manifest.schema.json) | JSON Schema for agent deployment configuration | DevOps, platform teams |

### Governance & Security

| Document | Purpose | Audience |
|----------|---------|----------|
| [spec/governance-model.md](spec/governance-model.md) | Capability tokens, policy enforcement, quota system, OPA integration, audit | Security teams, compliance officers |
| [spec/security-model.md](spec/security-model.md) | Threat model, trust domains, isolation boundaries, attack scenarios, defense-in-depth | Security architects, penetration testers |

### Resource Management

| Document | Purpose | Audience |
|----------|---------|----------|
| [spec/memory-model.md](spec/memory-model.md) | Three-tier memory virtualization (L1/L2/L3), PageOut/PageIn, consistency invariants | Memory system designers |

### RFCs & Design Documents

| Document | Purpose | Audience |
|----------|---------|----------|
| [spec/rfc-001-lifecycle.md](spec/rfc-001-lifecycle.md) | Formal agent lifecycle RFC: state machine, checkpoint semantics, v2 compliance | Standards committee, reviewers |

---

## Schemas & Configuration

### JSON Schemas

| File | Purpose | Version |
|------|---------|---------|
| [schemas/agent_manifest.schema.json](schemas/agent_manifest.schema.json) | Agent deployment manifest (runtime, security, memory, monitoring) | 1.0 |
| [schemas/capability.schema.json](schemas/capability.schema.json) | Capability token specification (if exists) | 1.0 |
| [schemas/policy.schema.json](schemas/policy.schema.json) | Governance policy specification (if exists) | 1.0 |

### YAML Examples

| File | Purpose | Complexity |
|------|---------|-----------|
| [examples/agent_manifest_basic.yaml](examples/agent_manifest_basic.yaml) | Minimal agent configuration | Basic |
| [examples/agent_manifest_advanced.yaml](examples/agent_manifest_advanced.yaml) | Complex agent with all features | Advanced |
| [examples/capability_set_default.yaml](examples/capability_set_default.yaml) | Default capability set (read-only, safe operations) | Standard |
| [examples/policy_governance_strict.yaml](examples/policy_governance_strict.yaml) | Strict governance for regulated environments (HIPAA, PCI-DSS, SOC2) | Advanced |

---

## Protocol Buffers

### Service Definition

| File | Purpose | Status |
|------|---------|--------|
| [proto/usage/v1/asi.proto](proto/usage/v1/asi.proto) | Agent Substrate Interface (gRPC service) | Complete (v2) |

### Supporting Messages

| File | Purpose | Status |
|------|---------|--------|
| [proto/usage/v1/messages.proto](proto/usage/v1/messages.proto) | Message definitions (if split) | TBD |
| [proto/usage/v1/types.proto](proto/usage/v1/types.proto) | Type definitions and enums (if split) | TBD |

---

## Document Organization

### By Audience

**For Software Architects**:
1. [spec/case-for-agent-os.md](spec/case-for-agent-os.md) - Market opportunity
2. [spec/usage-core.md](spec/usage-core.md) - Architecture overview
3. [spec/DIAGRAMS.md](spec/DIAGRAMS.md) - Visual design

**For Platform Engineers**:
1. [spec/usage-core.md](spec/usage-core.md) - Core principles
2. [spec/asi-system-calls.md](spec/asi-system-calls.md) - RPC reference
3. [spec/runtime-state-machine.md](spec/runtime-state-machine.md) - Lifecycle management
4. [spec/memory-model.md](spec/memory-model.md) - Resource management
5. [examples/agent_manifest_*.yaml](examples/) - Configuration examples

**For Security & Compliance**:
1. [spec/security-model.md](spec/security-model.md) - Threat model, controls
2. [spec/governance-model.md](spec/governance-model.md) - Policy enforcement
3. [examples/policy_governance_strict.yaml](examples/policy_governance_strict.yaml) - Compliance example
4. [spec/rfc-001-lifecycle.md](spec/rfc-001-lifecycle.md) - Formal guarantees

**For Substrate Implementers**:
1. [spec/usage-core.md](spec/usage-core.md) - Design principles
2. [proto/usage/v1/asi.proto](proto/usage/v1/asi.proto) - Wire format
3. [spec/asi-system-calls.md](spec/asi-system-calls.md) - Implementation details
4. [spec/governance-model.md](spec/governance-model.md) - Policy engine
5. [spec/memory-model.md](spec/memory-model.md) - Memory management
6. [spec/security-model.md](spec/security-model.md) - Security hardening

### By Topic

**Agent Lifecycle & State Machine**:
- [spec/runtime-state-machine.md](spec/runtime-state-machine.md) - Detailed state machine
- [spec/rfc-001-lifecycle.md](spec/rfc-001-lifecycle.md) - RFC with formal semantics
- [spec/asi-system-calls.md](spec/asi-system-calls.md) - System calls (UsageSpawn, UsageSetState, UsageYield)

**Governance & Capabilities**:
- [spec/governance-model.md](spec/governance-model.md) - Complete governance model
- [spec/asi-system-calls.md](spec/asi-system-calls.md) - System calls (UsageSetCapability, UsageRevokeCapability, UsageCheckCapability)
- [examples/capability_set_default.yaml](examples/capability_set_default.yaml) - Example capability sets
- [examples/policy_governance_strict.yaml](examples/policy_governance_strict.yaml) - Example governance policies

**Resource Management & Quotas**:
- [spec/memory-model.md](spec/memory-model.md) - Memory virtualization (L1/L2/L3)
- [spec/asi-system-calls.md](spec/asi-system-calls.md) - System calls (UsageMemPageOut, UsageMemPageIn)
- [spec/governance-model.md](spec/governance-model.md) - Token budgets and quota enforcement
- [spec/runtime-state-machine.md](spec/runtime-state-machine.md) - Cold-start scheduling pattern

**Security & Isolation**:
- [spec/security-model.md](spec/security-model.md) - Threat model, trust domains, controls
- [spec/governance-model.md](spec/governance-model.md) - Access control and audit
- [spec/DIAGRAMS.md](spec/DIAGRAMS.md) - Trust boundary diagrams

**Tool Execution & Mediation**:
- [spec/asi-system-calls.md](spec/asi-system-calls.md) - UsageCallTool system call
- [spec/governance-model.md](spec/governance-model.md) - Capability validation and constraint enforcement
- [spec/DIAGRAMS.md](spec/DIAGRAMS.md) - Tool execution sequence diagram

---

## Key Concepts Quick Reference

### State Machine (5 States)

```
PENDING → ACTIVE → THINKING → PAUSED → TERMINATED
         ↓________________↑          ↓
                    (ACTIVE ← PAUSED)
```

**States**:
- **PENDING**: Spawned, not yet activated
- **ACTIVE**: Ready to run, awaiting inference
- **THINKING**: Executing LLM inference or tools
- **PAUSED**: Suspended with checkpoint, can resume
- **TERMINATED**: Finished, irreversible

See: [spec/runtime-state-machine.md](spec/runtime-state-machine.md)

### Memory Tiers (L1/L2/L3)

```
L1 (Hot): In-process context window (128K tokens, <1ms)
  ↕ PageOut/PageIn
L2 (Warm): Redis/Memcached (5-50ms, TTL: 24h)
  ↕ Move
L3 (Cold): S3/GCS storage (50-500ms, TTL: 30-90d, encrypted)
```

See: [spec/memory-model.md](spec/memory-model.md)

### Capability Model

```
Capability Token = (resource, operation, constraints, signature)

Constraints:
- Time window (when capability is valid)
- Rate limit (how often tool can be called)
- Data minimization (which fields are accessible)
- Data classification (min/max sensitivity level)
```

See: [spec/governance-model.md](spec/governance-model.md)

### RPC Methods

| Method | Purpose | Section |
|--------|---------|---------|
| **UsageSpawn** | Create agent session | [ASI Reference §1](spec/asi-system-calls.md#1-usagespawn) |
| **UsageSetState** | Transition agent state | [ASI Reference §2](spec/asi-system-calls.md#2-usagesetstate) |
| **UsageCallTool** | Invoke external tool (mediated) | [ASI Reference §3](spec/asi-system-calls.md#3-usagecalltool) |
| **UsageMemPageOut** | Evict context to L2/L3 | [ASI Reference §4](spec/asi-system-calls.md#4-usagemempage-out) |
| **UsageMemPageIn** | Retrieve context from L2/L3 | [ASI Reference §5](spec/asi-system-calls.md#5-usagemempage-in) |
| **UsageYield** | Yield with checkpoint | [ASI Reference §6](spec/asi-system-calls.md#6-usageyield) |
| **UsageSetCapability** | Grant capability | [ASI Reference §7](spec/asi-system-calls.md#7-usagesetcapability) |
| **UsageRevokeCapability** | Revoke capability | [ASI Reference §8](spec/asi-system-calls.md#8-usagerevokecapability) |

---

## Compliance & Standards

### Standards Alignment

| Standard | Coverage | Document |
|----------|----------|----------|
| **SOC2 Type II** | Audit logging, access control, change management | [spec/governance-model.md](spec/governance-model.md), [spec/security-model.md](spec/security-model.md) |
| **HIPAA** | Encryption, access logs, data minimization, audit trail | [examples/policy_governance_strict.yaml](examples/policy_governance_strict.yaml) |
| **PCI-DSS** | Cryptography, least privilege, audit logging | [examples/policy_governance_strict.yaml](examples/policy_governance_strict.yaml) |
| **GDPR** | Data minimization, audit trail, consent enforcement | [spec/governance-model.md](spec/governance-model.md) |

### Compliance Checklist

See [spec/rfc-001-lifecycle.md](spec/rfc-001-lifecycle.md) for v2 compliance checklist (18 items).

---

## Glossary of Terms

| Term | Definition | See |
|------|-----------|-----|
| **Agent** | Autonomous system making decisions via LLM inference | [spec/usage-core.md §1](spec/usage-core.md) |
| **Substrate** | Runtime environment implementing USAGE interface | [spec/usage-core.md §5](spec/usage-core.md) |
| **Capability Token** | Cryptographically signed authorization grant | [spec/governance-model.md §2](spec/governance-model.md) |
| **Checkpoint** | Serialized agent state (header + opaque payload) | [spec/memory-model.md §1.2](spec/memory-model.md) |
| **Token Budget** | Hard limit on LLM tokens consumed per agent | [spec/governance-model.md §4.1](spec/governance-model.md) |
| **Cold-Start Scheduling** | Spawning agent in PAUSED, activating at scheduled time | [spec/runtime-state-machine.md §4](spec/runtime-state-machine.md) |
| **Attention Window** | Indices of context agent is authorized to reference | [spec/memory-model.md §2.1](spec/memory-model.md) |
| **Tool Call Depth** | Maximum nesting of tool→tool invocations | [spec/governance-model.md §4.2](spec/governance-model.md) |
| **PageOut/PageIn** | Bidirectional memory tier transitions (eviction/rehydration) | [spec/memory-model.md §3](spec/memory-model.md) |
| **Output Scrubbing** | Redaction of PII/secrets before returning results | [spec/governance-model.md §3 Step 6](spec/governance-model.md) |
| **Governance Gate** | Mediation layer validating all external actions | [spec/security-model.md §3.1](spec/security-model.md) |

---

## Version History

| Version | Release Date | Notable Changes |
|---------|--------------|-----------------|
| **2.0** | 2025-03-15 | Cold-start scheduling (ACTIVE→PAUSED), bidirectional memory control (PageIn), checkpoint header standardization, v2 compliance checklist |
| **1.0** | 2025-01-15 | Initial release: core ASI, governance model, memory tiers, security model |

---

## Additional Resources

### Diagrams & Visual References

- [spec/DIAGRAMS.md](spec/DIAGRAMS.md) - Comprehensive architectural diagrams (5 diagrams total)
  1. USAGE Protocol Stack (Layer 1-4 abstraction)
  2. Myelin-AX Pod Topography (agent-brain + myelin-proxy)
  3. State Machine (5 states with transitions)
  4. Tool Execution Sequence (9-step flow with governance)
  5. Memory Virtualization (L1/L2/L3 tiers with control loops)

### Configuration Examples

- [examples/agent_manifest_basic.yaml](examples/agent_manifest_basic.yaml) - Minimal configuration
- [examples/agent_manifest_advanced.yaml](examples/agent_manifest_advanced.yaml) - Full-featured configuration
- [examples/capability_set_default.yaml](examples/capability_set_default.yaml) - Standard capabilities (read-only)
- [examples/policy_governance_strict.yaml](examples/policy_governance_strict.yaml) - Regulatory compliance policies

### Wire Format

- [proto/usage/v1/asi.proto](proto/usage/v1/asi.proto) - gRPC service and messages

---

## Contributing

To propose changes to USAGE specifications:

1. File an issue describing the change
2. If RFC-level, propose RFC document in [spec/](spec/)
3. Changes require consensus from:
   - Core maintainers
   - Substrate implementers
   - Security & compliance reviewers
4. Version bump required (major.minor.patch)

---

## License

All USAGE specifications and documentation are licensed under the [LICENSE](../LICENSE) in the root repository.

---

**Last Updated**: 2025-03-15
**Maintainer**: USAGE Platform - Architecture Team
