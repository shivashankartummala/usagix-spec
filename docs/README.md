# USAGIX Documentation

This directory contains comprehensive documentation for the USAGIX specification, organized by audience and use case.

## Directory Structure

### [getting-started/](getting-started/)
Quick-start guides for different audiences:
- **[overview.md](getting-started/overview.md)** — What is USAGIX? High-level overview
- **[substrate-implementers.md](getting-started/substrate-implementers.md)** — Start building a USAGIX substrate
- **[adopters.md](getting-started/adopters.md)** — Deploying USAGIX-compliant systems
- **[policy-authors.md](getting-started/policy-authors.md)** — Writing governance policies

### [implementation/](implementation/)
Detailed implementation guides by platform:
- **[kubernetes/](implementation/kubernetes/)** — Implementing USAGIX on Kubernetes (reference: Myelin-AX)
- **[serverless/](implementation/serverless/)** — Implementing USAGIX on AWS Lambda, Google Cloud Run, Azure Functions
- **[wasm/](implementation/wasm/)** — Implementing USAGIX in WebAssembly (Wasmtime, WasmEdge)
- **[vm-based/](implementation/vm-based/)** — Implementing USAGIX on VMs (KVM, QEMU, systemd)

### [compliance/](compliance/)
Compliance and certification documentation:
- **[conformance.md](compliance/conformance.md)** — USAGIX conformance test suite
- **[hipaa.md](compliance/hipaa.md)** — HIPAA compliance profile
- **[pci-dss.md](compliance/pci-dss.md)** — PCI-DSS compliance profile
- **[soc2.md](compliance/soc2.md)** — SOC2 compliance profile
- **[gdpr.md](compliance/gdpr.md)** — GDPR compliance profile

### [api-reference/](api-reference/)
API and protocol reference documentation:
- **[asi-v1.md](api-reference/asi-v1.md)** — Agent Substrate Interface (ASI) v1 specification
- **[rpc-methods.md](api-reference/rpc-methods.md)** — ASI RPC methods reference
- **[memory-api.md](api-reference/memory-api.md)** — Memory virtualization API reference
- **[policy-api.md](api-reference/policy-api.md)** — Policy and governance API reference
- **[protobuf/](api-reference/protobuf/)** — Protocol buffer definitions

### [governance/](governance/)
Governance and policy documentation:
- **[policies.md](governance/policies.md)** — Policy schema and examples
- **[audit-logging.md](governance/audit-logging.md)** — Audit logging and compliance reporting
- **[capability-tokens.md](governance/capability-tokens.md)** — Capability ledgers and token formats
- **[resource-quotas.md](governance/resource-quotas.md)** — Resource quota and rate limiting

## Quick Links by Role

### For Substrate Implementers
1. Start: [getting-started/substrate-implementers.md](getting-started/substrate-implementers.md)
2. Deep dive: Choose platform in [implementation/](implementation/)
3. Reference: [api-reference/](api-reference/) for ASI gRPC methods
4. Security: [../SECURITY.md](../SECURITY.md) for mandatory controls

### For Adopters & Operators
1. Start: [getting-started/adopters.md](getting-started/adopters.md)
2. Governance: [governance/policies.md](governance/policies.md) for writing policies
3. Compliance: [compliance/](compliance/) for your regulatory requirements
4. Monitoring: [governance/audit-logging.md](governance/audit-logging.md)

### For Policy Authors
1. Start: [getting-started/policy-authors.md](getting-started/policy-authors.md)
2. Policy schema: [governance/policies.md](governance/policies.md)
3. Capability tokens: [governance/capability-tokens.md](governance/capability-tokens.md)
4. Examples: [../examples/policies/](../examples/policies/) for real-world examples

### For Security Engineers
1. Threat model: [../spec/security-model.md](../spec/security-model.md)
2. Security guarantees: [../SECURITY.md](../SECURITY.md)
3. Audit logging: [governance/audit-logging.md](governance/audit-logging.md)
4. Compliance: [compliance/](compliance/) for your organization's requirements

### For Specification Contributors
1. Contributing guidelines: [../CONTRIBUTING.md](../CONTRIBUTING.md)
2. Architecture overview: [../architecture/](../architecture/)
3. Specification files: [../spec/](../spec/)
4. RFC process: [../RFC/README.md](../RFC/README.md)

## Documentation Standards

All documentation follows these conventions:

### Format
- **Markdown** (.md) for all documentation
- **RFC 2119 keywords** in USAGIX specification documents (MUST, SHOULD, etc.)
- **Plain language** in guides and tutorials

### Structure
- **Headings**: Use H2 (#) for top level, H3 (##) for subsections
- **Code blocks**: Include language identifier (```python, ```bash, ```protobuf)
- **Links**: Relative paths to other docs; full URLs for external resources

### Navigation
- **Table of contents**: Include for documents >500 words
- **Related documents**: Link to related docs at the end
- **Cross-references**: Use `[link text](path)` format

## Documentation Maintenance

Documentation is maintained by:
- **Specification authors** for API reference and compliance guides
- **Implementers** for platform-specific implementation guides
- **Community** via [CONTRIBUTING.md](../CONTRIBUTING.md) for improvements

### Keeping Docs Up-to-Date

- Docs are reviewed whenever specification changes (PRs to /spec/)
- Platform guides are updated when new versions are released
- Compliance guides are reviewed annually or when regulatory changes occur

---

## Contributing Docs

Found an error? Want to improve documentation? See [../CONTRIBUTING.md](../CONTRIBUTING.md) for:
- **Bug reports** for incorrect information
- **Documentation improvements** (clarify ambiguous sections, add examples)
- **New guides** for underrepresented use cases

When adding new documentation:
1. Create document in appropriate subdirectory
2. Add to this README's directory structure
3. Include in a PR with other changes (or standalone doc PR)
4. Follow documentation standards above

---

**Last Updated**: 2025-03-15  
**Maintained By**: USAGIX Specification Stewards
