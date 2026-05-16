# RFC-0004: Governance Model & Policy Evaluation

**Status**: Implemented (v1.0)  
**Authors**: USAGIX Specification Stewards  
**Created**: 2025-03-25  
**Updated**: 2025-05-16

## Summary

This RFC defines the governance enforcement model for USAGIX agents. It formalizes policy evaluation using Open Policy Agent (OPA), capability ledger management, approval workflows, and mandatory output redaction. All governance decisions are immutable and audited; no agent code can bypass the Governance Enforcement Plane.

## 1. Semantic Definitions

### 1.1 Governance Concepts

**Policy**: A declarative set of rules that determine whether an agent is permitted to execute an action. Policies are written in Rego (OPA language) or JSON schema.

**Policy Version**: A numbered release of a policy set (e.g., "governance-v1.2.3"). All decisions are tagged with the policy version used.

**Capability**: An unforgeable cryptographic token granting an agent permission to invoke a specific tool with bounded scope. Each capability has TTL and scope constraints.

**Capability Ledger**: A distributed ledger (backed by a database or blockchain) recording which agents have which capabilities. Substrate queries the ledger before allowing tool invocation.

**Tool Invocation**: An agent's request to call an external tool (web.search, database.query, file.write, etc.). All invocations are gated by capability checks and policies.

**Approval**: A manual or automatic grant of permission for an agent to perform a normally-denied action.

**Audit Record**: An immutable log entry recording a governance decision (allow/deny), who made it, when, and why.

**Output Redaction**: Automatic detection and removal of sensitive information (PII, credentials, secrets) from agent outputs before returning to caller.

### 1.2 Formal Type System

```
Policy = {
  version: string (e.g., "governance-v1.2.3"),
  rules: List<PolicyRule>,
  metadata: {
    created_at: Timestamp,
    updated_at: Timestamp,
    author: string,
    description: string
  }
}

PolicyRule = {
  id: string (e.g., "no-dangerous-tools"),
  effect: Allow | Deny,
  conditions: List<Condition>,
  rationale: string
}

Condition = {
  type: SessionAttribute | ToolName | ResourcePath | TimeRange | etc.,
  operator: Equals | NotEquals | In | NotIn | Matches | Gt | Lt | etc.,
  value: string | List<string> | Regex | timestamp
}

Capability = {
  id: UUID,
  issuer: UUID (Governance Enforcement Plane),
  subject: SessionID,
  resource: ToolName,
  scope: {
    parameters: Map<string, List<AllowedValue>>,
    rate_limit: RequestsPerSecond,
    ttl_seconds: uint64
  },
  created_at: Timestamp,
  expires_at: Timestamp,
  signature: bytes (HMAC-SHA256 with governance key)
}

CapabilityLedger = {
  entries: List<CapabilityRecord>,
  revision: uint64,
  last_updated: Timestamp,
  signature: bytes
}

CapabilityRecord = {
  session_id: SessionID,
  capability_id: UUID,
  status: Active | Revoked | Expired,
  issued_at: Timestamp,
  revoked_at: Timestamp | null,
  reason_revoked: string | null
}

AuditRecord = {
  id: UUID,
  timestamp: Timestamp,
  session_id: SessionID,
  parent_session_id: SessionID | null,
  decision: Allow | Deny | Error,
  policy_version: string,
  tool_name: string,
  tool_parameters: Map<string, string> (redacted),
  rule_id: string,
  rationale: string,
  latency_ms: float,
  signature: bytes (HMAC-SHA256)
}

ApprovalRequest = {
  id: UUID,
  session_id: SessionID,
  requested_action: string,
  reason: string,
  created_at: Timestamp,
  approver_id: UUID | null,
  approved_at: Timestamp | null,
  status: Pending | Approved | Denied,
  expiration_at: Timestamp | null
}

RedactionConfig = {
  enabled: boolean,
  patterns: List<Redaction>,
  default_redaction_string: string (default: "[REDACTED]")
}

Redaction = {
  type: PII | Secret | APIKey | CreditCard | SSN | etc.,
  pattern: Regex,
  replacement: string
}
```

## 2. Formal Contracts

### 2.1 Policy Evaluation Contract

