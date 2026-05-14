# RAG Engine v5

Endpoint: `WS /v5/retrieval`

The RAG Engine owns retrieval planning, local knowledge search, live web
search, evidence packaging, and citation data. Per the 1:1 module-to-LLM rule
(see `../README.md`), it consumes its own **RAG-side LLM Host** for utility
tasks (when used) and its own **Embedding Service** for vector search.
Neither dependency is shared with the Chat Orchestrator or the Context
Fragmenter.

## Client Messages

- `retrieve_evidence`: payload is `{ "request": RetrieveEvidenceRequest }`.
  Full-pipeline retrieval intended for user-facing answer generation —
  produces ranked evidence chunks with citation metadata and emits a stream
  of progress events the caller can surface to its own UI.
  `RetrieveEvidenceRequest.query` may include an optional `userLocation`
  object (`{ latitude, longitude, accuracyMeters?, capturedAt? }`) when
  the caller has obtained location permission from the user. The RAG Engine
  forwards it to its own LLM so retrieval queries can be made
  location-accurate. The RAG Engine never derives or guesses a location on
  its own.
- `search.run`: payload is `{ "request": SearchRunRequest }`. Thin,
  non-streaming search-only capability for internal callers (notably
  the Context Fragmenter's reality-drift audit pass). Returns cleaned
  query-targeted highlights without evidence-chunk packaging, scoring
  rubrics, or citation-id generation. See "`search.run`" below.
- `health.get`
- `cancel`
- `heartbeat`

## `search.run`

A thin search capability for callers that want raw web material
without the full evidence-packaging pipeline. The primary consumer
is the Context Fragmenter, which uses it to spot-check assistant
claims during reality-drift audits. Other internal callers (future
agentic primitives) may use it similarly.

`search.run` exists so internal callers can ask the RAG Engine for
raw search material without paying for or coordinating with the full
evidence pipeline, and without duplicating Exa-client logic in
multiple repositories.

### Request

```json
{
  "type": "search.run",
  "correlationId": "<uuid>",
  "payload": {
    "request": {
      "requestContext": {
        "callerService": "context-fragmenter",
        "priority": "maintenance",
        "requestedAt": "2026-05-14T12:00:00Z",
        "timeoutMs": 30000
      },
      "query": {
        "queryText": "string — required, the search query to run",
        "extractLimit": 2,
        "freshness": "medium"
      }
    }
  }
}
```

Field rules:

- `query.queryText` (required, non-empty string): the search query.
  The caller is responsible for any phrasing — `search.run` does
  not run an LLM-side query-refinement pass.
- `query.extractLimit` (optional, integer; default 2, valid range
  1–5): how many top results to extract highlights from.
- `query.freshness` (optional, `"low" | "medium" | "high" | "realtime"`;
  default `"medium"`): same semantics as `retrieve_evidence`.
- `requestContext.priority`: callers SHOULD use `background` or
  `maintenance` for `search.run`. The RAG Engine may de-prioritise
  `search.run` traffic behind `retrieve_evidence` traffic from the
  same client; this is implementation-defined.

### Response

A single `search.completed` event is emitted for the request's
`correlationId`. No progress events are emitted (this is the
intentional contract difference from `retrieve_evidence`).

```json
{
  "type": "search.completed",
  "correlationId": "<uuid>",
  "payload": {
    "sequence": 1,
    "occurredAt": "2026-05-14T12:00:01.234Z",
    "data": {
      "queryText": "the query as actually executed",
      "results": [
        {
          "url": "https://example.com/...",
          "title": "...",
          "highlight": "cleaned, query-targeted excerpt",
          "publishedAt": "2026-04-01T00:00:00Z",
          "relevanceScore": 0.82
        }
      ],
      "diagnostics": {
        "extractLimit": 2,
        "resultCount": 1,
        "durationMs": 1234,
        "providerRequestId": "..."
      }
    }
  }
}
```

Field rules:

- `data.results` is an ordered array; result 0 is the highest-ranked
  by the underlying provider. May be empty.
- `data.results[].highlight` is cleaned by the RAG Engine's
  content-cleaner before return — the same cleaner used inside
  `retrieve_evidence`. Max length is implementation-defined but
  bounded (current cap: ~1200 characters).
- `data.results[].publishedAt` is null when the provider did not
  return a publication date.
- `data.diagnostics.providerRequestId` is present only when the
  underlying provider returns one (Exa does; future providers may
  not).

On error, the standard ecosystem `error` envelope is emitted for
the same `correlationId`; no `search.completed` follows.

### Contract differences from `retrieve_evidence`

- **No progress events.** `search.run` is request → single response.
  Callers do not need to subscribe to `retrieval.*`, `live.*`, or
  `embedding.*` event sequences.
- **No persona-style packaging.** Results are raw
  `{url, title, highlight, publishedAt, relevanceScore}` tuples — no
  `evidenceId` generation, no freshness/relevance composite scoring,
  no citation rendering.
- **No local-knowledge search.** `search.run` always targets the
  live web. Local-knowledge consultation is a `retrieve_evidence`
  concern.
- **No `userLocation`.** Callers needing location accuracy should
  encode the place name into `queryText` themselves.
- **No LLM-side query refinement.** The provided `queryText` is
  passed through to the live provider as-is.

## Server Events

For `retrieve_evidence`:

- `retrieval.started`
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
- `evidence.chunk`
- `retrieval.completed`
- `retrieval.failed`

For `search.run`:

- `search.completed`
- (no progress events)

Common to both / ecosystem-standard:

- `health`
- `error`
- `heartbeat`

Each progress-event payload contains `sequence`, `occurredAt`, and `data`.

`live.search.*` and `live.extract.*` events fire once per live retrieval
provider and include `data.provider` so callers can disambiguate. Current
provider set: `exa`.

## Amendments since v5 initial publication

The original v5 publication of this document listed event types and
providers that no longer exist after subsequent RAG Engine cleanup.
Amendments below are additive (new behaviour) or corrective (removed
behaviour no longer documented). Callers that depended on any
removed event should treat it as silently absent — no replacement to
listen for.

- **Added `search.run` client message and `search.completed` server
  event.** A thin, non-streaming search capability for internal
  callers (Context Fragmenter audits, future agentic primitives).
  Additive only; no existing message type or event is affected.
- **Removed `utility_llm.retrieval_support.*` events.** The Planning
  Search pass that emitted these was removed; the orchestrator's
  classifier now provides the search query directly.
- **Removed `utility_llm.extraction_summaries.*` events.** The
  per-document Summarize pass was removed; the persona LLM
  synthesises directly from Exa highlights at answer time.
- **Removed `utility_llm.document_retention.*` events.** The
  retention pass was removed; the knowledge store is not currently
  written to from live retrieval.
- **Removed `wikipedia` as a `live.*` provider.** The Wikipedia
  integration was deleted; `data.provider` on `live.*` events is
  now always `exa`.

## Migration from v4

- Endpoint path moves from `/v4/retrieval` to `/v5/retrieval`. A RAG Engine
  implementation may serve both during the migration window.
- Conceptually, the LLM the RAG Engine talks to is no longer "the
  orchestrator's Utility LLM"; it is the RAG Engine's own LLM Host.
  Existing deployments may continue to physically point at the same Model
  Host process during transition (per `CHATBOT_MANIFEST.md`
  § "Module-to-LLM Bindings"), but the contract treats it as RAG's own
  dependency.
- No event-type renames in v5; v4 event types are preserved except for
  those explicitly listed in "Amendments since v5 initial publication"
  above as removed.
