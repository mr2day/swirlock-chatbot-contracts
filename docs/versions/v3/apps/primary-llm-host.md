# Primary LLM Host

> Repository: `swirlock-llm-host` (shared with the Utility LLM Host)
> Contract: [`openapi/model-host.openapi.yaml`](../openapi/model-host.openapi.yaml)
> Local port (current convention): `3213`

## Role

The Primary LLM Host owns the health and serving boundary for the primary
model. It accepts the input types advertised by its model-host capabilities
and returns text output. In the normal live user path, the Chat Orchestrator
uses this host for final response inference. When the hosted model
advertises `imageInput: true`, the Chat Orchestrator should pass original
user images directly to this host instead of requiring a Utility LLM Host
pre-pass.

It does not own chat semantics, RAG semantics, memory semantics, prompt
construction policy, or final user-facing answer ownership.

## Final Form

- Full Model Host API:
  - WebSocket `/v2/infer/stream` for streamed `accepted` / `queued` /
    `started` / `thinking` / `chunk` / `done` / `error` events.
  - `GET /v2/health` for liveness/readiness.
  - `GET /v2/model/status` for hosted model status, capabilities, and
    capacity.
  - `POST /v2/model/preload` and `POST /v2/model/unload` for lifecycle
    control.
- Single-slot model serialization with priority-aware queueing and visible
  queue events on the streaming endpoint. Higher finite numeric
  `requestContext.priority` runs first; omitted priority is lower than
  every finite priority; equal priorities run in arrival order.
- Keep-alive, preload, and unload lifecycle behavior.
- Health/readiness reporting and capability advertising
  (`textInput` / `imageInput` / `textOutput` / `imageOutput`).
- Hosted model: `gemma4:e4b` on Ollama, with text and image input enabled.
- Runs on the Main Computer, dedicated to live final-answer inference.
- Owns no semantic ownership of chat, RAG, memory, or prompt policy beyond
  local safety/system guardrails.

## Roadmap

Codebase finished. The remaining work is operational: provision the Main
Computer deployment so the Primary role has a real backing host instead of
the temporary Utility LLM Host substitution described below.

## Current Implementation

The `swirlock-llm-host` codebase implements the Model Host API and is
considered finished. The Primary LLM Host **deployment** on the Main
Computer is not yet provisioned. Until it is, the Chat Orchestrator points
its final-answer Model Host slot at the Utility LLM Host on the local
machine as a temporary stand-in for Primary. The substitution is config-only
and lives in the Chat Orchestrator's `service.config.cjs`; no caller
contract changes when the substitution is reverted.

When the Main Computer comes online, the operational handoff is:

- Stand up `swirlock-llm-host` on the Main Computer with the chosen primary
  model (e.g. `gemma4:e4b` on Ollama, or whichever model the deployment
  picks at the time).
- Update the Chat Orchestrator's `llmHost.baseUrl` in `service.config.cjs`
  to the Main Computer's network URL.
- Leave the Utility LLM Host on the local machine running unchanged; it
  resumes its support-cognition role for RAG Engine and Context Fragmenter.
