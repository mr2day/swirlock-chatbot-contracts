# Interaction Model v3

## Purpose

This document defines the canonical interaction model for the Swirlock chatbot ecosystem in `v3`.

`v3` keeps the interaction model from `v2` unchanged. It only restructures the contract documentation by introducing per-app docs under `apps/`. The semantic boundaries, allowed direct calls, and routing rules below are identical to those in `v2`.

## Canonical Service Graph

```text
Client
  |
  v
Chat Orchestrator
  |------> Context Fragmenter --------+----> Utility LLM Host
  |                                   \----> Embedding Service
  |------> RAG Engine ----------------+----> Utility LLM Host
  |                                   \----> Embedding Service
  |------> Primary LLM Host
  |------> Utility LLM Host (support/fallback only)
  |
  v
Client response

Scheduler/Admin
  |
  v
Context Fragmenter
```

Model host calls and Embedding Service calls do not change ownership of RAG, memory, or chat semantics. They are infrastructure calls.

## Core Services

### Chat Orchestrator

Owns:

- user-facing session handling
- per-turn coordination
- deciding when to call Context Fragmenter
- deciding when to call RAG Engine
- deciding whether original user images should be passed to the final-answer model host
- preparing prompts or messages for model hosts
- returning final responses to clients

Does not own:

- retrieval internals
- memory internals
- model hosting

### RAG Engine

Owns:

- retrieval from local knowledge and live web
- evidence collection
- evidence scoring
- evidence packaging
- retrieval-oriented synthesis

May call:

- Utility LLM Host for query interpretation, rewriting, image observations, and evidence shaping

Does not own:

- memory semantics
- model hosting
- final user-facing answer ownership

### Context Fragmenter

Owns:

- memory extraction
- memory classification
- memory selection
- memory consolidation
- sleep jobs

May call:

- Utility LLM Host for salience detection, memory extraction, compression, classification, and consolidation support

Does not own:

- web retrieval
- model hosting
- final answer ownership

### Primary LLM Host

Owns:

- primary model availability
- inference for the primary model's advertised input capabilities, including image input when advertised
- model health and lifecycle protection
- model health/status

Does not own:

- final-answer semantics
- prompt construction policy
- chat sessions
- memory selection
- retrieval selection

The Chat Orchestrator normally uses this host for final response inference. When the hosted model advertises `imageInput: true`, the Orchestrator may pass original user images directly to it for final answering. The host itself remains purpose-agnostic.

### Utility LLM Host

Owns:

- utility model availability
- inference for the utility model's advertised input capabilities, including image input when advertised
- model health and lifecycle protection
- model health/status

Does not own:

- RAG semantics
- memory semantics
- image storage
- chat semantics
- task interpretation

Any internal app may call this host when it needs the utility model, subject to priority and capacity rules.

The Utility LLM Host is not the only vision-capable service. It is support capacity for internal cognition, not the owner of image semantics.

### Embedding Service

Owns:

- embedding model availability
- vectorization for the configured embedding model's advertised input capabilities
- model health and lifecycle protection
- model health/status

Does not own:

- retrieval semantics
- memory semantics
- similarity search
- vector persistence
- task interpretation

The Embedding Service is infrastructure, not a domain service. RAG Engine and Context Fragmenter may call it directly when they need vectors. The Embedding Service does not know whether the caller is indexing web content, embedding a live retrieval query, or vectorizing a memory fragment. Callers own persistence, similarity scoring, and any task-level interpretation of the returned vectors.

## Allowed Direct Calls

Allowed:

- `Client -> Chat Orchestrator`
- `Chat Orchestrator -> Context Fragmenter`
- `Chat Orchestrator -> RAG Engine`
- `Chat Orchestrator -> Primary LLM Host`
- `Chat Orchestrator -> Utility LLM Host` for support cognition or explicitly allowed degraded final-answer fallback
- `RAG Engine -> Utility LLM Host`
- `RAG Engine -> Embedding Service`
- `Context Fragmenter -> Utility LLM Host`
- `Context Fragmenter -> Embedding Service`
- `Scheduler/Admin -> Context Fragmenter`
- internal service calls to model hosts and the Embedding Service, when the caller owns the task semantics

## Forbidden Direct Calls

Forbidden unless the architecture is intentionally revised:

