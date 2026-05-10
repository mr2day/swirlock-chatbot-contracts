# Swirlock Contracts v5

`v5` codifies the architectural lesson learned in late v4: a single conversation
turn is a strict data-dependency chain, so cooperating LLMs cannot speed it up.
The orchestrator's "Primary + Utility" two-LLM design was operationally
incoherent and is removed in v5.

In its place, v5 introduces a **1:1 module-to-LLM binding** as a deliberate
architectural rule. Each module that needs an LLM owns exactly one, named by
the role it plays in that module:

- The **Vanamonde LLM** is the conversation-side LLM consumed by the Chat
  Orchestrator. It perceives the user input, reasons about it, and produces
  the response that the orchestrator streams back to the client. Named after
  the disembodied cosmic intelligence in Arthur C. Clarke's *The City and the
  Stars* (1956): pure perception and thought, embodied differently in every
  frontend (chatbot UI, coding agent, robot, character avatar, figurine,
  service-bot).
- The **Fragmenter LLM** is the consolidation-side LLM consumed by the Context
  Fragmenter. It runs in the background, summarises conversation history,
  cleans transcription artefacts, extracts long-term memory candidates, and
  reorganises the data the Vanamonde LLM will eventually see on its next
  turn.

Both deployments use the same **Model Host** contract
(`apps/model-host.md`) — they differ only in which model they run and which
module they serve.

## Hard Rules (carried over from v4)

- No ecosystem REST APIs.
- One persistent WebSocket per service-to-service relationship.
- Every application frame uses the shared envelope in
  `schemas/envelope.schema.json`.
- No deterministic word, phrase, or regular-expression filtering may alter or
  suppress conversational user or assistant text. Semantic text evaluation
  must be performed by an LLM through a Model Host call. See
  `INTERACTION_MODEL.md` § "Conversation Text Integrity" for the precise rule.

## New rules introduced in v5

- **1:1 module-to-LLM binding.** Each module that consumes an LLM consumes
  exactly one. A module never decides at runtime which of multiple LLMs to
  call. New roles get new module + new LLM.
- **No second LLM in the conversation hot path.** The Chat Orchestrator opens
  exactly one Model Host connection (the Vanamonde LLM Host) and uses it for
  every step of the live turn lifecycle (classification, control-step
  decisions, retrieval-query refinement when it exists, final-answer
  generation). Speculative parallelism is not part of the architecture.
- **Background work belongs to the Context Fragmenter, not the Orchestrator.**
  Memory consolidation, long-term-memory extraction, transcription cleanup,
  and persona-level maintenance run in a peer process consuming the Fragmenter
  LLM. The Orchestrator never invokes the Fragmenter LLM.
- **Orchestrator ↔ Fragmenter coordination is via shared SQLite, not RPC.**
  The two modules share a single SQLite database file with table-level
  ownership. The Orchestrator owns the live conversation tables; the
  Fragmenter owns its working tables and a small set of result tables that
  the Orchestrator reads. The Fragmenter never blocks the Orchestrator's turn
  pipeline; the Orchestrator reads whatever consolidation has been pre-computed
  by the time it builds a prompt, and degrades gracefully when nothing is
  available.
- **Co-location is an architectural assumption.** The Chat Orchestrator and
  the Context Fragmenter run on the same machine because they share a SQLite
  file. Other ecosystem apps (Model Host, RAG Engine, Embedding Service) may
  be on different machines.

## Canonical Documents

- `API_CONVENTIONS.md`: envelope, transport, errors, cancellation, heartbeat.
- `INTERACTION_MODEL.md`: message flow across the ecosystem, conversation
  text integrity rule, Fragmenter coordination model.
- `CHATBOT_MANIFEST.md`: service boundaries, endpoint registry, module-to-LLM
  bindings.
- `apps/*.md`: app-specific WebSocket message types. `apps/idp-base.md`
  documents the Identity Provider, the only ecosystem app that speaks plain
  HTTP rather than the WebSocket envelope (it is an OpenID Connect
  provider; see that file for the rationale).
- `schemas/envelope.schema.json`: machine-readable common envelope.

## Migration from v4

`v5` is a clean break. The `v4` directory remains for archival. Apps migrate
when they are ready; both versions can coexist in the running deployment as
long as each app is internally consistent. Notes on what changes for each app
are in its `apps/*.md` file under "Migration from v4".
