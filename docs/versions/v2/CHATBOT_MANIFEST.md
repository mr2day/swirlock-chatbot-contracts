# Swirlock Chatbot Manifest v2

## Status

This document is the canonical architectural statement for Swirlock contracts `v2`.

Version `v2` keeps the distributed-service architecture from `v1` and clarifies that LLM servers are agnostic model hosts. They protect and expose models. They do not own product semantics.

## Core Vision

Swirlock is a distributed chatbot system built from specialized services running on machines in the same local network.

The system is intentionally not a monolith. It separates:

- orchestration
- retrieval
- memory
- model hosting
- embedding infrastructure

This separation keeps responsibilities inspectable, replaceable, and commercially presentable.

## Architectural Principles

- Separation of concerns is mandatory.
- A model is infrastructure, not business logic.
- Model hosts are agnostic inference appliances.
- Retrieval and memory are different concerns.
- The final answering workflow belongs to the Orchestrator, not the model host.
- Support cognition should use the Utility LLM Host instead of overloading the primary model.
- Service contracts should be explicit and traceable.

## Hard Hardware Rule

One machine must run no more than one neural model.

Implications:

- no machine should host multiple LLMs
- no machine should host both an LLM and an embedding model
- if multiple apps need the same model, they call the local model host on that machine

## Physical Topology

### Main Computer

Role:

- host the Primary LLM Host
- expose primary model inference

Current intended model:

- Gemma 4 (`gemma4:e4b`) on Ollama, with vision enabled when available

This machine should not own retrieval, memory, or orchestration logic.

### Utility Computer

Role:

- host the Utility LLM Host
- expose text-plus-image model inference
- support RAG Engine, Context Fragmenter, and other internal apps

Current intended model:

- Qwen 3.5 (`qwen3.5:9b`) multimodal on Ollama

This machine may host multiple apps, but only one neural model service.

### Raspberry Pi 5

Role:

- host the Embedding Service

Current intended model:

- Gemma Embedding or equivalent embedding-only model

## Logical Services

### Chat Orchestrator

The Chat Orchestrator owns the user-turn lifecycle.

It coordinates memory, retrieval, image pre-processing, final prompt preparation, model-host calls, and final client responses.

### RAG Engine

The RAG Engine owns external knowledge retrieval and evidence packaging.

It may use the Utility LLM Host for query interpretation, query rewriting, image observations, and evidence shaping.

### Context Fragmenter

The Context Fragmenter owns conversational and identity memory.

It may use the Utility LLM Host for salience detection, memory extraction, memory compression, classification, and sleep consolidation.

### Primary LLM Host

The Primary LLM Host owns the health and serving boundary for the primary model.

It accepts the input types advertised by its model-host capabilities and returns text output.

It does not know whether the caller is performing chat, final answering, summarization, classification, or another task.

### Utility LLM Host

The Utility LLM Host owns the health and serving boundary for the utility multimodal model.

It accepts text and optional image input and returns text output.

It does not know whether the caller is performing RAG support, memory support, image understanding, chat, or another task.

### Embedding Service

The Embedding Service owns vectorization.

It should remain isolated from chat, retrieval policy, memory policy, and general LLM inference.

## Model Host Protection

Model hosts must protect model availability through:

- keep-alive/preload behavior
- unload lifecycle behavior
- one active model request at a time unless a host explicitly documents more slots
- queue reporting for additional requests
- process-level request body protection
- health/status reporting

## Shared Utility Model Principle

RAG Engine and Context Fragmenter may both depend on the Utility LLM Host.

This is acceptable because:

- the host is infrastructure
- caller services own semantics
- the first version is a low-scale local network system

Live user-path calls must be able to move ahead of background jobs through model-host numeric
priority. Omitted priority is the lowest model-host priority.

## Multimodal Role

Multimodality belongs primarily to the utility side.

Images may be inspected by the Utility LLM Host and converted into text observations when
downstream services need text-only inputs. If the Primary LLM Host advertises `imageInput: true`,
the Orchestrator may also route image input directly to it when that is the intended workflow.

Model-host capabilities, not host names, define whether image input is accepted.

## Canonical Naming

Canonical names in `v2`:

- Swirlock Chatbot Ecosystem
- Chat Orchestrator
- RAG Engine
- Context Fragmenter
- Primary LLM Host
- Utility LLM Host
- Embedding Service
- Model Host API
- fragmented context
- sleep

## Final Summary

Swirlock `v2` is a distributed chatbot architecture in which:

- domain services own product semantics
- model hosts own model health and inference only
- the Primary LLM Host exposes the capabilities of its hosted model
- the Utility LLM Host is multimodal input and text output
- the Orchestrator coordinates final answer generation
- RAG and memory remain separate domains
