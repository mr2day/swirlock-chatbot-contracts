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

## Conversation Text Integrity

Services must not apply deterministic word, phrase, or regular-expression
filters to conversational user or assistant text for semantic behavior control,
including greeting removal, persona enforcement, moderation, language detection,
intent routing, or safety decisions. The conversation language may be unknown,
mixed, transliterated, or invented, so lexical filters create silent,
language-biased behavior.

Any semantic evaluation of conversational text must be performed by an LLM
through a Model Host call. This evaluation should be part of an existing
classification, planning, moderation, or final-answer step. Services must not
add a dedicated extra model turn solely to detect whether a message is a
greeting.

Services may deterministically act on explicit protocol fields and structured
LLM outputs, such as a classifier's `intent` or `route`, because the semantic
text evaluation has already happened inside the model call.

Services may still perform transport-level and structure-level handling that
does not evaluate natural-language meaning, such as JSON validation, size
limits, envelope routing, chunk forwarding, persistence, escaping for display,
and lossless whitespace preservation or trimming at storage boundaries.
