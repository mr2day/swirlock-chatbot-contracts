# Embedding Service

> Repository: `swirlock-embedding-service`
> Contract: [`openapi/embedding-service.openapi.yaml`](../openapi/embedding-service.openapi.yaml)
> Local port (current convention): `3002`

## Role

The Embedding Service owns vectorization. It exposes the configured
embedding model's advertised input capabilities and reports model health
and lifecycle. It is infrastructure, not a domain service: RAG Engine and
Context Fragmenter call it directly when they need vectors.

It does not own retrieval semantics, memory semantics, similarity search,
vector persistence, or task interpretation. The Embedding Service does not
know whether the caller is indexing web content, embedding a live retrieval
query, or vectorizing a memory fragment. Callers own persistence,
similarity scoring, and any task-level interpretation of the returned
vectors.

## Final Form

- Full Embedding Service API: persistent WebSocket vectorization, model health,
  and lifecycle controls in the same agnostic-appliance pattern as the Model
  Host API.
- Single source of truth for model availability, capabilities, and health.
- CPU-only neural service when co-located with an accelerator-bound
  service. The current canonical example is co-location on the Utility
  Computer alongside the Utility LLM Host's GPU-resident model, subject
  to the Hard Hardware Rule and the CPU-only co-location rules in
  `INTERNAL_INFRASTRUCTURE.md#cpu-only-co-located-neural-services`.
- May be deployed on a dedicated low-power node (e.g. Raspberry Pi 5)
  when a future deployment prefers a separate physical box for
  vectorization. Same contract surface either way.
- Background and indexing workloads must use omitted or low numeric
  `requestContext.priority` so live user-path work moves ahead in the
  service's queue.

## Roadmap

The detailed roadmap has not yet been drafted in this contract version.
Co-location and CPU-only rules are pinned in
`INTERNAL_INFRASTRUCTURE.md#cpu-only-co-located-neural-services`.

## Current Implementation

Implementation status not surveyed for this v3 draft. The contract surface
is defined in `openapi/embedding-service.openapi.yaml` and the co-location
rules are pinned in
`INTERNAL_INFRASTRUCTURE.md#cpu-only-co-located-neural-services`. This
section will be filled in when the service is audited against running
code.
