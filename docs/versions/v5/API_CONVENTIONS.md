# API Conventions v5

## Purpose

`v5` continues v4's transport rules — every cross-app message is carried on a
persistent WebSocket using the same JSON envelope. The conventions below are
materially identical to v4. Differences from v4 are limited to endpoint paths
(now under `/v5/...`) and a single new role-priority value documented under
"Request Context".

## Transport

Each app exposes exactly one ecosystem WebSocket endpoint for its role:

- Chat Orchestrator: `/v5/chat`
- Model Host: `/v5/model`
- RAG Engine: `/v5/retrieval`
- Embedding Service: `/v5/embeddings`
- Context Fragmenter: `/v5/fragmenter`

A caller opens one WebSocket to each upstream service it depends on and keeps
it open until the caller or callee shuts down, the connection becomes
unhealthy, or the deployment rotates credentials. The socket supports multiple
concurrent application messages. Responses and progress events are routed by
`correlationId`.

A caller never opens more than one socket of the same role to the same upstream
process. If a deployment runs two distinct Model Host processes (e.g. a
Vanamonde LLM Host and a Fragmenter LLM Host), the consuming module — by the
1:1 module-to-LLM rule — sees only one of them and opens exactly one socket.

## Envelope

Every client-to-server request, server-to-client event, error, cancellation,
and heartbeat uses:

```json
{
  "type": "turn.submit",
  "correlationId": "0198d1ad-9b41-7c87-94a2-6dbdbb7f0602",
  "payload": {},
  "error": null
}
```

Fields:

- `type`: stable message type. Dot-separated domain names are preferred.
- `correlationId`: stable ID for the logical request, turn, job, or heartbeat.
- `payload`: message-specific object. Omit only when the type needs no data.
- `error`: structured error object. Present only when `type` is `error`.

Errors:

```json
{
  "type": "error",
  "correlationId": "0198d1ad-9b41-7c87-94a2-6dbdbb7f0602",
  "error": {
    "code": "validation_failed",
    "message": "payload.request is required.",
    "retryable": false,
    "details": {}
  }
}
```

## Cancellation

Cancel in-flight work by sending:

```json
{
  "type": "cancel",
  "correlationId": "0198d1ad-9b41-7c87-94a2-6dbdbb7f0602"
}
```

The `correlationId` is the work to cancel. Services should stop downstream work
where possible and emit either a final `error` with `code: "cancelled"` or a
domain-specific cancelled event.

Fire-and-forget messages (notifications without a server response, e.g. the
Context Fragmenter's `session.observed`) are not cancellable; the caller
simply stops sending them.

## Heartbeat

Either side may send:

```json
{
  "type": "heartbeat",
  "correlationId": "0198d1ad-9b41-7c87-94a2-6dbdbb7f0602",
  "payload": { "sentAt": "2026-05-09T12:00:00Z" }
}
```

The receiver answers with `heartbeat` using the same `correlationId` and a
`receivedAt` timestamp.

## Request Context

Domain requests keep their semantic context inside `payload.request`:

```json
{
  "requestContext": {
    "callerService": "chat-orchestrator",
    "priority": "interactive",
    "requestedAt": "2026-05-09T12:00:00Z",
    "timeoutMs": 30000,
    "debug": false
  }
}
```

`priority` values for domain services:

- `interactive` — user is actively waiting; minimum response time, do not
  defer.
- `background` — system-initiated work that does not block a user-facing
  request; may be queued behind interactive work.
- `maintenance` — scheduled or idle-time work (Context Fragmenter deep
  consolidation, persona long-term-memory extraction). May be deferred,
  paused, or pre-empted by interactive work. New in v5.

Model Host and Embedding Service may use numeric priorities internally for
queue ordering.

## Authentication

Authentication is negotiated during the WebSocket upgrade. Browser clients may
use `?token=...`; non-browser service clients should use an authorization
header or deployment-level private network authentication.

The first application frame must not carry credentials.

## Health, Status, And Admin

Health, model status, preload, unload, and other lifecycle operations are
ordinary WebSocket envelope messages such as `health.get`, `model.status`,
`model.preload`, and `model.unload`.

If a service exposes local debug or process-manager endpoints, those are
local-ops-only. They are not part of the Swirlock ecosystem API and are not
described here.
