# Embedding Service v5

Endpoint: `WS /v5/embeddings`

A generic embedding model host. A single deployment runs a single embedding
model. By the 1:1 module-to-LLM/embedding rule, each consuming module sees
exactly one Embedding Service.

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

## Migration from v4

No semantic changes. Only the endpoint path moves from `/v4/embeddings` to
`/v5/embeddings`. An Embedding Service implementation may serve both during
a migration window.
