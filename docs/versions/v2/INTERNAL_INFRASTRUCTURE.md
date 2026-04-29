# Internal Infrastructure v2

## Purpose

This document defines guidance for infrastructure services that support the Swirlock ecosystem but do not own product semantics.

## Runtime Configuration Source Of Truth

Every deployable service must have exactly one obvious source of truth for runtime configuration.

Runtime configuration means deploy-time or machine-specific values such as:

- host and port
- upstream service URLs
- model IDs and model allow-lists
- model keep-alive settings
- feature flags such as image input or thinking defaults
- request body size limits
- process-manager environment values

These values must not be duplicated across `.env`, `.env.example`, source-code defaults,
process-manager config, README examples, and ad hoc launch commands.

For Node services, the preferred local pattern is:

- put a committed root config file in the service repo, such as `host.config.cjs` for a model host
  or `service.config.cjs` for a domain service
- export one object containing the runtime values, commonly named `env`
- import that same object from the app bootstrap/config loader
- import that same object from the process-manager config, such as `ecosystem.config.cjs`
- make README setup instructions point to that file as the place to edit runtime settings
- fail fast during startup if required config values are missing or internally inconsistent

Service implementations should avoid source-code fallback values for deploy-time settings. A missing
model ID, upstream URL, host, port, or equivalent runtime setting should be treated as a
configuration error, not silently replaced with a hidden default.

Secrets may still come from a private secret store or an uncommitted local file when needed, but the
service must still have one named config-loading path. Secret loading must not reintroduce multiple
competing places for the same parameter.

The goal is operational clarity: changing a model, port, upstream URL, or equivalent runtime setting
must require one edit in one predictable place.

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
