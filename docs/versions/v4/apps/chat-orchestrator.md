# Chat Orchestrator v4

Endpoint: `WS /v4/chat`

## Client Messages

- `session.create`: payload is `{ "request": CreateSessionRequest }`.
- `session.get`: payload is `{ "sessionId": string }`.
- `session.delete`: payload is `{ "sessionId": string }`.
- `turn.submit`: payload is `{ "sessionId": string, "request": SubmitTurnRequest }`.
- `health.get`: payload may be omitted.
- `cancel`: cancels work with the same `correlationId`.
- `heartbeat`: liveness probe.

## Server Events

- `session.created`: payload is `{ "sessionId", "createdAt", "status" }`.
- `session.snapshot`: payload includes session metadata and persisted messages.
- `session.deleted`: payload is `{ "sessionId", "deleted": true }`.
- `health`: payload is `{ "status", "ready" }`.
- `turn.accepted`
- `turn.classifying`
- `turn.retrieval`
- `turn.queued`
- `turn.started`
- `turn.thinking`
- `turn.chunk`
- `turn.done`
- `error`
- `heartbeat`

`turn.done` contains the persisted assistant message, citations, finish reason,
and optional diagnostics.

