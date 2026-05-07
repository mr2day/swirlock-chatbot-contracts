# Chat Orchestrator v4

Endpoint: `WS /v4/chat`

## Client Messages

- `session.create`: payload is `{ "request": CreateSessionRequest }`.
- `session.get`: payload is `{ "sessionId": string }`.
- `session.delete`: payload is `{ "sessionId": string }`.
- `turn.submit`: payload is `{ "sessionId": string, "request": SubmitTurnRequest }`.
  `SubmitTurnRequest` may include an optional `userLocation` object
  (`{ latitude, longitude, accuracyMeters?, capturedAt? }`) when the client
  has already obtained location permission and wants the turn to be
  location-accurate without a round-trip.
- `turn.location_response`: client reply to a `turn.location_required` event.
  Payload is one of:
  - `{ "available": true, "location": { "latitude": number, "longitude": number, "accuracyMeters"?: number, "capturedAt"?: string } }`
  - `{ "available": false, "reason": "denied" | "unavailable" }`
  Routed by `correlationId` to the paused turn.
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
- `turn.location_required`: emitted when the Utility LLM classifier
  reports the turn requires the user's real location to be accurate AND
  the client did not already attach a `userLocation` to `turn.submit`.
  The orchestrator pauses the turn pipeline until the client replies
  with `turn.location_response` (or a server-side timeout fires, after
  which the turn continues without location). Payload:
  `{ "requestedAt": string, "timeoutMs": number }`.
- `turn.thinking`
- `turn.chunk`
- `turn.done`
- `error`
- `heartbeat`

`turn.done` contains the persisted assistant message, citations, finish reason,
and optional diagnostics.

