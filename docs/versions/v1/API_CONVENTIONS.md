# API Conventions

## Purpose

This document defines the shared conventions used by the Swirlock chatbot service contracts.

These conventions apply across:

- Chat Orchestrator
- Context Fragmenter
- RAG Engine
- Main LLM Server

## Protocol

The `v1` contracts use:

- HTTP
- JSON request and response bodies
- UTF-8 text
- URL versioning with `/v1/...`

Binary uploads are out of scope for `v1`.
Images should be referenced by identifier or URL, not sent as raw bytes.

## Versioning

The first contract generation uses `v1`.

Rules:

- additive fields are allowed within `v1`
- field removals are not allowed within `v1`
- semantic meaning of existing fields must not silently change
- breaking changes require a new API version

## Required Headers

All inter-service requests must include:

- `x-correlation-id`

This value must be stable across all service calls that belong to the same user turn or background job chain.

## Recommended Headers

Recommended where relevant:

- `x-idempotency-key` for mutating `POST` operations

This is especially important for:

- session creation
- turn recording
- sleep job creation

## Request Body Shape

All `POST` request bodies should contain a top-level `requestContext` object.

Canonical fields:

- `callerService`: the logical caller name
- `priority`: `interactive`, `background`, or `maintenance`
- `requestedAt`: ISO 8601 UTC timestamp
- `timeoutMs`: optional caller timeout hint
- `debug`: optional boolean

This object communicates semantic execution intent.
It does not replace transport-level tracing.

## Success Response Shape

Successful responses use a common envelope:

```json
{
  "meta": {
    "requestId": "0196f9e8-71b6-7dc0-8d2c-b0b3c4567890",
    "correlationId": "0196f9e8-65d3-7c91-a14e-0e1f12345678",
    "apiVersion": "v1",
    "servedAt": "2026-04-23T19:45:00Z"
  },
  "data": {}
}
```

## Error Response Shape

Errors use a common envelope:

```json
{
  "meta": {
    "requestId": "0196f9e8-71b6-7dc0-8d2c-b0b3c4567890",
    "correlationId": "0196f9e8-65d3-7c91-a14e-0e1f12345678",
    "apiVersion": "v1",
    "servedAt": "2026-04-23T19:45:00Z"
  },
  "error": {
    "code": "validation_failed",
    "message": "maxEvidenceChunks must be >= 1",
    "retryable": false,
    "details": {}
  }
}
```

## Canonical ID Format

The canonical ID format for cross-service identifiers is `UUIDv7` as a string.

This applies to values such as:

- `requestId`
- `sessionId`
- `turnId`
- `fragmentId`
- `evidenceId`
- `jobId`

Services may use other internal identifiers internally, but cross-service IDs should be treated as opaque strings by callers.

## Timestamp Format

All timestamps must be:

- ISO 8601
- UTC
- emitted with a trailing `Z`

Example:

- `2026-04-23T19:45:00Z`

## Priority Semantics

The canonical priorities are:

- `interactive`
- `background`
- `maintenance`

Meaning:

- `interactive`: directly serving a live user turn
- `background`: delayed or asynchronous non-urgent work
- `maintenance`: scheduled system work such as sleep consolidation

If a service must choose between work classes, `interactive` work should win over `background` and `maintenance`.

## Error Codes

Recommended canonical error codes:

- `bad_request`
- `validation_failed`
- `unauthorized`
- `forbidden`
- `not_found`
- `conflict`
- `rate_limited`
- `timeout`
- `upstream_unavailable`
- `internal_error`

The `retryable` flag should indicate whether an automatic retry is reasonable.

## Idempotency

Operations that create or mutate persistent state should support idempotent retry behavior when `x-idempotency-key` is supplied.

Examples:

- creating a chat session
- recording a completed turn
- creating a sleep job

## Media References

For `v1`, images must be referenced through one of:

- `imageId`
- `imageUrl`

The contract should not assume raw binary uploads.

## Authentication

Authentication and authorization transport details are not fixed by this draft.

The contract set assumes the services operate in a trusted private network in early development, but all APIs should still be written so that bearer auth, mTLS, or another mechanism can be added later without changing payload semantics.

## Compatibility Rule

If a service wants to expose additional internal fields for diagnostics, those fields must be optional and must not become required without a version change.
