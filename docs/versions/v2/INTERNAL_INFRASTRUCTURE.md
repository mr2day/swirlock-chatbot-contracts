# Internal Infrastructure v2

## Purpose

This document defines guidance for infrastructure services that support the Swirlock ecosystem but do not own product semantics.

## Model Host Principle

A model host is an agnostic model appliance.

It owns:

- model loading
- model keep-alive
- model unloading
- inference transport
- health/readiness reporting
- model-health protection
- model-slot serialization and queue reporting

It does not own:

- chat semantics
- RAG semantics
- memory semantics
- image storage semantics
- business task interpretation
- prompt policy beyond local safety/system guardrails

## Model Host Profiles

### Primary LLM Host

Profile:

- text input
- optional image input when the hosted model supports vision
- text output
- no image output

Typical caller:

- Chat Orchestrator

Common usage:

- final response inference from prepared input

### Utility LLM Host

Profile:

- text input
- image input
- text output
- no image output

Typical callers:

- RAG Engine
- Context Fragmenter
- Chat Orchestrator when image understanding is needed before final inference

Common usage:

- image observations
- query rewriting
- classification
- compression
- evidence shaping
- memory support

The Utility LLM Host does not know which of these tasks it is serving. The caller owns the prompt and interpretation.

## Safety Requirements

Every model host should protect the hosted model and local machine through:

- keep-alive behavior
- preload/unload lifecycle behavior
- one active model request at a time unless the host explicitly documents more slots
- a visible queue for additional requests
- process-level request body protection

Model hosts should not impose task-level limits such as output token caps or elapsed-time caps.
Callers own task policy and may pass model-runtime options when the host contract allows it.

## Queue Requirements

Model hosts with a single model slot should queue additional requests. Queue order is:

1. higher numeric `requestContext.priority` first
2. earlier arrival first when priorities are equal

If `requestContext.priority` is omitted, the request is treated as lower priority than every request
that provides a finite priority number.

Streaming model-host APIs should report queue position, queue depth, requests ahead, and estimated
start time when enough recent runtime samples exist. Queue estimates are advisory.

## Health Requirements

Every model host should expose:

- liveness
- readiness
- configured model ID
- loaded/running state
- capabilities
- active request count
- model slot count
- queue depth
- average request duration when available

## Media Handling

Domain services exchange media through `imageId` or `imageUrl`.

Model hosts may accept:

- `imageUrl`
- direct base64 image payloads

Model hosts should not become durable media stores. If a caller has an `imageId`, the caller should resolve it to bytes or URL before model inference unless the deployment explicitly provides a shared media resolver.
