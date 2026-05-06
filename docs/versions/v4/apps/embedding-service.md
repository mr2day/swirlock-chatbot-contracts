# Embedding Service v4

Endpoint: `WS /v4/embeddings`

## Client Messages

- `embed`: payload is `{ "request": EmbedRequest }`.
- `health.get`
- `model.status`
- `model.preload`: payload is `{ "request": ModelLifecycleRequest }`.
- `model.unload`: payload is `{ "request": ModelLifecycleRequest }`.
- `cancel`
- `heartbeat`

## Server Events

- `embed.done`
- `health`
- `model.status`
- `model.preload`
- `model.unload`
- `error`
- `heartbeat`

`embed.done` payload contains model id, dimensions, normalization flag, input
type, vectors, generation time, and applied options.

