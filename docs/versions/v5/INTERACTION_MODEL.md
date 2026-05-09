# Interaction Model v5

## User Chat Turn

A live user turn is a strict sequential pipeline. There is no second LLM in
the conversation hot path; the Vanamonde LLM Host is the only Model Host the
orchestrator calls during a turn.

1. The frontend (UI, coding agent, robot runtime, character runtime, etc.)
   opens one persistent WebSocket to Chat Orchestrator `/v5/chat`.
2. The frontend sends `session.create` when it needs a new chat session.
3. The frontend sends `turn.submit` for each user message. The payload
   contains the `sessionId` and the semantic turn request.
4. The Orchestrator persists the user message, builds a control-step prompt,
   and sends `infer` (JSON-mode, `temperature: 0`, brief output budget) over
   its persistent Vanamonde LLM Host socket. The Vanamonde LLM returns a
   control frame: either a tool/command to execute, or a signal that the
   final answer can be written.
5. If a tool was requested (e.g. `rag.retrieve`), the Orchestrator runs it
   over the appropriate persistent socket (RAG Engine, location request,
   etc.), records the result as an observation, and returns to step 4 for the
   next control-step decision. The same Vanamonde LLM Host is used for every
   control step.
6. When the control step signals "ready to answer", the Orchestrator builds
   the final-answer prompt and sends a streaming `infer` (text mode, full
   response budget) over the same Vanamonde LLM Host socket.
7. The Orchestrator streams `turn.*` events back to the frontend over the
   original `/v5/chat` socket and persists the assistant message before
   `turn.done`.
8. After persisting the turn, the Orchestrator sends a fire-and-forget
   `session.observed` to the Context Fragmenter (see "Context Fragmenter
   Coordination" below). The user-facing turn is already complete; this
   notification does not block.

## Context Fragmenter Coordination

The Context Fragmenter runs continuously alongside the Orchestrator and
prepares consolidated views of conversation history that the Orchestrator
will consume on later turns.

Coordination is intentionally minimal:

1. After persisting a new turn, the Orchestrator sends `session.observed`
   over its persistent Fragmenter socket. This is a fire-and-forget
   notification; the Orchestrator does not wait for any reply and does not
   block the user-facing turn on it.
2. The Fragmenter decides — using its own scheduling policy — when to run
   consolidation work for that session. Consolidation may include:
   - rolling per-session summaries,
   - long-term memory candidate extraction,
   - transcript cleanup (transliteration normalisation, model-output artefact
     removal),
   - cross-session persona-level memory formation.
3. When consolidation completes, the Fragmenter writes results into its
   own result tables in the **shared SQLite file** (see
   `apps/context-fragmenter.md` for table-level ownership rules). It may also
   emit an optional `consolidation.updated` event over the Orchestrator
   socket so the Orchestrator can invalidate any in-memory cache; the
   Orchestrator is not required to act on this event.
4. On the next turn, the Orchestrator's prompt builders read whatever
   consolidation is currently present in the result tables. There is no RPC,
   no synchronous fetch, no waiting on the Fragmenter LLM. If no
   consolidation is present (Fragmenter hasn't run yet, or is unavailable),
   the Orchestrator builds the prompt from raw history alone and the turn
   proceeds normally.

The Orchestrator may also send the following operational hints (none of which
block a turn):

- `session.invalidate` — the Orchestrator deleted a session; the Fragmenter
  should drop its working state and result rows for that session.

The Fragmenter chooses its own scheduling policy. Operational priority of
work uses the standard `priority` values from `API_CONVENTIONS.md`:
`background` for live-session catch-up, `maintenance` for idle-time deep
work.

## Routing

Every async operation has a `correlationId`. Services may interleave events
from multiple operations on the same WebSocket. Callers must route by
`correlationId`, not by connection order.

For fire-and-forget notifications without a server reply (e.g.
`session.observed`), the `correlationId` exists for trace-log correlation
only.

## Session Management

Session creation, session loading, session deletion, and turn submission are
all messages on `/v5/chat`.

The Chat Orchestrator is the canonical owner of sessions and persisted chat
messages. UI and other frontend clients may cache summaries locally, but they
reload canonical history through `session.get`.

When a session is deleted, the Orchestrator deletes its rows from the live
tables it owns and notifies the Fragmenter via `session.invalidate`. The
Fragmenter is responsible for cleaning up its own tables for that session.

## Conversation Text Integrity

Services must not apply deterministic word, phrase, or regular-expression
filters to conversational user or assistant text for semantic behavior
control, including greeting removal, persona enforcement, moderation,
language detection, intent routing, or safety decisions. The conversation
language may be unknown, mixed, transliterated, or invented, so lexical
filters create silent, language-biased behavior.

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

A mechanical text *transform* driven by structured trace data — for example,
substituting a prior assistant message in the control-step's view of history
with an objective summary derived from `agent_events` — is permitted, because
the substitution makes no semantic decision about conversational meaning;
it merely projects a different view of already-recorded structured events.
A mechanical *decision* on conversational meaning is not permitted.
