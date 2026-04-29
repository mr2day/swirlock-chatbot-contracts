# Swirlock Chatbot Contracts v2

Version `v2` keeps the service boundaries from `v1` but clarifies that LLM-serving applications are model hosts, not semantic services.

The important change is the introduction of a generic Model Host API:

- Primary LLM Host implements the Model Host API with the capabilities of its hosted model.
- Utility LLM Host implements the same generic Model Host API and may expose the same capabilities.
- Model hosts protect model health and expose inference; they do not own chat, RAG, memory, or final-answer semantics.

In the current local architecture, Primary LLM Host and Utility LLM Host may both run `gemma4:e4b`
with text and image input enabled. They are separated by operational role, not by semantic ownership:
Primary is the normal final-answer inference host, while Utility is support capacity for RAG, memory,
image-derived retrieval support, and background cognition.

## Normative Sources

1. `CHATBOT_MANIFEST.md`
2. `openapi/*.openapi.yaml`
3. `INTERACTION_MODEL.md`
4. `API_CONVENTIONS.md`
5. `INTERNAL_INFRASTRUCTURE.md`

## OpenAPI Files

- `openapi/chat-orchestrator.openapi.yaml`
- `openapi/context-fragmenter.openapi.yaml`
- `openapi/rag-engine.openapi.yaml`
- `openapi/model-host.openapi.yaml`

The v1 `main-llm-server` contract is replaced in v2 by the generic `model-host` contract.
