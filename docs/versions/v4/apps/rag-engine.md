# RAG Engine v4

Endpoint: `WS /v4/retrieval`

## Client Messages

- `retrieve_evidence`: payload is `{ "request": RetrieveEvidenceRequest }`.
  `RetrieveEvidenceRequest.query` may include an optional `userLocation`
  object (`{ latitude, longitude, accuracyMeters?, capturedAt? }`) when
  the caller has obtained location permission from the user. The RAG
  Engine forwards it to the Utility LLM so retrieval queries can be
  made location-accurate. The RAG Engine never derives or guesses a
  location on its own.
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
- `utility_llm.extraction_summaries.started`
- `utility_llm.extraction_summaries.completed`
- `evidence.chunk`
- `retrieval.completed`
- `retrieval.failed`
- `health`
- `error`
- `heartbeat`

Each retrieval event payload contains `sequence`, `occurredAt`, and `data`.

