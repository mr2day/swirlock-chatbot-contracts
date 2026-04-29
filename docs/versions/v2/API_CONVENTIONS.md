# API Conventions v2

## Purpose

This document defines shared conventions for Swirlock service contracts in `v2`.

The main change from `v1` is that LLM-serving applications are treated as agnostic model hosts. Model hosts expose inference and operational health. Caller services own task semantics.

## Protocol

The `v2` contracts use:

- HTTP
- JSON request and response bodies
- UTF-8 text
- URL versioning with `/v2/...`

Binary multipart uploads are out of scope for cross-service domain APIs.

## Required Headers

All inter-service requests must include:

- `x-correlation-id`

This value is stable across service calls that belong to the same user turn or background job chain.

## Request Body Shape

All `POST` request bodies should contain a top-level `requestContext` object.

Canonical fields:

- `callerService`: logical caller name
- `priority`: `interactive`, `background`, or `maintenance`
- `requestedAt`: ISO 8601 UTC timestamp
- `timeoutMs`: optional caller timeout hint
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
    "message": "maxOutputTokens exceeds host limit",
    "retryable": false,
    "details": {}
  }
}
```

## Priority Semantics

Priorities are:

- `interactive`: live user-path work
- `background`: deferred non-urgent work
- `maintenance`: scheduled system work

Services under load should prefer `interactive` work over `background` and `maintenance` work.

Model hosts may reject or defer lower-priority requests when accepting them would harm model availability for interactive work.

## Media References

Domain service APIs use media references:

- `imageId`
- `imageUrl`

Exactly one of `imageId` or `imageUrl` must be supplied for an image input part.

Model Host APIs may additionally accept direct base64 image payloads because the model host is the final boundary before model inference. A caller with only an `imageId` must resolve it before calling a model host unless that deployment explicitly provides media resolution.

## Model Host Safety

Model hosts own operational safety limits, including:

- max text size
- max image count
- max image bytes
- max output tokens
- max context size
- request timeout
- max concurrent requests
- load/preload/unload behavior

Caller-supplied generation options are hints. A model host may clamp or reject any option that exceeds configured limits.

## Authentication

Authentication details are not fixed by this draft. Payloads and headers should be designed so bearer auth, mTLS, or another private-network mechanism can be added without changing semantic payloads.