**Definition**: `EvaluatePolicy(session_id, tool_name, tool_params, policy_version) → (decision: Allow|Deny, rule_id: string, rationale: string)`

**Preconditions**:
- Session exists and is active
- Tool name is registered in tool registry
- Policy version exists and is valid
- Tool parameters are parseable

**Postconditions (Decision Made)**:
1. OPA evaluates all rules in policy against session context
2. Decision returned: Allow (first matching Allow rule) or Deny (default)
3. Matching rule ID and rationale included
4. Audit record created atomically: `{timestamp, session_id, decision, policy_version, tool_name, rule_id}`
5. Audit signature computed over all fields except signature

**Atomicity Guarantee**: Policy evaluation and audit record creation are atomic. If system crashes mid-evaluation, agent receives no response (and must retry).

**Latency SLO**: Policy evaluation completes within T_policy_eval seconds (default: 500ms).

**Determinism Guarantee**: Given identical session context and identical policy, evaluation MUST produce identical decision.

**Proof Sketch**:
- OPA evaluation is deterministic (pure logic programming)
- Session context read once before evaluation
- Same inputs → same outputs

### 2.2 Capability Ledger Contract

**Definition**: `IssueCapability(session_id, tool_name, scope, ttl_seconds) → (capability_token, error)`

**Preconditions**:
- Session exists and is active
- Tool exists in registry
- Scope constraints are valid
- Session has permission to request this capability (via policy)

**Postconditions**:
1. Unique capability ID (UUID) generated
2. Capability record created with:
   - subject = session_id
   - resource = tool_name
   - scope = parameter constraints
   - expires_at = now + ttl_seconds
3. Capability signed with governance private key
4. Ledger entry created: status = Active
5. Audit record: `{type: CAPABILITY_ISSUED, session: session_id, capability: capability_id, ttl: ttl_seconds}`

**Revocation Contract**:

**Definition**: `RevokeCapability(capability_id, reason) → (ok, error)`

**Preconditions**:
- Capability exists in ledger
- Revocation authority confirmed

**Postconditions**:
1. Ledger entry updated: status = Revoked, revoked_at = now
2. All subsequent invocations of this capability fail with PERMISSION_DENIED
3. In-flight tool calls with this capability are allowed to complete (best effort)
4. Audit record: `{type: CAPABILITY_REVOKED, capability: capability_id, reason: reason}`

**Revocation Latency SLO**: Revocation propagates to all enforcement points within T_revocation_slo seconds (default: 1 second).

**Implementation**: Capability cache at enforcement point has TTL ≤ T_revocation_slo.

### 2.3 Approval Workflow Contract

**Definition**: `RequestApproval(session_id, action_description, reason) → (approval_request_id, error)`

**Preconditions**:
- Session exists
- Action is configured as requiresApproval
- No pending approval exists for this session+action

**Postconditions**:
1. ApprovalRequest created: status = Pending
2. Approval queue notified
3. Approval request ID returned
4. Agent typically yields (checkpoints) while awaiting approval
5. Audit record: `{type: APPROVAL_REQUESTED, session: session_id, action: action_description}`

**Approval Grant**:

**Definition**: `ApproveRequest(approval_request_id, approver_id) → (ok, error)`

**Preconditions**:
- Approval request exists and status = Pending
- Approver has permission to approve this action

**Postconditions**:
1. Status updated to Approved
2. approved_at timestamp set
3. Capability issued for the requested action (if applicable)
4. Session resumed (if checkpointed)
5. Audit record: `{type: APPROVAL_GRANTED, request: approval_request_id, approver: approver_id}`

**Approval Denial**:

**Definition**: `DenyRequest(approval_request_id, approver_id, reason) → (ok, error)`

**Postconditions**:
1. Status updated to Denied
2. Reason recorded
3. Session fails with PERMISSION_DENIED error
4. Audit record: `{type: APPROVAL_DENIED, request: approval_request_id, approver: approver_id, reason: reason}`

**Expiration**: Approval requests expire after T_approval_ttl seconds (default: 3600 / 1 hour).

### 2.4 Capability Verification Contract

