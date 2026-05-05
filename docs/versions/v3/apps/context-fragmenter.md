# Context Fragmenter

> Repository: `swirlock-context-fragmenter` *(by convention; not surveyed)*
> Contract: [`openapi/context-fragmenter.openapi.yaml`](../openapi/context-fragmenter.openapi.yaml)

## Role

The Context Fragmenter owns conversational and identity memory. It performs
memory extraction, classification, selection, consolidation, and sleep
jobs. It may use the Utility LLM Host for salience detection, memory
extraction, compression, classification, and consolidation support, and the
Embedding Service for vectorization.

It does not own web retrieval, model hosting, or final-answer ownership.
Direct calls from `RAG Engine -> Context Fragmenter` and `Context Fragmenter -> RAG Engine`
are forbidden by the v3 interaction model unless the architecture is
intentionally revised.

## Final Form

- Full memory lifecycle behind the Context Fragmenter API.
- Memory selection on demand for the Chat Orchestrator on every user turn.
- Memory recording for every completed turn the Chat Orchestrator forwards.
- Background consolidation and sleep jobs that submit Utility LLM Host
  requests with low or omitted `requestContext.priority` so live
  user-path work moves ahead in the queue.
- Vectorization through the Embedding Service.
- Owns its own internal durable store for memory fragments and metadata.
  This store is implementation-internal and must not be collapsed with
  the RAG Engine's web-derived knowledge store.
- Identity memory in addition to conversational memory.

## Roadmap

The detailed roadmap has not yet been drafted in this contract version.
The expected high-level milestones, in rough order, follow from the
service boundary defined in `CHATBOT_MANIFEST.md` and
`INTERACTION_MODEL.md`:

1. Memory recording API and persistence.
2. Memory selection API used by the Chat Orchestrator on each turn.
3. Salience detection, classification, and compression via the Utility
   LLM Host.
4. Sleep / consolidation background jobs at low or omitted model-host
   priority.
5. Identity memory.

## Current Implementation

Implementation status not surveyed for this v3 draft. The contract surface
is defined in `openapi/context-fragmenter.openapi.yaml`. This section will
be filled in when the service is audited against running code.
