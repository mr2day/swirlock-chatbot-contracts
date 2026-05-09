# RAG Engine v5

Endpoint: `WS /v5/retrieval`

The RAG Engine owns retrieval planning, local knowledge search, live web
search, evidence packaging, and citation data. Per the 1:1 module-to-LLM rule
(see `../README.md`), it consumes its own **RAG-side LLM Host** for query
refinement and document retention decisions, and its own
**Embedding Service** for vector search. Neither dependency is shared with
the Chat Orchestrator or the Context Fragmenter.

## Client Messages

- `retrieve_evidence`: payload is `{ "request": RetrieveEvidenceRequest }`.
  `RetrieveEvidenceRequest.query` may include an optional `userLocation`
  object (`{ latitude, longitude, accuracyMeters?, capturedAt? }`) when
  the caller has obtained location permission from the user. The RAG Engine
  forwards it to its own LLM so retrieval queries can be made
  location-accurate. The RAG Engine never derives or guesses a location on
  its own.
- `health.get`
- `cancel`
- `heartbeat`

## Server Events

Retrieval progress events are emitted as normal envelope `type` values:

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
- `utility_llm.document_retention.started`
- `utility_llm.document_retention.completed`
- `evidence.chunk`
- `retrieval.completed`
- `retrieval.failed`
- `health`
- `error`
- `heartbeat`

Each retrieval event payload contains `sequence`, `occurredAt`, and `data`.

`live.search.*` and `live.extract.*` events fire once per live retrieval
provider (currently `exa` and `wikipedia`) and include
`data.provider: "exa" | "wikipedia"` so callers can disambiguate.
`utility_llm.document_retention.*` events report the LLM-governed retention
pass that decides which live documents to persist into the local knowledge
store.

## Note On The `utility_llm.*` Event Names

The `utility_llm.*` event-type prefix is preserved in v5 for backwards
compatibility with existing callers. The terminology refers to the **role**
of the LLM (assisting retrieval, summarising documents, deciding retention)
rather than the deprecated v4 "Utility LLM" deployment label. In v5
deployment terms, these calls go to the RAG Engine's own LLM Host (the
"RAG-side LLM Host" in `CHATBOT_MANIFEST.md`).

A future minor revision may rename the prefix to `assist_llm.*` or
`rag_llm.*` once existing callers have migrated; until then the v4 names
remain stable.

## Migration from v4

- Endpoint path moves from `/v4/retrieval` to `/v5/retrieval`. A RAG Engine
  implementation may serve both during the migration window.
- Conceptually, the LLM the RAG Engine talks to is no longer "the
  orchestrator's Utility LLM"; it is the RAG Engine's own LLM Host. Existing
  deployments may continue to physically point at the same Model Host process
  during transition (per `CHATBOT_MANIFEST.md` § "Module-to-LLM Bindings"),
  but the contract treats it as RAG's own dependency.
- No event-type renames in v5; all v4 event types are preserved.
