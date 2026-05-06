# Swirlock Contracts v4

`v4` is a breaking transport contract.

The ecosystem API surface is WebSocket-only. Every app-to-app relationship keeps
one persistent WebSocket open and multiplexes application messages by
`correlationId`.

## Hard Rules

- No ecosystem REST APIs.
- One persistent WebSocket per service-to-service relationship.
- One persistent WebSocket from UI clients to the Chat Orchestrator.
- Every application frame uses the shared envelope in
  `schemas/envelope.schema.json`.
- No legacy hot-path compatibility endpoints.
- Health, status, and admin commands are WebSocket messages, not REST routes.
- Local ops-only debug surfaces, when present, are not ecosystem APIs and must
  not appear in these contracts.

## Canonical Documents

- `API_CONVENTIONS.md`: envelope, transport, errors, cancellation, heartbeat.
- `INTERACTION_MODEL.md`: message flow across the ecosystem.
- `CHATBOT_MANIFEST.md`: service boundaries and endpoint names.
- `apps/*.md`: app-specific WebSocket message types.
- `schemas/envelope.schema.json`: machine-readable common envelope.

