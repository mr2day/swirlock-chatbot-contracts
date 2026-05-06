# Model Host v4

Endpoint: `WS /v4/model`

## Client Messages

- `infer`: payload is `{ "request": InferRequest }`.
- `health.get`
- `model.status`
- `model.preload`: payload is `{ "request": ModelLifecycleRequest }`.
- `model.unload`: payload is `{ "request": ModelLifecycleRequest }`.
- `cancel`
- `heartbeat`

## Server Events

- `accepted`
- `queued`
- `started`
- `thinking`
- `chunk`
- `done`
- `health`
- `model.status`
- `model.preload`
- `model.unload`
- `error`
- `heartbeat`

Inference events use the same `correlationId` as the `infer` message. `chunk`
payload is `{ "text": string }`. `done` payload includes `finishReason` and
`appliedOptions`.

