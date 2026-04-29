# Interaction Model v2

## Purpose

This document defines the canonical interaction model for the Swirlock chatbot ecosystem in `v2`.

The core change from `v1` is that LLM-serving apps are model hosts. They are infrastructure, not domain services.

## Canonical Service Graph

```text
Client
  |
  v
Chat Orchestrator
  |------> Context Fragmenter
  |------> RAG Engine
  |------> Primary LLM Host
  |
  v
Client response

RAG Engine ------------\
                       v
Context Fragmenter ---> Utility LLM Host

Scheduler/Admin
  |
  v
Context Fragmenter
```

Model host calls do not change ownership of RAG, memory, or chat semantics. They are infrastructure calls.

## Core Services

### Chat Orchestrator

Owns:

- user-facing session handling
- per-turn coordination
- deciding when to call Context Fragmenter
- deciding when to call RAG Engine
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
- text-only inference
- model health and lifecycle protection
- model health/status

Does not own:

- final-answer semantics
- prompt construction policy
- chat sessions
- memory selection
- retrieval selection

The Chat Orchestrator commonly uses this host for final response inference, but the host itself remains purpose-agnostic.

### Utility LLM Host

Owns:

- utility multimodal model availability
- text and image inference
- model health and lifecycle protection
- model health/status

Does not own:

- RAG semantics
- memory semantics
- image storage
- chat semantics
- task interpretation

Any internal app may call this host when it needs the utility model, subject to priority and capacity rules.

## Allowed Direct Calls

Allowed:

- `Client -> Chat Orchestrator`
- `Chat Orchestrator -> Context Fragmenter`
- `Chat Orchestrator -> RAG Engine`
- `Chat Orchestrator -> Primary LLM Host`
- `RAG Engine -> Utility LLM Host`
- `Context Fragmenter -> Utility LLM Host`
- `Scheduler/Admin -> Context Fragmenter`
- internal service calls to model hosts, when the caller owns the task semantics

## Forbidden Direct Calls

Forbidden unless the architecture is intentionally revised:

- `RAG Engine -> Context Fragmenter`
- `Context Fragmenter -> RAG Engine`
- `Primary LLM Host -> RAG Engine`
- `Primary LLM Host -> Context Fragmenter`
- `Utility LLM Host -> RAG Engine`
- `Utility LLM Host -> Context Fragmenter`
- `Client -> Context Fragmenter`
- `Client -> RAG Engine`
- `Client -> model hosts`

## Turn Lifecycle

1. Client submits a user message to the Chat Orchestrator.
2. Chat Orchestrator requests memory context from Context Fragmenter.
3. If the user input includes images, Chat Orchestrator or a downstream service may call Utility LLM Host for image observations.
4. Chat Orchestrator decides whether retrieval is needed and calls RAG Engine when needed.
5. RAG Engine may call Utility LLM Host for query/evidence support.
6. Chat Orchestrator prepares final model input.
7. Chat Orchestrator calls Primary LLM Host.
8. Chat Orchestrator returns the final response to the client.
9. Chat Orchestrator records the completed turn with Context Fragmenter.

## Background Lifecycle

Context Fragmenter may call Utility LLM Host during background memory processing or sleep. Those
calls should either omit model-host priority or use a low numeric priority so live user-path work
can submit a higher numeric priority and move ahead in the model-host queue.
