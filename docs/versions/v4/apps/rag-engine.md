# RAG Engine v4

Endpoint: `WS /v4/retrieval`

## Client Messages

- `retrieve_evidence`: payload is `{ "request": RetrieveEvidenceRequest }`.
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
- `live.search.started`
- `live.search.completed`
- `live.extract.started`
- `live.extract.completed`
- `evidence.chunk`
- `retrieval.completed`
- `retrieval.failed`
- `health`
- `error`
- `heartbeat`

Each retrieval event payload contains `sequence`, `occurredAt`, and `data`.

