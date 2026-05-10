# Chat Orchestrator v5

Endpoint: `WS /v5/chat`

The Chat Orchestrator owns the user-turn lifecycle: session management, the
agentic conversation flow, prompt assembly, single-LLM inference (Vanamonde
LLM), retrieval coordination via the RAG Engine, and persistence.

It opens exactly two upstream WebSocket connections:

- one to the **Vanamonde LLM Host** (`/v5/model`) — used for every inference
  in the live turn pipeline (control-step decisions and final-answer
  generation);
- one to the **Context Fragmenter** (`/v5/fragmenter`) — used to send
  notifications about session activity and to receive optional
  `consolidation.updated` events.

Plus, when retrieval is enabled:

- one to the **RAG Engine** (`/v5/retrieval`).

The orchestrator does **not** open a connection to the Fragmenter LLM Host.
It does not see the Fragmenter LLM directly; the Fragmenter mediates all
access to its LLM.

## Client Messages

- `session.create`: payload is `{ "request": CreateSessionRequest }`.
  `CreateSessionRequest` may include an optional `persona` object
  (`{ "name": string, "systemPrompt": string }`) — the client app's
  persona variable. The orchestrator stores it on the session and pipes
  `systemPrompt` to the LLM on every turn. The orchestrator does not own
  any persona definition; if `persona` is omitted, the model receives
  no persona system message for that session.
- `model.status`: payload may be omitted. The orchestrator forwards to
  the configured LLM Host and returns the model identity + capabilities
  unchanged. Clients use this to interpolate `${model}` into persona
  prompt templates and to gate model-dependent UI affordances (e.g.
  a "Force thinking" toggle).
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

- `model.status`: payload is
  `{ "modelId": string, "thinkingSupported": boolean }` (e.g.
  `{ "modelId": "gemma3:12b", "thinkingSupported": false }`). Returned
  in response to a client `model.status` request. `thinkingSupported`
  reflects the LLM Host's `runtime.thinkingEnabled` flag; clients use
  it to hide thinking-related UI when the model can't honor it.
- `session.created`: payload is `{ "sessionId", "createdAt", "status" }`.
- `session.snapshot`: payload includes session metadata and persisted messages.
- `session.deleted`: payload is `{ "sessionId", "deleted": true }`.
- `health`: payload is `{ "status", "ready" }`.
- `turn.accepted`
- `turn.classifying`: emitted whenever the orchestrator's agent enters a
  control-step inference. May fire multiple times per turn — once at the
  start of the turn and once before each subsequent agent control step
  (e.g. after a retrieval). Payload may include `{ "step": number }`.
- `turn.retrieval`
- `turn.agent`: agent activity outside of retrieval. Emitted when the agent
  issues a command (`command_started`), when a command finishes
  (`command_completed`), and when a plan is created or progressed
  (`plan`). Payload:
  `{ "phase": "command_started"|"command_completed"|"plan"|"classifying", "command"?: string, "summary": string, "data"?: object }`.
  `summary` is short, user-friendly copy ("Searching: 'current weather Bucharest'", "Plan: Compare prices (3 steps).").
- `turn.queued`
- `turn.started`: the orchestrator is now streaming the user-visible final
  answer. Emitted only when the final-answer infer begins. Agent control-step
  inferences do not emit `turn.started`.
- `turn.location_required`: emitted when the orchestrator's classifier
  reports the turn requires the user's real location to be accurate AND the
  client did not already attach a `userLocation` to `turn.submit`. The
  orchestrator pauses the turn pipeline until the client replies with
  `turn.location_response` (or a server-side timeout fires, after which the
  turn continues without location). Payload:
  `{ "requestedAt": string, "timeoutMs": number }`.
- `turn.thinking`
- `turn.chunk`
- `turn.done`
- `error`
- `heartbeat`

`turn.done` contains the persisted assistant message, citations, finish
reason, and optional diagnostics.

## Notes On The Vanamonde LLM Connection

The orchestrator opens exactly one Model Host socket. Every inference in a
turn uses the same socket:

- **Control-step inferences** use `responseFormat: "json"`,
  `temperature: 0`, brief output budget. They produce structured tool
  decisions, never user-visible prose.
- **Final-answer inference** uses `responseFormat: "text"`, full response
  budget, streaming chunks. It produces the user-visible response.

The contract makes no statement about whether these two call modes use the
same model variant; the deployment may serve both from one model.

## Notes On The Context Fragmenter Connection

The orchestrator's interaction with the Fragmenter is intentionally minimal
and never blocking. See `context-fragmenter.md` for the full message set.

The orchestrator never queries the Fragmenter for consolidation results.
Consolidation is read directly from the shared SQLite file's Fragmenter-owned
result tables during prompt assembly.

## Migration from v4

- Endpoint path moves from `/v4/chat` to `/v5/chat`. An orchestrator
  implementation may serve both during the migration window.
- The orchestrator drops its `utilityLlmHost` config slot. It opens a single
  Model Host connection (the Vanamonde LLM Host). Any service that previously
  pointed at the orchestrator's Utility LLM should redirect to its own LLM
  Host (most commonly: the RAG Engine, which now manages its own RAG-side
  LLM Host directly).
- A new `turn.agent` server event is introduced.
- `turn.classifying`'s multi-emit semantics are documented (see above).
- `turn.started` semantics are tightened: it fires only when the
  final-answer inference begins.