- `RAG Engine -> Context Fragmenter`
- `Context Fragmenter -> RAG Engine`
- `Primary LLM Host -> RAG Engine`
- `Primary LLM Host -> Context Fragmenter`
- `Primary LLM Host -> Embedding Service`
- `Utility LLM Host -> RAG Engine`
- `Utility LLM Host -> Context Fragmenter`
- `Utility LLM Host -> Embedding Service`
- `Embedding Service -> RAG Engine`
- `Embedding Service -> Context Fragmenter`
- `Embedding Service -> Utility LLM Host`
- `Embedding Service -> Primary LLM Host`
- `Client -> Context Fragmenter`
- `Client -> RAG Engine`
- `Client -> model hosts`
- `Client -> Embedding Service`

## Canonical User Message Route

This route defines how a user message moves through the system. Agents developing any ecosystem app should preserve this ownership model unless the contracts are intentionally revised.

1. Client submits a user message to the Chat Orchestrator. The message may contain text, images, attachments, and client-side metadata.
2. Chat Orchestrator validates the session and turn, assigns or propagates a correlation ID, and normalizes media references without interpreting the task semantics inside a model host.
3. Chat Orchestrator requests relevant memory context and retrieval hints from Context Fragmenter. Context Fragmenter may call Utility LLM Host for salience, classification, or memory-support inference, but Context Fragmenter owns all memory decisions.
4. Chat Orchestrator decides whether retrieval is needed. If retrieval is needed, it calls RAG Engine with the normalized user input, memory hints, and media references relevant to retrieval.
5. RAG Engine performs retrieval and evidence packaging. It may call Utility LLM Host for query rewriting, image-derived retrieval intent, OCR-like observations, or evidence shaping, but RAG Engine owns all retrieval decisions.
6. Chat Orchestrator assembles the final model input from the user message, selected memory, evidence package, instructions, and media that should be visible to the final-answer model.
7. Chat Orchestrator calls Primary LLM Host for final response inference. If the user message includes an image and Primary LLM Host advertises `imageInput: true`, the Orchestrator should pass the original image directly to Primary LLM Host when the final answer benefits from visual understanding.
8. Chat Orchestrator returns the final assistant response to the client, including citations, diagnostics, or degraded-mode indicators when the caller contract requires them.
9. Chat Orchestrator records the completed turn with Context Fragmenter. Context Fragmenter may queue background memory extraction, compression, or sleep work that uses Utility LLM Host at low or omitted model-host priority.

## Routing Rules

- Route by responsibility, not by capability. Two model hosts may expose the same model and the same multimodal capabilities while serving different operational roles.
- Use Primary LLM Host for normal final user-facing answer generation.
- Use Utility LLM Host for support cognition: retrieval support, memory support, image-derived search intent, classification, compression, evidence shaping, and background jobs.
- Do not require an image to pass through Utility LLM Host before final answering when Primary LLM Host can accept the original image.
- Do not let model hosts decide product semantics. The caller owns prompt construction, task policy, and interpretation of model output.
- Degraded final-answer fallback to Utility LLM Host is allowed only when the Orchestrator explicitly supports that mode and exposes it in diagnostics or operational logs.
- Live user-path calls should provide higher numeric model-host priority than background memory or sleep jobs.

## Streaming Variants

A service may expose a WebSocket sibling for an otherwise blocking operation when callers need to
render intermediate output, such as model thinking text, response token chunks, or queue position
updates. The streaming endpoint sits beside the blocking one and uses an event shape modeled on
the Model Host stream events: `accepted`, `queued`, `started`, `thinking`, `chunk`, `done`,
`error`.

The Chat Orchestrator's streaming variant of `submitChatTurn` preserves the same ownership
boundaries as the blocking turn submission: the Chat Orchestrator still owns session lifecycle,
turn persistence, retrieval orchestration, memory recording, and final response shape. Streaming
is a transport variation, not a different service responsibility. The orchestrator's terminal
`done` event therefore carries the persisted identifiers (`turnId`,
`assistantMessage.messageId`, `createdAt`) in addition to `finishReason`, because the orchestrator
is what writes them.

Bearer authentication on the WebSocket upgrade follows
`API_CONVENTIONS.md#websocket-authentication`. Authorization failures discovered after a
successful upgrade, such as a session that exists but belongs to another user, are reported as
the service-defined `error` event before the server closes the connection.

## Background Lifecycle

Context Fragmenter may call Utility LLM Host during background memory processing or sleep. Those
calls should either omit model-host priority or use a low numeric priority so live user-path work
can submit a higher numeric priority and move ahead in the model-host queue.
