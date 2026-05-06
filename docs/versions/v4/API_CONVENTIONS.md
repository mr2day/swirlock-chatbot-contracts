# API Conventions v4

## Purpose

`v4` removes ecosystem REST APIs. Every cross-app message is carried on a
persistent WebSocket using the same JSON envelope.

## Transport

Each app exposes exactly one ecosystem WebSocket endpoint for its role:

- Chat Orchestrator: `/v4/chat`
- Model Host: `/v4/model`
- RAG Engine: `/v4/retrieval`
- Embedding Service: `/v4/embeddings`
- Context Fragmenter: `/v4/context`

A caller opens one WebSocket to each upstream service it depends on and keeps it
open until the caller or callee shuts down, the connection becomes unhealthy, or
the deployment rotates credentials. The socket supports multiple concurrent
application messages. Responses and progress events are routed by
`correlationId`.

## Envelope

Every client-to-server request, server-to-client event, error, cancellation, and
heartbeat uses:

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

## Heartbeat

Either side may send:

```json
{
  "type": "heartbeat",
  "correlationId": "0198d1ad-9b41-7c87-94a2-6dbdbb7f0602",
  "payload": { "sentAt": "2026-05-06T19:30:00Z" }
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
    "requestedAt": "2026-05-06T19:30:00Z",
    "timeoutMs": 30000,
    "debug": false
  }
}
```

Model and embedding hosts may use numeric priorities. Domain services may use
`interactive`, `background`, and `maintenance`.

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

