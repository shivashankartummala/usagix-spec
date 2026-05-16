# Myelin-AX Reference Implementation Architecture

## Overview

Myelin-AX is a reference implementation of the USAGIX specification for Kubernetes environments. 
This document describes the Myelin-AX architecture and how it maps USAGIX abstract concepts 
to Kubernetes primitives.

## Abstract ↔ Concrete Mapping

| USAGIX Abstract Concept | Myelin-AX Implementation |
|---|---|
| Cognitive Container | Agent process in Kubernetes Pod (container named `agent-brain`) |
| Governance Enforcement Plane | Sidecar container in same Pod (container named `myelin-proxy`) |
| Local Mediation Endpoint | gRPC service on `localhost:50051` |
| Tool Executor | Separate ServiceAccount with RBAC policies |
| Capability Ledger | Kubernetes Secret mounted into myelin-proxy |
| Audit Logging | Integration with cloud logging (Stackdriver, Loki, etc.) |

## Pod Topography

Myelin-AX deploys agents as Kubernetes Pods with two containers:

```
Pod: agent-instance-12345
├── Container: agent-brain
│   ├── Runtime: Python 3.11
│   ├── Code: Agent logic + LLM inference client
│   ├── Network: localhost only (no egress except 127.0.0.1:50051)
│   ├── Filesystem: read-only code mount + ephemeral tmpfs
│   ├── Seccomp: whitelist-only (no shell, socket, fork, etc.)
│   └── Calls: localhost:50051 (myelin-proxy gRPC)
│
└── Container: myelin-proxy
    ├── Runtime: Go 1.22+
    ├── Code: USAGIX ASI gRPC server + policy engine + tool executor
    ├── Network: Full (reaches databases, APIs, tools)
    ├── Filesystem: Full (policy files, audit logs, secrets)
    ├── Seccomp: permissive
    └── Exposes: localhost:50051 to agent-brain
```

## gRPC Interface

Agent-brain → myelin-proxy communication:

```protobuf
// Stub in agent-brain
service UsageASI {
  rpc UsageSpawn(SpawnRequest) returns (SpawnResponse);
  rpc UsageCallTool(ToolRequest) returns (ToolResponse);
  // ... etc
}

// Server in myelin-proxy
service UsageASI {
  // Implements all USAGIX RPC methods
}
```

Connection: `localhost:50051` (TLS on loopback for container-to-container auth).

## Kubernetes Resources

### AgentSession CRD (usage.io/v1)

```yaml
apiVersion: usage.io/v1
kind: AgentSession
metadata:
  name: agent-12345
  namespace: agents
spec:
  # Agent manifest (from USAGIX spec)
  agentConfig:
    image: agent-image:latest
    llmModel: gpt-4-turbo
    tokenBudget: 500000
  
  # Myelin-AX specific
  myelin:
    proxyImage: myelin-proxy:latest
    isolation: "pod"  # Pod-level isolation
    policySecret: strict-governance-hipaa
    auditSink: stackdriver
```

### RBAC and SecurityPolicy

Example RBAC for tool executor:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: usage-tool-executor
rules:
- apiGroups: [""] # Core API
  resources: ["secrets"]
  verbs: ["get"]  # Tool invocation secrets (DB passwords, API keys)
- apiGroups: [""] # Core API
  resources: ["pods"]
  verbs: ["get", "list"]  # For logging context
```

Example SecurityPolicy (pod isolation):

```yaml
apiVersion: policy/v1
kind: PodSecurityPolicy
metadata:
  name: agent-brain-restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  capabilities:
    add: []
  runAsUser:
    rule: "MustRunAsNonRoot"
  seLinux:
    rule: "MustRunAs"
    seLinuxOptions:
      type: usage_agent_container_t
  fsGroup:
    rule: "MustRunAs"
  seccomp:
    type: "RuntimeDefault"
```

## Checkpoint and State Management

Myelin-AX stores checkpoints in S3/GCS:

```
s3://usage-checkpoints/{namespace}/{agentSessionID}/checkpoint-{timestamp}.pb
```

Checkpoint format: Protocol Buffer (opaque to agent, transparent header as per USAGIX spec).

On agent resume (`UsageMemPageIn`):
1. Fetch checkpoint from S3
2. Validate header and signature
3. Rehydrate into agent-brain L1 context
4. Resume agent execution

## Audit and Logging

All decisions logged to Stackdriver/Loki:

```json
{
  "timestamp": "2025-03-15T14:23:47.123456Z",
  "sessionID": "ag-12345",
  "decision": "ALLOWED",
  "tool": "database.query",
  "reason": "Capability cap-abc123 permits SELECT on users table",
  "signature": "hmac-sha256-...",
  "previousHash": "sha256-..."
}
```

Audit logs are immutable (append-only to Stackdriver).

## Usage

To deploy an agent with Myelin-AX:

```bash
kubectl apply -f agent-session.yaml
kubectl logs -f pod/agent-instance-12345 -c agent-brain
```

For details on policy configuration, see `reference/myelin-ax/policies/`.

## Differences from Other Substrates

Myelin-AX assumes Kubernetes. Other substrates might:
- **Serverless (AWS Lambda)**: Deploy agent as function, governance in Lambda layer
- **WASM**: Deploy agent as WASM module, governance in host VM
- **VM**: Deploy agent as VM process, governance in privileged host process

All would implement the same USAGIX specification, but trust domain separation would look different.
