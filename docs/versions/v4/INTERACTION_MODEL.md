# Interaction Model v4

## User Chat Turn

1. The UI opens one persistent WebSocket to Chat Orchestrator `/v4/chat`.
2. The UI sends `session.create` when it needs a new chat session.
3. The UI sends `turn.submit` for each user message. The payload contains the
   `sessionId` and the semantic turn request.
4. The Orchestrator classifies the turn through its persistent Model Host
   socket using a utility inference with thinking disabled.
5. If retrieval is needed, the Orchestrator sends `retrieve_evidence` over its
   persistent RAG Engine socket.
6. The RAG Engine uses persistent sockets to its Utility Model Host and
   Embedding Service dependencies.
7. The Orchestrator sends final-answer `infer` over its persistent Model Host
   socket.
8. The Orchestrator streams `turn.*` events back to the UI over the original
   `/v4/chat` socket and persists the turn before `turn.done`.

## Routing

Every async operation has a `correlationId`. Services may interleave events from
multiple operations on the same WebSocket. Callers must route by
`correlationId`, not by connection order.

## Session Management

Session creation, session loading, session deletion, and turn submission are
all messages on `/v4/chat`.

The Chat Orchestrator is the canonical owner of sessions and persisted chat
messages. UI clients may cache summaries locally, but they reload canonical
history through `session.get`.

