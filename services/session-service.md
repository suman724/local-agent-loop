# Session Service — Detailed Design

**Phase:** 1 (MVP)
**Repo:** `backend-session-service`
**Bounded Context:** SessionCoordination

---

## Purpose

The Session Service is the entry point for every agent session. It establishes sessions, performs compatibility checks, resolves workspaces, fetches policy from the Policy Service, and returns everything the Local Agent Host needs to begin work.

---

## Responsibilities

- Create and resume sessions
- Version and capability compatibility checks between client and backend
- Workspace resolution — create or retrieve the workspace for this session based on `workspaceHint`
- Fetch policy bundle from Policy Service and return it to the client
- Track session metadata and status transitions
- Session cancellation

---

## Relationships

| Calls | Purpose |
|-------|---------|
| Policy Service | Fetch policy bundle for the session |
| Workspace Service | Create or resolve workspace from `workspaceHint` |

| Called by | Purpose |
|-----------|---------|
| Local Agent Host | Create session, resume session, cancel session |
| Desktop App | Query session status |

---

## API Endpoints

### POST /sessions — Create Session

Called by the Local Agent Host at startup to establish a new session.

**Request:**
```json
{
  "tenantId": "tenant_abc",
  "userId": "user_123",
  "executionEnvironment": "desktop",
  "workspaceHint": {
    "localPaths": ["/Users/suman/projects/demo"]
  },
  "clientInfo": {
    "desktopAppVersion": "1.0.0",
    "localAgentHostVersion": "1.0.0",
    "osFamily": "macOS",
    "osVersion": "14.6"
  },
  "supportedCapabilities": [
    "File.Read",
    "File.Write",
    "Shell.Exec",
    "Git.Status",
    "Git.Diff",
    "Git.Commit",
    "Workspace.Upload",
    "LLM.Call"
  ],
  "supportedTools": [
    "ReadFile",
    "WriteFile",
    "RunCommand",
    "GitStatus",
    "GitDiff",
    "GitCommit"
  ]
}
```

**Response:**
```json
{
  "sessionId": "sess_789",
  "workspaceId": "ws_456",
  "compatibilityStatus": "compatible",
  "policyBundle": { ... },
  "llmGateway": {
    "endpoint": "https://llm-gateway.example.com",
    "authToken": "tok_abc"
  },
  "featureFlags": {
    "approvalUiEnabled": true,
    "mcpEnabled": false
  }
}
```

**Workspace resolution logic:**
- If `workspaceHint.localPaths` is provided → resolve or create a `local`-scoped workspace matching that path
- If no `workspaceHint` → create a new `general`-scoped workspace for this session only

---

### POST /sessions/{sessionId}/resume — Resume Session

Called by the Local Agent Host after a desktop restart when a checkpoint exists in the Local State Store.

**Request:**
```json
{
  "sessionId": "sess_789",
  "checkpointCursor": "step_004"
}
```

**Response:**
```json
{
  "sessionId": "sess_789",
  "workspaceId": "ws_456",
  "compatibilityStatus": "compatible",
  "policyBundle": { ... },
  "llmGateway": {
    "endpoint": "https://llm-gateway.example.com",
    "authToken": "tok_abc"
  }
}
```

---

### POST /sessions/{sessionId}/cancel — Cancel Session

**Request:**
```json
{
  "reason": "user_requested"
}
```

**Response:** `204 No Content`

---

### GET /sessions/{sessionId} — Get Session Metadata

**Response:**
```json
{
  "sessionId": "sess_789",
  "workspaceId": "ws_456",
  "tenantId": "tenant_abc",
  "userId": "user_123",
  "executionEnvironment": "desktop",
  "status": "SESSION_RUNNING",
  "createdAt": "2026-02-21T14:00:00Z",
  "expiresAt": "2026-02-21T18:30:00Z"
}
```

---

## Session Handshake Flow

```mermaid
sequenceDiagram
  participant LocalAgentHost as Local Agent Host
  participant SessionService as Session Service
  participant PolicyService as Policy Service
  participant WorkspaceService as Workspace Service

  LocalAgentHost->>SessionService: POST /sessions (clientInfo, capabilities, workspaceHint)
  SessionService->>WorkspaceService: Resolve or create workspace
  WorkspaceService-->>SessionService: workspaceId
  SessionService->>PolicyService: GET policy bundle (tenantId, userId, capabilities)
  PolicyService-->>SessionService: Policy bundle
  SessionService-->>LocalAgentHost: sessionId, workspaceId, policyBundle, llmGateway config
```

---

## Session Resume Flow

```mermaid
sequenceDiagram
  participant LocalAgentHost as Local Agent Host
  participant LocalStateStore as Local State Store
  participant SessionService as Session Service

  LocalAgentHost->>LocalStateStore: Load checkpoint
  LocalStateStore-->>LocalAgentHost: Session state and step cursor
  LocalAgentHost->>SessionService: POST /sessions/{sessionId}/resume
  SessionService-->>LocalAgentHost: Session metadata and refreshed policy bundle
  LocalAgentHost->>LocalAgentHost: Reconstruct message thread from checkpoint
```

---

## Session Metadata Model

| Field | Type | Description |
|-------|------|-------------|
| `sessionId` | string | Unique session identifier |
| `workspaceId` | string | Always present — resolved or created at session start |
| `tenantId` | string | Tenant |
| `userId` | string | User |
| `executionEnvironment` | enum | `desktop` or `cloud_sandbox` |
| `status` | enum | `SESSION_CREATED`, `SESSION_RUNNING`, `SESSION_COMPLETED`, `SESSION_FAILED`, `SESSION_CANCELLED` |
| `createdAt` | datetime | Session creation time |
| `expiresAt` | datetime | Policy bundle expiry — session must not continue past this |

---

## Compatibility Check

During handshake the Session Service validates:
- `localAgentHostVersion` is within the supported version range
- `desktopAppVersion` is within the supported version range
- Requested capabilities are a subset of what the tenant policy allows

If incompatible, returns `compatibilityStatus: "incompatible"` with a reason. The Local Agent Host must not proceed.

---

## Policy Bundle Validation (client-side)

After receiving the policy bundle, the Local Agent Host must verify:
- `expiresAt` is in the future
- `sessionId` in the bundle matches the returned `sessionId`
- `schemaVersion` is supported by this version of the Local Agent Host

If any check fails, the session must not start.
