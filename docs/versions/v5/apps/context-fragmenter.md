# Context Fragmenter v5

Endpoint: `WS /v5/fragmenter`

The Context Fragmenter is a peer module to the Chat Orchestrator. Its job is
to run **background consolidation** of conversation history so the
Orchestrator's next prompt is built from a richer, cleaner, more compressed
view than raw turn history alone.

The Fragmenter consumes the **Fragmenter LLM** (a Model Host process,
independent from the Vanamonde LLM Host the orchestrator consumes). The
Orchestrator never sees the Fragmenter LLM. The Orchestrator never blocks on
Fragmenter work.

## Co-location and Shared SQLite

The Fragmenter and Orchestrator **must run on the same machine** because they
share a single SQLite database file with **table-level ownership**:

- The **Orchestrator** owns the live conversation tables: `sessions`,
  `messages`, `agent_events`, `agent_plans`, `agent_plan_steps`,
  `personas`, `persona_*`, `identity_mutation_candidates`. It is the only
  writer for these tables. The Fragmenter may read them.
- The **Fragmenter** owns its own working tables (intermediate state,
  per-session digestion scratch space) and its own **result tables** that
  contain the consolidated views the Orchestrator will read. It is the only
  writer for these tables. The Orchestrator may only read them.
- The **Fragmenter** is responsible for migrating its own tables. The
  Orchestrator's migrations never touch Fragmenter-owned tables, and vice
  versa.

Concurrent access uses SQLite's WAL mode. The Orchestrator's prompt builders
read consolidation tables with normal `SELECT` statements at prompt-assembly
time; no IPC is involved.

The exact Fragmenter result-table schema is owned by the Fragmenter
implementation and described separately in its repo. Stable identifiers are
`sessionId` (matches `sessions.id` in the Orchestrator's table) and
`personaId` (matches `personas.id`).

## Client Messages (from Orchestrator → Fragmenter)

All Orchestrator → Fragmenter messages are **fire-and-forget** unless
explicitly noted otherwise. The Orchestrator does not wait for any reply
on the live-turn hot path.

- `session.observed`: a session has activity worth considering for
  consolidation. Sent after the Orchestrator persists a turn.
  Payload:
  ```json
  {
    "sessionId": "string",
    "lastTurnId": "string",
    "lastSeq": 12,
    "observedAt": "2026-05-09T12:00:00Z"
  }
  ```
  No reply expected. The Fragmenter decides if and when to consolidate.
- `session.invalidate`: the Orchestrator deleted a session. The Fragmenter
  should drop its working state and result rows for that session.
  Payload:
  ```json
  { "sessionId": "string", "reason"?: "string" }
  ```
  No reply expected.
- `health.get`: standard health probe. Payload may be omitted. Fragmenter
  replies with a `health` event.
- `cancel`: cancels in-flight consolidation work for the given
  `correlationId` if applicable. Cancel does not apply to fire-and-forget
  notifications.
- `heartbeat`: standard liveness probe.

The contract intentionally does **not** define an RPC by which the
Orchestrator queries consolidation results. Consolidation results live in
the shared SQLite file and the Orchestrator reads them with plain SQL.

## Server Events (from Fragmenter → Orchestrator)

- `consolidation.updated`: optional notification, emitted when a
  consolidation run finishes and result tables for a session were updated.
  The Orchestrator does not need to act on this; it exists so that an
  Orchestrator implementation that caches consolidations in process memory
  can invalidate that cache cheaply.
  Payload:
  ```json
  {
    "sessionId": "string",
    "consolidationKind": "string",
    "occurredAt": "2026-05-09T12:00:05Z"
  }
  ```
  `consolidationKind` is a Fragmenter-defined identifier (e.g.
  `"session.summary"`, `"identity.user"`, `"identity.app"`,
  `"answer.reality_check"`) describing which result-table family was
  updated.
- `health`: payload is `{ "status": "ok"|"degraded"|"unavailable", "ready": boolean }`.
- `error`: standard error envelope.
- `heartbeat`: standard heartbeat reply.

## Scheduling

The Fragmenter chooses its own scheduling policy. The contract makes no
statement about whether consolidation runs immediately on each
`session.observed`, on a debounced schedule, on idle ticks, or on explicit
maintenance windows. Suggested operational modes:

- **Live nudge**: `session.observed` arrives, Fragmenter prioritises the
  consolidation that the Orchestrator is most likely to need on the next
  turn (`priority: background`).
- **Idle catch-up**: no recent `session.observed` for some interval, the
  Fragmenter walks its backlog of un-consolidated sessions
  (`priority: background`).
- **Deep maintenance**: scheduled or explicitly invoked, the Fragmenter
  performs expensive cross-session work like persona long-term-memory
  formation (`priority: maintenance`).

The Fragmenter's scheduling is its private concern and may evolve without a
contract change.

## Failure Behaviour and Degradation

- If the Fragmenter is unreachable, the Orchestrator's
  `session.observed`/`session.invalidate` notifications are dropped (after
  a short retry on the persistent socket). The user-facing turn is
  unaffected.
- If the Fragmenter LLM call fails, the Fragmenter logs and skips that
  consolidation. The Orchestrator on the next turn simply finds the same
  result rows it had before, or none.
- The Orchestrator's prompt builders **must** treat the presence of any
  consolidated view as optional. A turn must build a coherent prompt even
  when no consolidation rows exist for the session.

## Out of Scope for v5 MVP

The v5 contract reserves room for these but does not require them in the
MVP implementation:

- Cross-app maintenance commands (e.g. an `orchestrator.idle` or
  `orchestrator.busy` hint signalling resource availability) — may be added
  in a later v5 minor revision without breaking existing implementations.
- Persona-level long-term-memory consolidation — Fragmenter implementations
  may add this when ready; result tables for it use a distinct
  `consolidationKind`.

## Migration from v4

`v4` defined this app only as a placeholder ("not implemented yet"). `v5`
gives it a real surface. There is no v4 → v5 wire compatibility because no
v4 implementation existed.
