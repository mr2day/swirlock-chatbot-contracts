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
  `ChatStreamDoneEvent.data`.
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

- Exposes `GET /v2/health`.
- Exposes WebSocket `/v2/retrieval/evidence/stream` for the ecosystem
  retrieval progress path consumed by Chat Orchestrator.
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

## Streaming Retrieval

The Chat Orchestrator must call WebSocket `/v2/retrieval/evidence/stream`
for ChatGPT-style visible search progress. After upgrade, the
Orchestrator sends exactly one message:

```json
{
  "type": "retrieve_evidence",
  "correlationId": "...",
  "request": { "...": "RetrieveEvidenceRequest" }
}
```

The embedded `request` is a `RetrieveEvidenceRequest`. The RAG Engine then
emits one JSON `RetrievalStreamEvent` per WebSocket message and closes
after `retrieval.completed` or `retrieval.failed`.

The stream is a progress channel, not an answer-generation channel. RAG
does not own the final chatbot answer or final model context window. The
Orchestrator should use early events such as `live.search.completed`,
`live.extract.completed`, and `evidence.chunk` to display "searching" UI
and discovered sources. It should wait for `retrieval.completed` before
building the final answer prompt from the completed evidence package.

Current event types:

- `retrieval.started`
- `utility_llm.retrieval_support.started`
- `utility_llm.retrieval_support.completed`
- `query.normalized`
- `embedding.query.started`
- `embedding.query.completed`
- `local.search.started`
- `local.search.completed`
- `retrieval.policy.decided`
- `live.search.started`
- `live.search.completed`
- `live.extract.started`
- `live.extract.completed`
- `utility_llm.extraction_summaries.started`
- `utility_llm.extraction_summaries.completed`
- `evidence.chunk`
- `utility_llm.evidence_synthesis.started`
- `utility_llm.evidence_synthesis.completed`
- `retrieval.completed`
- `retrieval.failed`

`retrieval.completed` contains `data.retrieval`, whose shape is
`RetrievalPackage` in `openapi/rag-engine.openapi.yaml`.
`retrieval.failed` is emitted when an error occurs after the WebSocket
stream has already been established. If validation fails before the stream
is established, the service emits `retrieval.failed` and closes.
