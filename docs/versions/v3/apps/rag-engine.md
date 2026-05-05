# RAG Engine

> Repository: `swirlock-rag-engine`
> Contract: [`openapi/rag-engine.openapi.yaml`](../openapi/rag-engine.openapi.yaml)
> Local port (current convention): `3001`

## Role

The RAG Engine owns external knowledge retrieval and evidence packaging.
It performs retrieval from local knowledge and the live web, evidence
collection and scoring, evidence packaging, and retrieval-oriented
synthesis.

It may use the Utility LLM Host for query interpretation, query rewriting,
image observations, and evidence shaping, and the Embedding Service for
vectorization. It does not own memory semantics, model hosting, or the
final user-facing answer.

## Final Form

- Full retrieval and evidence-packaging API behind the RAG Engine
  contract.
- Local knowledge store with chunked, indexed, and embedding-backed
  retrieval. Current canonical local store is `swirlock_rag` in
  PostgreSQL 17 with `vector`, `pg_trgm`, `unaccent`, and `citext`, on a
  dedicated SSD tablespace
  (`D:\swirlock\postgresql\tablespaces\rag_knowledge`), per
  `INTERNAL_INFRASTRUCTURE.md#local-persistent-stores`.
- Live web retrieval for queries that warrant fresh sources.
- Evidence packaging that the Chat Orchestrator carries through to
  `SubmitTurnResponse.citations` and `ChatStreamDoneEvent.data`.
- May call the Embedding Service directly. Direct calls
  `RAG Engine -> Context Fragmenter` are forbidden by the v3 interaction
  model unless the architecture is intentionally revised.

## Roadmap

The detailed roadmap has not yet been drafted in this contract version.
The PostgreSQL store conventions are pinned in
`INTERNAL_INFRASTRUCTURE.md#local-persistent-stores`.

## Current Implementation

Implementation status not surveyed for this v3 draft. The contract surface
is defined in `openapi/rag-engine.openapi.yaml` and the local durable-store
conventions are pinned in
`INTERNAL_INFRASTRUCTURE.md#local-persistent-stores`. This section will be
filled in when the service is audited against running code.