**Definition**: `VerifyCapability(capability_token, tool_name, tool_params) → (valid: boolean, error: string)`

**Verification Steps**:
1. Parse JWT token (signature, expiration, subject)
2. Verify signature with governance public key
3. Check expiration: expires_at > now
4. Verify subject matches current session_id
5. Verify resource matches requested tool_name
6. Verify tool_params within scope constraints
7. Check ledger: status = Active (not revoked)

**Postconditions (if Valid)**:
- Verification succeeds
- Tool invocation proceeds
- Audit record: `{type: CAPABILITY_VERIFIED, capability: capability_id, result: VALID}`

**Postconditions (if Invalid)**:
- Verification fails
- Tool invocation denied with reason
- Audit record: `{type: CAPABILITY_VERIFIED, capability: capability_id, result: INVALID, reason: error_message}`

**Latency SLO**: Capability verification completes within 100ms.

### 2.5 Output Redaction Contract

**Definition**: `RedactOutput(raw_output: string, redaction_config: RedactionConfig) → (redacted_output: string, changes: List<Change>)`

**Preconditions**:
- raw_output is a string
- redaction_config specifies patterns to redact

**Postconditions**:
1. All matches to redaction patterns are replaced with redaction_string
2. List of redactions applied is recorded
3. Redacted output returned to agent caller
4. Audit entry: `{type: OUTPUT_REDACTED, session: session_id, pattern_count: N, matches_redacted: M}`

**Redaction Patterns**:
- PII: Name, email, phone, address patterns
- Credentials: API keys, tokens, passwords (regex: `[a-z0-9_-]{20,}`)
- Financial: Credit card numbers (regex: `\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b`)
- Healthcare: SSN, medical record numbers
- Custom: User-defined regex patterns

**Idempotency Guarantee**: Running redaction twice on same output produces same result (no re-replacement of already-redacted text).

**Latency SLO**: Redaction completes within 100ms for outputs < 1MB.

## 3. Governance Enforcement Architecture

### 3.1 Policy Evaluation Flow

```
Agent Tool Invocation Request
         ↓
[Governance Enforcement Plane]
         ↓
1. Lookup policy version (policy_version from request or default)
2. Build session context:
   - session_id, parent_id, user_id
   - token_budget remaining
   - tool_name, tool_parameters
   - parent_session's capabilities
         ↓
3. OPA Policy Evaluation
   - Load policy rules
   - Evaluate each rule condition
   - Aggregate decisions
         ↓
4. Rule Matching
   - Policy allows? → Proceed
   - Policy denies? → Check requiresApproval
   - Approval pending? → Block, return AWAITING_APPROVAL
         ↓
5. Capability Check
   - Verify capability token signature
   - Verify scope constraints
   - Check ledger (not revoked)
         ↓
6. Audit & Allow
   - Create audit record
   - Sign audit record
   - Return ALLOW to cognitive container
         ↓
Tool Invocation Proceeds
```

### 3.2 Policy Language (Rego)

USAGIX recommends OPA/Rego for policy authoring:

```rego
package usagix.governance

# Deny all by default
default allow = false

# Allow web search for all agents
allow {
  input.tool_name == "web.search"
  input.policy_version == "governance-v1.0"
}

# Allow database query only for analytics agents
allow {
  input.tool_name == "database.query"
  input.session.labels["domain"] == "analytics"
  input.policy_version == "governance-v1.0"
}

# Deny file deletion except with approval
deny {
  input.tool_name == "file.delete"
  not has_approval(input.session_id, "file.delete")
}

# Rate limit: max 10 requests/second per session
deny {
  tool_invocation_count_in_window > 10
  window_size_ms == 1000
}
```

## 4. Audit Trail Semantics

### 4.1 Audit Record Format

Every governance decision generates an audit record:

```
AuditRecord {
  id: UUID,
  timestamp: 2025-05-16T14:23:45.123Z,
  session_id: 550e8400-e29b-41d4-a716-446655440000,
  parent_session_id: null,
  decision: Allow,
  policy_version: governance-v1.2.3,
  tool_name: web.search,
  tool_parameters: {
    query: "[REDACTED: contains 8 chars]"
  },
  rule_id: "allow-web-search",
  rationale: "Tool is permitted for all sessions",
  latency_ms: 42.5,
  signature: 0x2f3a9c1d...
}
```

