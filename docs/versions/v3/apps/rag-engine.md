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

The implementation roadmap is tracked in the RAG repository at
`docs/RAG_ENGINE_ROADMAP.md`. As of the current phase-one branch, the
Utility LLM Host client, PostgreSQL store, ingestion/indexing baseline,
Embedding Service client, embedding worker, and baseline hybrid retrieval
are implemented. Remaining larger work includes shared media resolution
for `imageId`, stronger reranking, provider cost tracking, contract-driven
validation, broader e2e coverage, and larger retrieval evaluations.

## Current Implementation

Current local implementation status:

- Exposes `POST /v2/retrieval/evidence` and `GET /v2/health`.
- Calls the Utility LLM Host over the v2 WebSocket inference stream for
  query support, image observations from `imageUrl`, extraction summaries,
  and evidence shaping.
- Calls the Embedding Service over HTTP on `127.0.0.1:3002` for query and
  document embeddings. Live user-path query embeddings use a higher queue
  priority than background indexing.
- Stores web-derived retrieval knowledge in PostgreSQL database
  `swirlock_rag`, with JSON file fallback only for no-database development
  and tests.
- Queues changed chunks in `rag_embedding_jobs`; a local worker drains the
  queue through the Embedding Service and writes 384-dimensional
  `pgvector` embeddings for model `bge-small-en-v1.5`.
- Local retrieval uses a baseline hybrid strategy that combines
  PostgreSQL full-text/trigram ranking, vector similarity, freshness,
  source quality, and source diversity.
- Retrieval degrades instead of failing when Utility LLM or Embedding
  Service support is unavailable; diagnostics are returned under
  `retrievalDiagnostics.utilityLlm` and
  `retrievalDiagnostics.embeddingService`.
