# Chat Orchestrator

> Repository: `swirlock-chat-orchestrator`
> Contract: [`openapi/chat-orchestrator.openapi.yaml`](../openapi/chat-orchestrator.openapi.yaml)
> Local port (current convention): `3200`

## Role

The Chat Orchestrator owns the user-turn lifecycle. It coordinates memory,
retrieval, prompt assembly, model-host calls, image routing, and the final
client-facing response. It does not own retrieval internals, memory
internals, or model hosting.

## Final Form

When complete, the Chat Orchestrator provides:

- The full client-facing API surface:
  - `POST /v2/chat/sessions` to create a session.
  - WebSocket `/v2/chat/sessions/{sessionId}/turns/stream` for streamed
    `accepted` / `retrieval` / `queued` / `started` / `thinking` /
    `chunk` / `done` / `error` events terminating with persisted-turn
    identifiers (`sessionId`, `turnId`, `assistantMessage.messageId`,
    `createdAt`) and citations.
- Coordination across:
  - Context Fragmenter for memory recording, selection, and consolidation.
  - RAG Engine for retrieval, evidence packaging, and citation propagation
    into `ChatStreamDoneEvent.data`.
  - Primary LLM Host for normal final-answer inference, including direct
    image-input routing when the hosted model advertises `imageInput: true`.
  - Utility LLM Host as explicitly allowed degraded final-answer fallback,
    with the fallback exposed in diagnostics or operational logs.
- v3 envelope responses (`meta` + `data`, or `meta` + `error` on failure)
  on every HTTP response and as the JSON shape inside every WebSocket event.
- Bearer-auth, mTLS, or another deployment-chosen authentication mechanism
  on every chat endpoint, on both HTTP and WebSocket. WebSocket auth follows
  `API_CONVENTIONS.md#websocket-authentication` (header, `?token=` query,
  or `Sec-WebSocket-Protocol: bearer, <token>` subprotocol).
- Stable correlation IDs propagated to every downstream service.
- Optional diagnostics on demand (`includeDiagnostics: true`).
- Multi-user support with the resource-ownership model defined in
  `INTERACTION_MODEL.md` (sessions, turns, and persisted messages bound
  to the authenticated user identity).

## Roadmap

In priority order:

1. **Real authentication.** Replace the single hardcoded dev bearer token
   with a deployment-chosen mechanism (rotated bearer secrets, mTLS, or
   another private-network auth). Token storage moves out of the committed
   `service.config.cjs`.
2. **Multi-user support.** Bind sessions, turns, and persisted messages to
   the authenticated user identity rather than the single dev user.
3. **RAG Engine integration.** Implement the WebSocket client behind the
   existing `RagService` hook, forward retrieval progress as `retrieval`
   chat-stream events, surface returned evidence as `citations` on the
   streaming `done` event, and surface retrieval diagnostics when
   requested.
4. **Context Fragmenter integration.** Call Context Fragmenter for memory
   selection before final inference, and for memory recording after the
   turn is persisted. Stop persisting whole conversations once the
   Fragmenter is the source of truth for selected memory.
5. **Image-input handling.** Resolve `imageId` parts via a deployment-
   provided media resolver. Route original user images directly to the
   Primary LLM Host when the model advertises `imageInput: true`.
6. **Degraded final-answer fallback** to the Utility LLM Host with explicit
   diagnostics when the Primary LLM Host is unavailable.
7. **Operational hardening:** structured access logs, request budgets,
   per-correlation-id tracing, persistent-store backup story.

## Current Implementation

The current implementation is the smallest useful slice of the Final Form.
It is deliberately incomplete and intended to be extended along the roadmap
above.

What is built:

- Single hardcoded dev user, protected by a hardcoded bearer token in
  `service.config.cjs` (single source of truth per
  `INTERNAL_INFRASTRUCTURE.md#runtime-configuration-source-of-truth`).
- Sessions and per-turn user/assistant messages stored whole in a local
  SQLite database. No Context Fragmenter, no memory compression.
- RAG Engine integration over WebSocket. The orchestrator keeps a persistent
  `ws://<rag-engine>/v2/retrieval/evidence/stream` connection, sends one
  `retrieve_evidence` message per retrieval request, forwards each returned
  `RetrievalStreamEvent` for that `correlationId` to the client as a
  `retrieval` chat-stream event, and uses
  `retrieval.completed.data.retrieval` to build the final-answer prompt and
  citations.
- The orchestrator calls a single configured Model Host directly:
  - Persistent WebSocket `/v2/infer/stream` for Utility LLM classification
    and final-answer generation, forwarding `queued` / `started` /
    `thinking` / `chunk` events through to the client for final-answer
    requests and persisting the assembled assistant message before emitting
    its own terminal `done` event.
  - **Operational substitution**: the configured Model Host slot is
    semantically the Primary LLM Host, but in the current local deployment
    the Main Computer is not yet provisioned. The orchestrator therefore
    points at the Utility LLM Host on the local machine as a temporary
    stand-in for Primary. When the Main Computer comes online, the
    orchestrator's `service.config.cjs` `llmHost.baseUrl` is updated to the
    Primary's network URL — no code change is required because the Model
    Host API is agnostic.
- Endpoints implemented:
  - `POST /v2/chat/sessions` (per contract).
  - WebSocket `/v2/chat/sessions/{sessionId}/turns/stream` (per contract,
    streaming).
  - `GET /v2/chat/sessions/{sessionId}` (extension, for local development:
    inspect session and full message history).
  - `DELETE /v2/chat/sessions/{sessionId}` (extension, for local
    development: delete a session and all its messages).
  - `GET /v2/health`.
- v3 envelope shape on all responses, including the `ErrorEnvelope` shape
  on every non-2xx HTTP response and on the WebSocket `error` event.
- `x-correlation-id` is honored when supplied by the client and generated
  otherwise. The same correlation ID is forwarded to the RAG Engine and
  Primary LLM Host on the WebSocket turn path.
- Image input is supported via `imageUrl` and client-ingress `imageBase64`.
  `imageBase64` is forwarded to the configured Model Host for final-answer
  inference and is not forwarded to RAG Engine, whose current contract accepts
  only `imageId` / `imageUrl`. `imageId` resolution is not implemented and is
  rejected with a `bad_request` error.
- WebSocket bearer auth on the upgrade accepts all three transports listed
  in `API_CONVENTIONS.md#websocket-authentication`.
- CORS allow-list for browser clients. The orchestrator's
  `service.config.cjs` carries an `http.corsOrigins` array; on bootstrap the
  app calls `enableCors` with `Authorization`, `Content-Type`, and
  `x-correlation-id` allowed and `x-correlation-id` exposed. The default
  list permits `http://localhost:4200` and `http://127.0.0.1:4200` so the
  Angular dev server of `swirlock-chatbot-ui` can reach the orchestrator
  without proxying. Production deployments narrow the list to the deployed
  UI URL(s).

What is not yet built:

- Real authentication and multi-user identity.
- Context Fragmenter integration and memory selection/recording.
- Degraded final-answer fallback to the Utility LLM Host.
- `imageId` resolution.
- Operational hardening (structured tracing, persistent-store backup).
