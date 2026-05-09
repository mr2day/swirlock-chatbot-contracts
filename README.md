# Swirlock Chatbot Contracts

This repository is the canonical contract source for the Swirlock chatbot ecosystem.

It defines how the core chatbot services talk to each other:

- Chat Orchestrator
- Context Fragmenter
- RAG Engine
- Main LLM Server

## Purpose

This repository exists so that service implementations do not invent their own APIs independently.

Its job is to hold:

- the interaction model
- cross-service API conventions
- machine-readable OpenAPI specifications

It does not exist to hold:

- service implementation code
- model prompts
- internal storage schemas that do not cross service boundaries
- machine-specific deployment scripts

## Canonical Sources

The legacy draft entry point is:

1. `docs/CHATBOT_MANIFEST.md`
2. `docs/openapi/*.openapi.yaml`
3. `docs/INTERACTION_MODEL.md`
4. `docs/API_CONVENTIONS.md`

If there is a conflict:

- `docs/CHATBOT_MANIFEST.md` wins for system vision, philosophy, and service boundaries.
- OpenAPI files win for exact field shapes and endpoint details.
- Markdown files win for architectural intent and boundary interpretation.

## Versioning

Versioned contracts live under `docs/versions/`.

- `docs/versions/v1/` preserves the original draft contracts unchanged.
- `docs/versions/v2/` contains the revised draft that introduces agnostic Model Host APIs for Primary LLM Host and Utility LLM Host.
- `docs/versions/v3/` splits app-specific docs under `apps/`.
- `docs/versions/v4/` introduces the breaking transport contract: no ecosystem REST APIs, one persistent WebSocket per app relationship, and one shared envelope.
- `docs/versions/v5/` is the current architectural target. It removes the v4 Primary/Utility two-LLM split (operationally incoherent in a sequential turn pipeline), introduces the 1:1 module-to-LLM rule, names the live-conversation LLM the **Vanamonde LLM** (after Arthur C. Clarke's *The City and the Stars*) and the background-consolidation LLM the **Fragmenter LLM**, and promotes the Context Fragmenter to a peer module that shares a SQLite file with the Chat Orchestrator under table-level ownership.

See `docs/VERSIONING.md` for compatibility and promotion rules.

## Current Scope

The top-level `docs/` files currently remain the draft `v1` entry point for:

- client-facing chat orchestration
- memory preparation and memory recording
- retrieval and evidence packaging
- final LLM inference

The Utility LLM and Embedding Service are intentionally not first-class API contracts in this first pass.
They are internal infrastructure behind other services.

The current implementation target is `v5`. `v4` remains supported and the
two versions can coexist in a running deployment during migration.

## Repository Structure

```text
docs/
  VERSIONING.md
  CHATBOT_MANIFEST.md
  INTERACTION_MODEL.md
  API_CONVENTIONS.md
  openapi/
    chat-orchestrator.openapi.yaml
    context-fragmenter.openapi.yaml
    rag-engine.openapi.yaml
    main-llm-server.openapi.yaml
  versions/
    v1/
    v2/
    v3/
    v4/
    v5/
```

## Working Rule

This repository is contract-first.

Service repos should implement these contracts.
They should not become the canonical owners of cross-service API definitions.
