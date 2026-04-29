# Swirlock Chatbot Manifest

## Status

This document is the current canonical description of the Swirlock chatbot ecosystem.

Its purpose is to let a human or another LLM understand:

- what the system is
- why it is split into multiple apps
- what each app is responsible for
- what each app must not do
- how the apps interact
- what hardware and model constraints shape the design

This manifest describes the intended architecture, even where some parts are not yet implemented.

## Core Vision

The project is a distributed chatbot system built from multiple specialized applications running on different computers in the same local network.

The system is intentionally not designed as a single monolithic chatbot app. Instead, it is decomposed into focused services so that:

- responsibilities stay clear
- each machine has a specific role
- each model has a specific role
- the architecture remains inspectable and explainable
- future commercial evolution remains possible

The chatbot should behave as a coherent intelligence, but internally it is a federation of apps.

## Architectural Philosophy

The project follows these principles:

- Separation of concerns is mandatory.
- A service should own one domain clearly.
- A model is infrastructure, not business logic.
- Retrieval and memory are different concerns.
- The final answering model should not also manage support cognition if that can be avoided.
- The system should be professional and investor-presentable, even if the first version is a proof of concept.
- The system should favor clarity, traceability, and explicit contracts over hidden coupling.

## High-Level System Definition

The Swirlock chatbot ecosystem is a multi-service AI system composed of:

- a Main LLM Server
- a RAG Engine
- a Context Fragmenter
- an Embedding Service
- a future chat-facing application or orchestrator that coordinates user interaction

Together, these services form one chatbot product.

This manifest is centrally maintained in the chatbot contracts repository and applies to the full chatbot ecosystem.

## Hard Hardware Rule

One machine must run no more than one neural model.

This is a strict system rule.

Implications:

- no machine should host multiple LLMs
- no machine should host both a main LLM and an embedding model
- no machine should host two different support models
- if multiple apps need the same model, they must call one local model-serving app on that machine

This rule exists to preserve operational simplicity, predictable resource use, and clean machine roles.

## Physical Topology

### 1. Main Computer

Role:

- host the primary answering model
- expose an API for final inference

Current intended model:

- Gemma 4 (gemma4:e4b) on Ollama

Current hardware assumption:

- NVIDIA 4070 with 16 GB VRAM

This machine should behave as the final reasoning and response generation node.

It should not own retrieval logic.
It should not own memory consolidation logic.
It should not own context-fragment policy.

### 2. Utility Computer

Role:

- host the utility multimodal model
- support the RAG Engine
- support the Context Fragmenter

Current intended model:

- Qwen 3.5 (qwen3.5:9b) multimodal on Ollama

Current hardware assumption:

- NVIDIA 3060 with 12 GB VRAM

This machine is the support cognition node.

It should handle tasks such as:

- query interpretation
- query rewriting
- routing hints
- image understanding
- evidence shaping
- memory classification
- memory consolidation
- background cognitive processing

The Utility Computer may host multiple apps, but only one neural model service.

### 3. Raspberry Pi 5

Role:

- host the embedding model

Current intended model:

- Gemma Embedding or equivalent embedding-only model

This machine is the vectorization node.

It should serve embedding generation for:

- indexing
- retrieval support
- similarity workflows

It should not serve as a general LLM node.

## Logical Services

### Main LLM Server

The Main LLM Server is the final-answer model service.

Its job is narrow by design:

- load the primary model
- expose an inference API
- generate the final answer when given prepared input

It is intentionally not the place for:

- retrieval
- context fragmentation
- memory policy
- search strategy
- web crawling
- long-term memory maintenance

It is the final generator, not the system orchestrator.

### RAG Engine

The RAG Engine is the retrieval and evidence service.

Its purpose is to find, store, reuse, and shape external knowledge.

Its core responsibilities are:

- search the web
- maintain a local database built from search results and retrieved web content
- decide, with help from the utility LLM and deterministic rules, whether to use live web search, the local database, or both
- process multimodal user input when retrieval requires image understanding
- collect evidence
- score and filter evidence
- return a structured evidence package for downstream consumption

The RAG Engine is not responsible for:

- tenant management
- user identity management
- long-term memory policy
- short-term memory policy
- context window packing for the main answering model
- final user-facing answer generation as a product concern

The RAG Engine may generate an evidence-oriented synthesis, but that synthesis is still a retrieval artifact, not the final chatbot answer.

### Context Fragmenter

The Context Fragmenter is the memory management service.

Its purpose is to manage conversational and identity memory in a way that mimics aspects of human memory.

Its core responsibilities are:

- read conversation flow continuously or near-continuously
- interpret what should become memory
- classify memories by type
- classify memories by importance
- separate short-term and long-term memory
- maintain a general user identity memory
- maintain a general app identity memory
- select and recombine memory fragments when the main LLM needs context
- perform nightly or scheduled reorganization during a process called sleep

The Context Fragmenter is not responsible for:

- web retrieval
- general web crawling
- maintaining the web-derived knowledge cache
- embedding infrastructure
- final inference

The Context Fragmenter owns memory semantics.
The RAG Engine owns retrieval semantics.

### Embedding Service

The Embedding Service is a specialized infrastructure service.

Its job is:

- generate embeddings
- support indexing
- support similarity retrieval workflows

It should remain simple and isolated.

It should not contain business logic about memory, search policy, or answer generation.

### Future Chat App Orchestrator

The final chatbot product will likely require a thin coordination layer above the services.

This layer may be implemented as a dedicated app or distributed across existing apps, but conceptually it is the interaction orchestrator.

Its likely responsibilities are:

- accept user input
- call the Context Fragmenter for memory-aware context preparation
- call the RAG Engine for external knowledge retrieval when needed
- prepare the final request to the Main LLM Server
- return the final answer to the user

