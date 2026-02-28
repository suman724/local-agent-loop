# Audit Service — Detailed Design

> **Status: Future Implementation**
> The Audit Service is planned but not required for Phase 1 or 2. It should be designed and built in Phase 3 alongside stronger governance features. References in other service docs to `POST /audit/events` should be treated as no-ops or logged locally until this service exists.

**Phase:** 3
**Repo:** `backend-audit-service`
**Bounded Context:** ObservabilityAudit

---

## Purpose

The Audit Service provides an immutable, append-only record of security-relevant events across the system. It supports compliance reporting, incident investigation, and governance review.

---

## Responsibilities

- Ingest structured audit events from all services and the desktop client
- Store events in an immutable or append-only store
- Provide query interfaces for audit review and compliance reporting
- Retain audit records per tenant retention policy

---

## Relationships

| Called by | Purpose |
|-----------|---------|
| Local Agent Host | Session lifecycle events, tool invocations, approval events |
| Approval Service | Approval decision records |
| Session Service | Session creation and cancellation |
| Workspace Service | Artifact upload events |
| Policy Service | Policy bundle generation events |

---

## Minimum Audit Records

At minimum, the Audit Service must retain:

| Event | Why |
|-------|-----|
| Session created / cancelled | Session accountability |
| Policy bundle generated | What permissions were granted and when |
| Approval requested | What the agent asked to do |
| Approval resolved | What the user decided |
| High-risk tool invocations | Shell.Exec, File.Delete, Git.Push, Network.Http |
| LLM request and response metadata | Token usage, model used (not content) |
| Session state transitions | When sessions moved to failed or cancelled |

---

## Event Envelope

All events use a standard envelope:

```json
{
  "eventId": "evt_123",
  "eventType": "tool_completed",
  "timestamp": "2026-02-21T15:09:00Z",
  "tenantId": "tenant_abc",
  "userId": "user_123",
  "workspaceId": "ws_456",
  "sessionId": "sess_789",
  "taskId": "task_001",
  "component": "LocalAgentHost",
  "boundedContext": "AgentExecution",
  "severity": "info",
  "payload": {}
}
```

### Standard Event Names

| Event | Emitted by |
|-------|-----------|
| `session_created` | Session Service |
| `session_started` | Local Agent Host |
| `step_started` | Local Agent Host |
| `llm_request_started` | Local Agent Host |
| `llm_request_completed` | Local Agent Host |
| `tool_requested` | Local Agent Host |
| `tool_completed` | Local Agent Host |
| `approval_requested` | Local Agent Host |
| `approval_resolved` | Approval Service |
| `session_completed` | Local Agent Host |
| `session_failed` | Local Agent Host |

### Allowed component values

`DesktopApp`, `LocalAgentHost`, `LocalToolRuntime`, `LocalPolicyEnforcer`, `LocalApprovalUI`, `SessionService`, `PolicyService`, `ApprovalService`, `WorkspaceService`, `AuditService`, `TelemetryService`, `BackendToolService`

---

## API Endpoints

### POST /audit/events — Ingest Audit Event

```json
{
  "eventId": "evt_123",
  "eventType": "approval_resolved",
  "timestamp": "2026-02-21T15:08:11Z",
  "tenantId": "tenant_abc",
  "userId": "user_123",
  "workspaceId": "ws_456",
  "sessionId": "sess_789",
  "taskId": "task_001",
  "component": "ApprovalService",
  "boundedContext": "Approval",
  "severity": "info",
  "payload": {
    "approvalId": "appr_001",
    "decision": "approved",
    "riskLevel": "medium"
  }
}
```

**Response:** `202 Accepted`

---

## Implementation Notes

- Events are append-only — no updates or deletes
- `eventId` must be unique — callers should generate a UUID before sending
- Callers should treat audit event submission as fire-and-forget with retry. Agent execution must not be blocked waiting for audit confirmation.
- Retention period is configurable per tenant (default: 1 year)
