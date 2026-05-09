# Model Host v5

Endpoint: `WS /v5/model`

A Model Host is a single-process, single-model inference server. It exposes
the same WebSocket contract regardless of which role its deployment plays
(Vanamonde LLM Host, Fragmenter LLM Host, RAG-side LLM Host, etc.). The
contract makes no statement about model size, family, vendor, or capability —
those are deployment-time concerns.

The 1:1 module-to-LLM rule (see `../README.md`) means a Model Host process
serves exactly one consuming module's role at a time. During development a
single physical Model Host may be temporarily shared across multiple roles by
having two consumers point at the same URL, but the contract still treats the
two roles as separate; the deployment is allowed to collapse them physically
when only one model is available.

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

When `messages` is present, role boundaries must be preserved by the Model
Host when talking to the model runtime. `parts` remains available for
binary-like attachments and for older callers; image parts attach to the last
user message. This lets persona identity and operational context be supplied
as model context without flattening them into the user's conversational text.

`InferRequest.options` may include:

- `responseFormat`: `"text" | "json"`. JSON mode is used for control-step and
  classifier inferences where the consumer parses structured output.
- `thinking`: `boolean`. Enables the model's reasoning/thinking mode if the
  underlying runtime supports it.
- `ollama`: object passed through to Ollama-backed runtimes
  (`{ "temperature": 0, ... }`).

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

## Migration from v4

No semantic changes. Only the endpoint path moves from `/v4/model` to
`/v5/model`. A Model Host implementation may serve both `/v4/model` and
`/v5/model` from the same process during a migration window.
