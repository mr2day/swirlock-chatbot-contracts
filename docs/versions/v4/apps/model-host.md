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

`InferRequest.input` accepts either the legacy flat `parts` array, a structured
`messages` array, or both:

```json
{
  "input": {
    "messages": [
      { "role": "system", "content": "Core persona identity..." },
      { "role": "system", "content": "Operational turn context..." },
      { "role": "user", "content": "What is your name?" }
    ],
    "parts": [{ "type": "image", "imageUrl": "https://example.test/image.png" }]
  }
}
```

When `messages` is present, role boundaries must be preserved by the Model Host
when talking to the model runtime. `parts` remains available for binary-like
attachments and for older callers; image parts attach to the last user message.
This lets persona identity and operational context be supplied as model context
without flattening them into the user's conversational text.

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
