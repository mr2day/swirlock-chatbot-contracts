# Swirlock Chatbot Manifest v3

## Status

This document is the canonical architectural statement for Swirlock contracts `v3`.

Version `v3` keeps the architectural stance and semantic surface of `v2` (LLM-serving apps are agnostic model hosts; orchestration, retrieval, and memory remain separate domains). It introduces a per-app contract document format under `apps/` so every Swirlock app has one canonical document combining its **Final Form**, **Roadmap**, and **Current Implementation** state. The URL versioning of the cross-service APIs remains `/v2/...` because the API surface itself has not changed.

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
- Final user-facing answer generation should normally use the Primary LLM Host.
- Support cognition should use the Utility LLM Host instead of overloading the primary model.
- Equal model-host capabilities do not collapse service ownership.
- Service contracts should be explicit and traceable.

## Hard Hardware Rule

One machine must run no more than one neural model on any single hardware accelerator
(GPU, NPU, or equivalent).

Implications:

- no GPU should host more than one neural model at a time
- no GPU should host both an LLM and an embedding model
- if multiple apps need the same accelerator-bound model, they call the local model host on that machine

A second neural model may be co-located on the same machine if and only if it runs strictly
on CPU and its runtime cannot touch the accelerator. The CPU-only model must not be able to
allocate VRAM, evict accelerator-resident weights, or contend with the accelerator under any
code path. This relaxation exists so a lightweight model such as the Embedding Service can
share a machine with an accelerator-bound LLM Host without taking a second physical machine.

The CPU-co-located service must also leave enough CPU headroom for the accelerator-bound
service's own CPU-side work, such as tokenization, sampling, and runtime IO. The exact
discipline for that is defined in `INTERNAL_INFRASTRUCTURE.md`.

## Physical Topology

### Main Computer

Role:

- host the Primary LLM Host
- expose primary model inference, including image input when the hosted model supports it

Current intended model:

- Gemma 4 (`gemma4:e4b`) on Ollama, with text and image input enabled

This machine should not own retrieval, memory, or orchestration logic.

### Utility Computer

Role:

- host the Utility LLM Host on its accelerator
- expose the same generic model-host capability profile as the Primary LLM Host
- support RAG Engine, Context Fragmenter, and other internal apps
- optionally co-host the Embedding Service as a CPU-only neural model service, subject to the Hard Hardware Rule

Current intended model:

- Gemma 4 (`gemma4:e4b`) on Ollama, with text and image input enabled

This machine may host multiple non-neural apps and exactly one accelerator-bound neural
model service. A CPU-only neural model service such as the Embedding Service may also be
co-located here when its runtime is built and configured to never touch the accelerator and
its CPU usage stays bounded enough to leave headroom for the Utility LLM's CPU-side work.

### Embedding Service Host

The Embedding Service is hosted by whichever machine in the local network is configured to
run it under the CPU-only co-location rules above. In the current local deployment, that
machine is the Utility Computer. The Embedding Service is not pinned to any specific
hardware class as long as the Hard Hardware Rule is satisfied.

A dedicated low-power node such as a Raspberry Pi 5 remains an acceptable alternative
deployment when a future deployment prefers a separate physical box for vectorization. The
contract surface is identical in either case.

## Logical Services

### Chat Orchestrator

The Chat Orchestrator owns the user-turn lifecycle.

It coordinates memory, retrieval, final prompt preparation, model-host calls, direct image routing to the Primary LLM Host when useful for final answering, and final client responses.

### RAG Engine

The RAG Engine owns external knowledge retrieval and evidence packaging.

It may use the Utility LLM Host for query interpretation, query rewriting, image observations, and evidence shaping.

### Context Fragmenter

The Context Fragmenter owns conversational and identity memory.

It may use the Utility LLM Host for salience detection, memory extraction, memory compression, classification, and sleep consolidation.

### Primary LLM Host

The Primary LLM Host owns the health and serving boundary for the primary model.

It accepts the input types advertised by its model-host capabilities and returns text output.

In the normal live user path, the Orchestrator uses this host for final response inference. If the final answer benefits from a user image and this host advertises `imageInput: true`, the Orchestrator should pass the original image directly to this host instead of requiring a Utility LLM Host pre-pass.

It does not know whether the caller is performing chat, final answering, summarization, classification, or another task.

### Utility LLM Host

The Utility LLM Host owns the health and serving boundary for the utility multimodal model.

It accepts text and optional image input and returns text output.

It is not the unique vision node. It is the support model-host capacity used for RAG support, memory support, image-derived retrieval support, background work, and explicitly allowed degraded final-answer fallback.

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

## Shared Model Host Capability, Different Operational Role

Primary LLM Host and Utility LLM Host may run the same model with the same advertised capabilities.

That capability symmetry does not change ownership:

- the Chat Orchestrator owns the final user-facing turn
- the Primary LLM Host is the normal final-answer inference host
- RAG Engine owns retrieval and evidence preparation
- Context Fragmenter owns memory selection and memory maintenance
- Utility LLM Host is support capacity for internal reasoning tasks

RAG Engine and Context Fragmenter may both depend on the Utility LLM Host. This is acceptable because:

- the host is infrastructure
- caller services own semantics
- the first version is a low-scale local network system

Live user-path calls must be able to move ahead of background jobs through model-host numeric
priority. Omitted priority is the lowest model-host priority.

## Multimodal Role

Multimodality is available wherever a model host advertises `imageInput: true`.

For the normal user-answer path, images should be routed by the Orchestrator directly to the Primary
LLM Host when the final answer benefits from seeing the original image.

Images may still be inspected by the Utility LLM Host and converted into text observations when RAG,
memory, or another internal service needs image-derived support data, such as retrieval search intent,
object observations, OCR-like descriptions, or memory classification signals.

Model-host capabilities define whether image input is accepted. Service ownership defines why the
image is being interpreted and which service is allowed to make decisions from that interpretation.

## Canonical Naming

Canonical names in `v3`:

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

Swirlock `v3` is a distributed chatbot architecture in which:

- domain services own product semantics
- model hosts own model health and inference only
- the Primary LLM Host exposes the capabilities of its hosted model
- the Utility LLM Host may expose the same capabilities as the Primary LLM Host
- Primary and Utility model hosts are separated by operational role, not necessarily by model family
- the Orchestrator coordinates final answer generation
- RAG and memory remain separate domains
