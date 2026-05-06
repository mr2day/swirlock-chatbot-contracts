# Chatbot Manifest v4

## Apps

- `swirlock-chatbot-ui`: browser client. Connects only to Chat Orchestrator
  `/v4/chat`.
- `swirlock-coding-agent`: IDE client. Connects only to Chat Orchestrator
  `/v4/chat`.
- `swirlock-chat-orchestrator`: owns sessions, turns, prompt assembly, model
  calls, memory calls, and retrieval coordination.
- `swirlock-rag-engine`: owns retrieval planning, local knowledge search, live
  web search, evidence packaging, and citation data.
- `swirlock-llm-host`: generic model host for primary and utility LLM roles.
- `swirlock-embedding-service`: generic embedding model host.
- `context-fragmenter`: future memory service.

## Endpoint Registry

| App | Ecosystem WebSocket |
| --- | --- |
| Chat Orchestrator | `/v4/chat` |
| Model Host | `/v4/model` |
| RAG Engine | `/v4/retrieval` |
| Embedding Service | `/v4/embeddings` |
| Context Fragmenter | `/v4/context` |

There are no ecosystem REST endpoints in `v4`.

