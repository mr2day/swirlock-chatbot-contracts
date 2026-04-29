# Interaction Model

## Purpose

This document defines the canonical interaction model between the core services of the Swirlock chatbot ecosystem.

It answers:

- who may call whom
- which responsibilities belong to which service
- where coupling is allowed
- where coupling is forbidden

## Canonical Service Graph

The canonical call graph for `v1` is:

```text
Client
  |
  v
Chat Orchestrator
  |------> Context Fragmenter
  |------> RAG Engine
  |------> Main LLM Server
  |
  v
Client response

Scheduler/Admin
  |
  v
Context Fragmenter
```

## Rule Of The Graph

The Chat Orchestrator is the hub.

It is the service that coordinates the user-turn lifecycle.
It is not the place where retrieval logic, memory logic, or model-hosting logic should live.

## Core Service Responsibilities

### Chat Orchestrator

Owns:

- user-facing chat session handling
- per-turn coordination
- calling Context Fragmenter
- calling RAG Engine when external knowledge is needed
- calling Main LLM Server with prepared input
- returning the final answer to the client

Does not own:

- memory policy
- retrieval policy internals
- model hosting

### Context Fragmenter

Owns:

- memory extraction
- memory categorization
- memory selection for a turn
- memory consolidation
- sleep jobs

Does not own:

- web search
- evidence retrieval
- final answer generation

### RAG Engine

Owns:

- retrieval from local knowledge and live web
- evidence collection
- evidence scoring
- evidence packaging
- local web-derived knowledge accumulation

Does not own:

- memory semantics
- context packing policy
- final answer generation

### Main LLM Server

Owns:

- primary model inference
- final response generation from prepared input

Does not own:

- memory selection
- retrieval selection
- cross-service orchestration

## Allowed Direct Calls

Allowed:

- `Client -> Chat Orchestrator`
- `Chat Orchestrator -> Context Fragmenter`
- `Chat Orchestrator -> RAG Engine`
- `Chat Orchestrator -> Main LLM Server`
- `Scheduler/Admin -> Context Fragmenter`

## Forbidden Direct Calls In v1

Forbidden unless the architecture is intentionally revised:

- `RAG Engine -> Context Fragmenter`
- `Context Fragmenter -> RAG Engine`
- `Main LLM Server -> RAG Engine`
- `Main LLM Server -> Context Fragmenter`
- `Client -> Context Fragmenter`
- `Client -> RAG Engine`
- `Client -> Main LLM Server`

The goal is to avoid hidden coupling and preserve one clear coordination point.

## Turn Lifecycle

The canonical synchronous user-turn lifecycle is:

1. The client submits a user message to the Chat Orchestrator.
2. The Chat Orchestrator requests context preparation from the Context Fragmenter.
3. The Chat Orchestrator decides whether retrieval is needed and, if so, requests evidence from the RAG Engine.
4. The Chat Orchestrator assembles the final input for the Main LLM Server.
5. The Main LLM Server returns the assistant response.
6. The Chat Orchestrator returns the assistant response to the client.
7. The Chat Orchestrator records the completed turn with the Context Fragmenter.

## Background Sleep Lifecycle

The canonical memory consolidation lifecycle is:

1. A scheduler or admin process triggers a sleep job on the Context Fragmenter.
2. The Context Fragmenter performs memory reorganization internally.
3. The Context Fragmenter exposes job status for monitoring.

The Chat Orchestrator is not required to participate in sleep execution.

## Internal Infrastructure Out Of Scope

The following are important to the overall ecosystem, but are not defined as first-class contracts in this repository yet:

- the Utility LLM serving interface
- the Embedding Service interface
- internal storage layouts
- internal job queues

Those are implementation details behind current service boundaries.

## Why The Hub Model Is Preferred

The hub model is preferred because it keeps retrieval and memory services independent.

That gives the architecture:

- fewer hidden dependencies
- easier local reasoning
- clearer debugging
- cleaner future replacement of one service without rewriting the others

If a future version introduces direct service-to-service calls outside the hub, that should be treated as an explicit architectural change, not an accidental convenience.