### 4.2 Audit Guarantees

**Append-Only**: Audit entries are written once and never modified.

**Immutability**: Entries are cryptographically signed; tampering is detectable.

**Completeness**: Every governance decision produces exactly one audit entry.

**Atomicity**: Audit entry is created atomically with the decision (at-most-once).

**Retention**: Audit logs retained for T_audit_retention (default: 365 days / 1 year).

**Accessibility**: Audit logs queryable by session_id, timestamp range, decision type, tool_name.

## 5. Mandatory Governance Controls

For a substrate to claim USAGIX v1.0 compliance:

1. **Policy Evaluation**: Deterministic policy evaluation (OPA or equivalent) required
2. **Capability Ledger**: Unforgeable capability tokens with signature verification
3. **Atomicity**: Policy decision and audit entry created atomically
4. **Determinism**: Identical input → identical decision
5. **Audit Completeness**: All decisions logged with session, timestamp, reason
6. **Revocation SLO**: Capability revocation propagates within 1 second
7. **Output Redaction**: Automatic detection of PII/credentials with configurable patterns
8. **Approval Workflows**: Support for requiresApproval actions with timeout

## 6. Example: Governance in Action

```
Timeline:
T0:   Agent requests file.delete /data/temp/cache.txt

T1:   [Governance Enforcement Plane]
      Policy evaluation:
      - Policy rule: "deny file.delete without approval"
      - Approval for file.delete exists? No
      → Decision: DENY (requires approval)
      → Audit: {type: TOOL_INVOCATION_DENIED, rule: no-delete-without-approval}

T2:   Agent receives AWAITING_APPROVAL error
      Agent submits approval request:
      RequestApproval(
        session_id=X,
        action="file.delete /data/temp/cache.txt",
        reason="Cleaning up old cache"
      )
      → approval_request_id = Y

T3:   Agent yields (checkpoints)
      → Waiting for human approval

T4:   [Human Approver]
      Reviews request: "Cleaning up old cache" ✓
      Approves: ApproveRequest(request_id=Y, approver=admin@org)
      → Capability issued for file.delete
      → Session resumed

T5:   Agent resumed
      Retries: file.delete /data/temp/cache.txt
      Passes capability to substrate
      
T6:   [Governance Enforcement Plane]
      VerifyCapability(capability_token)
      → Valid (signature OK, not revoked, in scope)
      → Decision: ALLOW
      → Audit: {type: TOOL_INVOCATION_ALLOWED, rule: approved-file-delete}

T7:   Tool invocation proceeds
      file.delete succeeds
      Output is scrubbed (no sensitive data)
      Audit: {type: TOOL_COMPLETED, tool: file.delete, status: success}
```

## 7. Backward Compatibility

USAGIX v1.0 defines governance model. Future versions may:
- Add new policy languages (Lua, Starlark, etc.) without breaking OPA
- Add new capability scopes
- Add new approval workflow types

Changes requiring v2.0:
- Breaking changes to audit record format
- Changes to capability token signature algorithm
- Changes to policy evaluation semantics

## 8. Normative References

- RFC-0001: USAGIX Core Architecture & Trust Domains
- RFC-0005: ASI System Calls (UsagixCallTool)
- Open Policy Agent (OPA) documentation
- RFC 2119 — Keywords for use in specs

## 9. Open Questions & Future Work

- **Q1**: Should policy evaluation support custom languages (Lua, WASM) or only OPA? Answer deferred to extensibility RFC.
- **Q2**: Can parent sessions delegate approval authority to child sessions? Answer deferred to multi-agent coordination RFC.
- **Q3**: Should capabilities support delegation (agent A grants capability to agent B)? Answer: TBD in delegation RFC.
- **Q4**: How are policy conflicts resolved (multiple rules with opposite effects)? Answer deferred to implementation guide (recommend: first match wins, then deny).

---

**Document Hash**: `sha256(this_document)`  
**Approvals**:
- [ ] TSC Security Lead
- [ ] TSC Governance Lead
- [ ] Community Review (2-week period)
