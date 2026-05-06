# Swirlock Chatbot Contracts v3

`v3` keeps the architectural stance and semantic surface of `v2` (LLM-serving
apps are agnostic model hosts; orchestration, retrieval, and memory remain
separate domains). The change is structural: every Swirlock app gets one
canonical document under `apps/`.

## What changed from v2

The cross-cutting documents — `CHATBOT_MANIFEST.md`, `INTERACTION_MODEL.md`,
`API_CONVENTIONS.md`, `INTERNAL_INFRASTRUCTURE.md` — and the OpenAPI files
keep their semantic content from v2. Cross-service API URL versioning stays
at `/v2/...` because the API itself has not changed.

The new piece is `apps/`. Each Swirlock app has one document combining:

1. **Final Form** — the aspirational state the app should reach when its
   roadmap is complete. Replaces v2 prose that scattered per-app
   responsibilities across `CHATBOT_MANIFEST.md` and `INTERACTION_MODEL.md`.
2. **Roadmap** — the work items remaining to reach Final Form, ordered when
   the order is meaningful.
3. **Current Implementation** — what the app actually does today.
   Non-binding until verified against the running code, but useful when
   callers and contributors need to know what the app can do right now.

The OpenAPI files remain the canonical machine-readable contract for each
surface.

## Normative Sources

In conflict-resolution order:

1. `CHATBOT_MANIFEST.md` — system vision, philosophy, service boundaries.
2. `openapi/*.openapi.yaml` — exact field shapes and endpoint details.
3. `INTERACTION_MODEL.md` — interaction model and routing rules.
4. `API_CONVENTIONS.md` — shared protocol/payload conventions.
5. `INTERNAL_INFRASTRUCTURE.md` — local deployment, ports, and durable
   stores.
6. `apps/*.md` — per-app aspirational, roadmap, and current state.

In a conflict between an `apps/<name>.md` Final Form section and the
canonical OpenAPI for that app, the OpenAPI wins. In a conflict between an
`apps/<name>.md` Current Implementation section and the actual running
code, the running code wins until the document is updated.

## Apps

- [Chatbot UI](apps/chatbot-ui.md) — `swirlock-chatbot-ui`, the
  user-facing client (visible name **Gigi the Robot**).
- [Chat Orchestrator](apps/chat-orchestrator.md)
- [Context Fragmenter](apps/context-fragmenter.md)
- [RAG Engine](apps/rag-engine.md)
- [Primary LLM Host](apps/primary-llm-host.md)
- [Utility LLM Host](apps/utility-llm-host.md)
- [Embedding Service](apps/embedding-service.md)

## OpenAPI Files

- `openapi/chat-orchestrator.openapi.yaml`
- `openapi/context-fragmenter.openapi.yaml`
- `openapi/rag-engine.openapi.yaml`
- `openapi/model-host.openapi.yaml` (implemented by both Primary LLM Host
  and Utility LLM Host)
- `openapi/embedding-service.openapi.yaml`
