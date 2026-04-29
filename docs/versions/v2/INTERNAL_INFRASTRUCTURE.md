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
- model safety limits
- concurrency protection
- timeout behavior

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
- text output
- no image input
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

Every model host should enforce:

- maximum text input size
- maximum image count
- maximum image byte size
- maximum output tokens
- maximum concurrent requests
- request timeout
- safe generation option whitelist
- keep-alive behavior

Caller options are not authoritative. The host may clamp or reject them.

## Health Requirements

Every model host should expose:

- liveness
- readiness
- configured model ID
- loaded/running state
- capabilities
- configured limits
- active request count
- queue depth if queueing exists

## Media Handling

Domain services exchange media through `imageId` or `imageUrl`.

Model hosts may accept:

- `imageUrl`
- direct base64 image payloads

Model hosts should not become durable media stores. If a caller has an `imageId`, the caller should resolve it to bytes or URL before model inference unless the deployment explicitly provides a shared media resolver.
