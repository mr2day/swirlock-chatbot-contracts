# API Conventions v3

## Purpose

This document defines shared conventions for Swirlock service contracts in `v3`.

`v3` keeps every convention defined in `v2`. The change is structural: per-app contract docs live under `apps/`. URL versioning of cross-service APIs stays at `/v2/...` because the API surface has not changed.

## Protocol

The `v3` contracts use:

- HTTP
- JSON request and response bodies
- UTF-8 text
- URL versioning with `/v2/...` (unchanged from `v2`)

Binary multipart uploads are out of scope for cross-service domain APIs.

## Runtime Configuration

Each service implementation must define runtime configuration in one obvious source of truth and make
the application runtime and process manager read from that same source. Configuration values such as
host, port, upstream URLs, model IDs, allow-lists, keep-alive behavior, and process-manager
environment values must not be copied across `.env`, `.env.example`, source defaults, README
examples, and deployment files.

For the implementation pattern and failure expectations, see
`INTERNAL_INFRASTRUCTURE.md#runtime-configuration-source-of-truth`.

## Required Headers

All inter-service requests must include:

- `x-correlation-id`

This value is stable across service calls that belong to the same user turn or background job chain.

## Request Body Shape

All `POST` request bodies should contain a top-level `requestContext` object.

Canonical fields:

- `callerService`: logical caller name
- `priority`: optional priority hint; exact type is service-specific
- `requestedAt`: ISO 8601 UTC timestamp
- `timeoutMs`: optional caller timeout hint only where a specific service contract defines it; model
  hosts do not impose elapsed-time limits
- `debug`: optional boolean

## Success Response Shape

Successful responses use:

```json
{
  "meta": {
    "requestId": "0196f9e8-71b6-7dc0-8d2c-b0b3c4567890",
    "correlationId": "0196f9e8-65d3-7c91-a14e-0e1f12345678",
    "apiVersion": "v2",
    "servedAt": "2026-04-23T19:45:00Z"
  },
  "data": {}
}
```

## Error Response Shape

Errors use:

```json
{
  "meta": {
    "requestId": "0196f9e8-71b6-7dc0-8d2c-b0b3c4567890",
    "correlationId": "0196f9e8-65d3-7c91-a14e-0e1f12345678",
    "apiVersion": "v2",
    "servedAt": "2026-04-23T19:45:00Z"
  },
  "error": {
    "code": "validation_failed",
    "message": "requestContext.requestedAt must be an ISO 8601 UTC timestamp",
    "retryable": false,
    "details": {}
  }
}
```

## Priority Semantics

Domain services may use named priorities where the contract defines them:

- `interactive`: live user-path work
- `background`: deferred non-urgent work
- `maintenance`: scheduled system work

Services under load should prefer live user-path work over background and maintenance work.

Model hosts use numeric priority because they are agnostic model appliances. For model-host
requests, higher finite numbers run first. If `requestContext.priority` is omitted, the request is
treated as lower priority than every request that provides a finite priority number. Equal-priority
requests, including omitted-priority requests, run in arrival order.

## Model Host Queue Events

The model-host WebSocket stream accepts one inference request per connection. The client sends a
`StreamInferMessage` with:

- `type`: `infer`
- `correlationId`: stable request or turn id
- `request`: an `InferRequest`

If the model slot is busy, the request remains accepted and the host sends `queued` events while it
waits. These events let the client decide whether to keep waiting or close the WebSocket. A queued
request is not rejected and does not need a retry registration number.

Queued event data contains:

- `position`: current 1-based position inside the waiting queue
- `requestsAhead`: active request plus queued requests ahead
- `queueDepth`: current total queued requests
- `defaultPriority`: `true` when the caller omitted `requestContext.priority`
- `priority`: present only when the caller provided `requestContext.priority`
- `averageRequestDurationMs`, `estimatedWaitMs`, and `estimatedStartAt` when recent runtime samples exist

## Media References

Domain service APIs use media references:

- `imageId`
- `imageUrl`

Exactly one of `imageId` or `imageUrl` must be supplied for an image input part.

Model Host APIs may additionally accept direct base64 image payloads because the model host is the final boundary before model inference. A caller with only an `imageId` must resolve it before calling a model host unless that deployment explicitly provides media resolution.

## Model Host Safety

Model hosts own model-health protections, including:

- load/preload/unload behavior
- keep-alive behavior
- single-model-slot serialization, unless the host explicitly documents otherwise
- queue reporting
- process-level request body protection

Model hosts should not invent task-level limits such as output token caps, elapsed-time caps, or
domain-specific input limits unless the hosted model, transport, or machine health requires them.
Caller services own task semantics and caller-side policy.

## Authentication

Authentication details are not fixed by this draft. Payloads and headers should be designed so bearer auth, mTLS, or another private-network mechanism can be added without changing semantic payloads.

## WebSocket Authentication

When a service exposes a WebSocket endpoint, bearer-token auth must be acceptable on the upgrade
request through any of the following transports:

1. `Authorization: Bearer <token>` request header. Preferred for any non-browser caller.
2. `?token=<token>` query parameter. Required for browsers, since the standard `WebSocket`
   constructor cannot set custom headers.
3. `Sec-WebSocket-Protocol: bearer, <token>` subprotocol. Browser-friendly alternative via
   `new WebSocket(url, ['bearer', '<token>'])`. The server should echo `bearer` back as the
   negotiated subprotocol when it accepts the upgrade.

A service may accept any subset of these transports. If it accepts only a subset, its OpenAPI
description must say which ones. The first WebSocket message after upgrade must not carry the
bearer token; the token belongs on the upgrade handshake, not in the application protocol.

Tokens passed in `?token=` will appear in HTTP server logs and access logs. Deployments that
consider that unacceptable should disable the query transport and require the header or
subprotocol form.

A failed token check should respond `401` to the upgrade request and close the TCP connection
without completing the WebSocket handshake. Authorization failures discovered after a successful
upgrade, such as an authenticated user who does not own the resource referenced in the URL,
should be reported as a service-defined `error` event on the open WebSocket before the server
closes the connection.