This orchestrator should remain thin.

It should coordinate services rather than absorb their logic.

## Memory Philosophy

The project uses a custom context philosophy called fragmented context.

This means chatbot context is not treated as a single flat conversation history. Instead, the system tries to organize remembered information into distinct memory classes and importance tiers.

The intended memory categories include:

- short-term memory
- long-term memory
- user identity memory
- app identity memory

Each category may be fragmented further by importance level.

The intent is to imitate human-like memory organization rather than naive transcript accumulation.

## Sleep

Sleep is a scheduled background process in which the Context Fragmenter reorganizes memory.

During sleep, the system may:

- reevaluate memory importance
- promote or demote memories
- merge overlapping memories
- compress redundant memories
- redistribute memory fragments across importance levels
- refine user identity memory
- refine app identity memory

Sleep is not a cosmetic feature.

It is a core architectural idea of the project.

## Distinction Between Memory And Knowledge

The system must preserve the difference between memory and retrieved knowledge.

Memory means:

- what the chatbot knows about the user
- what the chatbot knows about itself
- what the chatbot remembers from past interaction

Retrieved knowledge means:

- information gathered from the web
- information cached locally from prior web retrieval
- evidence collected to answer a query grounded in external sources

These two domains may interact, but they must not be collapsed into one storage or one mental model.

## Why The Apps Are Split

The apps are split because each part of the chatbot has a distinct epistemic role.

The Main LLM answers.
The RAG Engine looks outward.
The Context Fragmenter looks inward.
The Embedding Service converts content into vector form.

This split improves:

- clarity
- maintainability
- debuggability
- replaceability of components
- future commercialization

## Service Boundaries

The following boundaries are part of the intended canon.

### Main LLM Server Boundary

The Main LLM Server should receive prepared input and produce final completions.

It should not need to know how web retrieval worked internally.
It should not need to know how memory sleep was performed internally.

### RAG Engine Boundary

The RAG Engine should receive a retrieval brief, not full system internals.

It may need:

- the user question
- resolved references or rewritten search intent
- freshness expectations
- relevant memory hints if they affect retrieval
- modality information such as text or image input

It should not need:

- full memory graphs
- full tenant policy
- full long-term identity internals
- full context-window assembly logic

### Context Fragmenter Boundary

The Context Fragmenter should receive conversation and system information needed for memory work.

It may use the utility LLM for:

- salience detection
- memory extraction
- memory compression
- importance scoring
- nightly consolidation

It should not own retrieval infrastructure.

## Shared Utility Model Principle

Both the RAG Engine and the Context Fragmenter may depend on the same utility LLM hosted on the Utility Computer.

This is acceptable because:

- they remain separate applications
- the shared model is infrastructure
- the first version is a low-scale proof of concept

However, this creates an operational requirement:

- live user-path requests must have priority over background jobs

If the utility model is busy, the system must avoid blocking the full chatbot unnecessarily.

## Retrieval Philosophy

The RAG Engine should behave as a web-backed evidence system with a persistent local knowledge cache.

Its local database is not primarily a memory store.

It is a knowledge store derived from:

- prior searches
- prior fetched pages
- normalized extracted content
- indexed retrieval artifacts
- metadata about freshness and provenance

The purpose of this database is to reduce repeated web work, improve retrieval quality, and accumulate useful external knowledge over time.

## Intended Data Flow

The intended high-level flow for a user interaction is:

1. A user sends a message to the chatbot.
2. The interaction layer sends conversation material to the Context Fragmenter.
3. The Context Fragmenter selects or prepares relevant memory fragments.
4. If external knowledge is needed, a retrieval brief is sent to the RAG Engine.
5. The RAG Engine chooses local retrieval, live web retrieval, or both.
6. The RAG Engine returns structured evidence.
7. The interaction layer prepares a final request using memory fragments and retrieved evidence.
8. The Main LLM Server generates the final answer.
9. The resulting conversation is later re-read by the Context Fragmenter for memory updates.

## Multimodal Role

The system is intended to support multimodal input.

At minimum, this means the utility model may inspect images and convert them into useful semantic observations for:

- retrieval
- memory extraction
- downstream reasoning support

Multimodality belongs primarily to the utility side of the architecture, not the main answering side.

## Professional Quality Requirement

Even as a proof of concept, the system must be engineered to commercial standards.

That means:

- explicit contracts
- modular code
- testable boundaries
- structured logging
- reproducible behavior where possible
- failure handling
- clear ownership of responsibilities

The system is not allowed to be a casual prototype with entangled logic.

## Non-Goals Of The Current Scope

The current scope does not require:

- large-scale multi-tenant SaaS features inside the RAG Engine
- a monolithic all-in-one chatbot service
- production-scale load handling from day one
- collapsing memory and retrieval into one store
- using the Main LLM Server for all cognitive support tasks

## Canonical Naming

The following names are canonical unless intentionally revised later:

- Swirlock Chatbot Ecosystem
- Main LLM Server
- RAG Engine
- Context Fragmenter
- Embedding Service
- Utility LLM
- Primary LLM
- fragmented context
- sleep

## Final Canonical Summary

This project is a distributed chatbot architecture in which:

- one machine hosts the main answering model
- one machine hosts a utility multimodal model
- one machine hosts an embedding model
- the chatbot is split into specialized services
- retrieval and memory are separate domains
- memory is organized as fragmented context
- memory is periodically reorganized during sleep
- the RAG Engine manages external knowledge retrieval and a local web-derived knowledge base
- the Context Fragmenter manages conversational and identity memory
- the Main LLM Server performs final answer generation

This separation is intentional and foundational.

Any future design, implementation, or contract should remain consistent with this manifest unless the architecture is explicitly changed.
