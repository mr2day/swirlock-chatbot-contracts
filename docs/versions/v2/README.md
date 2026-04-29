# Swirlock Chatbot Contracts v2

Version `v2` keeps the service boundaries from `v1` but clarifies that LLM-serving applications are model hosts, not semantic services.

The important change is the introduction of a generic Model Host API:

- Primary LLM Host implements the Model Host API with text input only.
- Utility LLM Host implements the Model Host API with text and image input.
- Model hosts protect model health and expose inference; they do not own chat, RAG, memory, or final-answer semantics.

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
