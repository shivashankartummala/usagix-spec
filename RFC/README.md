# USAGIX RFCs (Request for Comments)

This directory contains RFC (Request for Comments) proposals for the USAGIX specification.

## What is an RFC?

An RFC is a formal proposal for significant changes to the USAGIX specification. RFCs are used for:

- **MAJOR version changes** (breaking changes requiring v2.0.0)
- **Significant MINOR features** with architectural implications
- **Policy or governance changes**
- **New trust models or deployment patterns**

## RFC Process

### When to Use an RFC

Use an RFC for:
- ✅ Breaking changes (ASI v2, checkpoint format v2, policy schema v2)
- ✅ New trust models (multi-agent governance, federated policies, hardware attestation)
- ✅ Architectural changes (new memory tiers, new security guarantees)
- ✅ New RPC methods or protocol extensions
- ❌ Bug fixes (no RFC needed)
- ❌ Documentation improvements (no RFC needed)
- ❌ Minor feature additions without architectural impact

### RFC Timeline

1. **Open Issue** (Day 1): Submit GitHub issue with "RFC" label and link to RFC document
2. **Community Review** (Days 1-14): 2-week minimum review period; community feedback in GitHub comments
3. **Maintainer Feedback** (Day 7-14): Maintainers provide technical assessment
4. **RFC Refinement** (Days 1-14): RFC author updates proposal based on feedback
5. **TSC Vote** (Day 14+): Proposal goes to TSC for final approval
6. **Approval/Rejection** (Day 14+): TSC votes on accepting RFC as specification change
7. **Implementation** (Post-approval): If approved, RFC author or maintainer implements changes

### RFC Status

RFCs progress through the following states:

- **Draft**: Initial proposal, open for feedback
- **In Review**: Community review period (Days 1-14 minimum)
- **Ready for Vote**: All feedback addressed, ready for TSC vote
- **Approved**: TSC approved; proceed with implementation
- **Implemented**: Changes merged into specification
- **Rejected**: TSC rejected; proposal archived

## RFC Template

Each RFC should be named `rfc-NNN-short-title.md` where:
- `NNN` is the next sequential RFC number (see numbering scheme below)
- `short-title` is a kebab-case description (e.g., `rfc-002-memory-hints.md`)

Use this template:

```markdown
# RFC NNN: Title

## Status
Draft | In Review | Ready for Vote | Approved | Implemented | Rejected

## Summary
One paragraph summary of the proposal.

## Motivation
Why is this change necessary? What problem does it solve? What's the impact of NOT making this change?

## Specification
Detailed technical specification. Include:
- New RPCs (if any)
- Protocol changes (if any)
- Schema changes (if any)
- Wire format changes (if any)

## Backward Compatibility
- Can v1.0.0 code run on v1.1.0 with this change? (Or justify breaking change)
- Deprecation window: How long will old behavior be supported?
- Migration path: How do implementers upgrade?

## Security Considerations
- What are the security implications?
- Any new attack surfaces?
- How do security controls (trust domains, capability-based access, audit logging) apply?
- Threat model impact?

## Implementation Guide
- How would a substrate implement this?
- Which components need changes?
- Estimated implementation effort for reference implementation
- Testing requirements

## Examples
Concrete examples showing the proposal in action:
- API usage examples
- Configuration examples
- Before/after code snippets

## Alternatives Considered
What other approaches were considered? Why were they rejected?

## Open Questions
Unresolved questions for community feedback. Include:
- Design decisions still debated
- Edge cases to consider
- Feedback sought from specific stakeholders (e.g., Kubernetes implementers, serverless platforms)

## References
Links to related issues, RFCs, spec sections, or external standards.
```

## RFC Numbering

RFCs are numbered sequentially starting from 001:

- `RFC 001`: Agent Lifecycle (approved, implemented)
- `RFC 002`: Memory Hints (draft/example)
- `RFC 003`: Policy Schema v2 (draft/example)
- etc.

Do NOT reuse RFC numbers. If an RFC is rejected, the number is retired.

## Current RFCs

| # | Title | Status | Link |
|---|-------|--------|------|
| 001 | [Agent Lifecycle](rfc-001-lifecycle.md) | Implemented | v1.0.0 |

---

## How to Propose an RFC

1. **Create a draft** using the RFC template above
2. **Open a GitHub issue** with:
   - Title: `RFC: Short Title`
   - Label: `rfc`
   - Body: Brief summary + link to RFC document
3. **Submit PR** with the RFC document in `/RFC/` directory
4. **Community review** for minimum 2 weeks
5. **TSC vote** at TSC meeting or async vote
6. **Approval** → implementation begins
7. **Merge** RFC and specification changes together

## Tips for RFC Authors

- **Be specific**: Use concrete examples, not abstract prose
- **Address concerns**: Anticipate questions and address them in "Open Questions"
- **Explain tradeoffs**: Every design decision has tradeoffs; explain them
- **Security first**: Always consider security implications
- **Backward compatibility**: Explain impact on existing implementations
- **Get feedback early**: Share drafts in GitHub discussions before formal RFC

## Tips for RFC Reviewers

- **Be constructive**: Help improve proposals, don't just criticize
- **Ask questions**: "Why?" is a valid review comment
- **Consider tradeoffs**: Some proposals trade simplicity for flexibility; discuss which is better
- **Think long-term**: Will this decision make sense in 5 years?
- **Test implementations**: Does this work for Kubernetes? Serverless? WASM?

---

**Last Updated**: 2025-03-15  
**Maintained By**: USAGIX Specification Stewards
