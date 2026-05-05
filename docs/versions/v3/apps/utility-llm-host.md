# Utility LLM Host

> Repository: `swirlock-llm-host` (shared with the Primary LLM Host)
> Contract: [`openapi/model-host.openapi.yaml`](../openapi/model-host.openapi.yaml)
> Local port (current convention): a Model Host port distinct from the
> Primary LLM Host's, per `INTERNAL_INFRASTRUCTURE.md#local-port-conventions`.

## Role

The Utility LLM Host owns the health and serving boundary for the utility
multimodal model. It is support capacity for retrieval support, memory
support, image-derived retrieval support, background work, and explicitly
allowed degraded final-answer fallback. Any internal app may call this host
when it needs the utility model, subject to priority and capacity rules.

It is not the unique vision node. It does not own RAG semantics, memory
semantics, image storage, chat semantics, or task interpretation.

## Final Form

- Same Model Host API surface as the Primary LLM Host (`/v2/infer/stream`,
  `/v2/health`, `/v2/model/status`,
  `/v2/model/preload`, `/v2/model/unload`).
- Hosted model: `gemma4:e4b` on Ollama, with text and image input enabled.
  Capability symmetry with the Primary LLM Host does not collapse
  ownership: the two hosts are operationally distinct and used for
  different roles (final-answer vs. support).
- Runs on the Utility Computer, dedicated to support cognition.
- May co-host the Embedding Service as a CPU-only neural model service,
  subject to the Hard Hardware Rule and the CPU-only co-location rules in
  `INTERNAL_INFRASTRUCTURE.md#cpu-only-co-located-neural-services`.
- Live user-path callers should provide higher numeric model-host priority
  than background memory or sleep jobs.

## Roadmap

Codebase finished. No roadmap items pending for the Utility role itself.

## Current Implementation

The `swirlock-llm-host` codebase implements the Model Host API and is
considered finished. The Utility LLM Host deployment is the only Model
Host currently running in the local network. While the Primary LLM Host
deployment on the Main Computer is not yet provisioned, the Chat
Orchestrator points its final-answer slot at this same Utility LLM Host
as a temporary stand-in for Primary; this is a Chat Orchestrator
configuration choice, not a Utility LLM Host concern. The Utility host
itself remains agnostic — it serves whatever inference requests reach it,
and does not know whether the caller is performing chat, retrieval support,
memory support, or final answering.

The Primary and Utility LLM Hosts are designed as operationally distinct
deployments built from the same codebase, with distinct ports per
`INTERNAL_INFRASTRUCTURE.md#local-port-conventions`. Once the Main Computer
deployment exists, the Chat Orchestrator's final-answer traffic moves to
that Primary deployment and the Utility LLM Host returns to handling only
support-cognition traffic for RAG Engine and Context Fragmenter.
