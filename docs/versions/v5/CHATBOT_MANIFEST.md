# Chatbot Manifest v5

## Apps

- `swirlock-chatbot-ui`: browser client. Connects only to Chat Orchestrator
  `/v5/chat`.
- `swirlock-coding-agent`: IDE client. Connects only to Chat Orchestrator
  `/v5/chat`.
- (Future) `swirlock-robot-runtime`, `swirlock-figurine-runtime`,
  `swirlock-avatar-runtime`, etc.: physical and virtual frontends. Each
  connects only to Chat Orchestrator `/v5/chat`. The orchestrator is
  frontend-agnostic.
- `swirlock-chat-orchestrator`: owns sessions, turns, prompt assembly, model
  calls, and retrieval coordination. Consumes one Vanamonde LLM Host and
  commands one Context Fragmenter. Does **not** consume the Fragmenter LLM.
- `swirlock-context-fragmenter`: owns background memory consolidation,
  long-term memory formation, and transcript cleanup. Consumes one Fragmenter
  LLM Host. Co-located with the Chat Orchestrator (shared SQLite). Does
  **not** speak to clients directly.
- `swirlock-rag-engine`: owns retrieval planning, local knowledge search, live
  web search, evidence packaging, and citation data. Consumes one RAG-side
  LLM Host (used for query refinement and document retention) and one
  Embedding Service.
- `swirlock-llm-host`: generic Model Host. A single deployment runs a single
  model. Multiple deployments exist (Vanamonde, Fragmenter, RAG-side) — each
  is a separate process, possibly on a separate machine.
- `swirlock-embedding-service`: generic embedding model host.

## Endpoint Registry

| App | Ecosystem WebSocket |
| --- | --- |
| Chat Orchestrator | `/v5/chat` |
| Context Fragmenter | `/v5/fragmenter` |
| Model Host (any deployment) | `/v5/model` |
| RAG Engine | `/v5/retrieval` |
| Embedding Service | `/v5/embeddings` |

There are no ecosystem REST endpoints in `v5`.

## Module-to-LLM Bindings

The 1:1 module-to-LLM rule (see `README.md`) is enforced as concrete
deployment bindings:

| Module | LLM Role | Process | Default Port |
| --- | --- | --- | --- |
| `swirlock-chat-orchestrator` | Vanamonde LLM | `swirlock-llm-host` (Vanamonde deployment) | 3213 |
| `swirlock-context-fragmenter` | Fragmenter LLM | `swirlock-llm-host` (Fragmenter deployment) | 3214 (reserved) |
| `swirlock-rag-engine` | RAG-side LLM | `swirlock-llm-host` (RAG deployment) | 3216 (reserved) |

Each LLM-consuming module sees exactly one Model Host process. A consuming
module never decides at runtime which of multiple Model Hosts to call.

During development and early deployment, a single Model Host process may serve
multiple roles by being pointed at by multiple modules. This is a temporary
substitution explicitly documented in each module's `service.config.cjs`. The
contract still treats each module as having its own LLM; the deployment is
allowed to collapse them physically when only one model is available.

## Co-location

The Chat Orchestrator and Context Fragmenter share a SQLite file with
table-level ownership and **must** run on the same machine.

All other apps may be co-located or distributed independently.
