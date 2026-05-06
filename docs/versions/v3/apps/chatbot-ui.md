# Chatbot UI

> Repository: `swirlock-chatbot-ui`
> Visible app name: **Gigi the Robot** (the active persona's name).
> Implements: the **Client** in `INTERACTION_MODEL.md`.
> Consumes: [`openapi/chat-orchestrator.openapi.yaml`](../openapi/chat-orchestrator.openapi.yaml).

## Role

The Chatbot UI is the user-facing client of the Swirlock chatbot
ecosystem. It owns:

- Session lifecycle from the user's perspective (start a chat, open one
  from history, delete one).
- Composing user turns and rendering assistant turns, including live
  thinking text, retrieval progress, and streamed tokens.
- Persona presentation — the same UI re-skins for any persona via
  CSS custom properties on `:root`.
- Mobile-first layout, designed to be wrapped by Capacitor for iOS and
  Android without changing the chat surface.

It does **not** own session storage, retrieval, memory, model hosting,
or any cross-service decision. It is the visible surface of whatever
the Chat Orchestrator decides.

## Final Form

When complete, the Chatbot UI provides:

- The full client surface against the Chat Orchestrator:
  - `POST   /v2/chat/sessions` to create a session.
  - `GET    /v2/chat/sessions/:sessionId` to rehydrate a session.
  - `DELETE /v2/chat/sessions/:sessionId` to delete a session.
  - WebSocket `/v2/chat/sessions/:sessionId/turns/stream` for every turn.
- v3 envelope handling — surfaces both successful `data` shapes and the
  v3 `ErrorEnvelope` (orchestrator-side validation failures, transport
  errors, upstream Model Host failures) inline on the message bubble
  and as a banner above the composer.
- WebSocket bearer auth on the upgrade using one of the three transports
  pinned by `API_CONVENTIONS.md#websocket-authentication`. The browser
  variant defaults to `?token=<token>` because the standard `WebSocket`
  constructor cannot set custom headers.
- Live rendering of every chat-stream event:
  - `accepted` / `queued` / `started` — connection lifecycle indicators.
  - `retrieval` — RAG progress events forwarded by the orchestrator,
    surfaced as friendly inline labels ("Searching the web…", "Reading
    sources…", etc.).
  - `thinking` — collapsed-by-default reveal next to the assistant
    avatar.
  - `chunk` — appended into the assistant bubble character-by-character.
  - `done` — terminal success; persists the assistant message id and
    turn id, displays the **Sources** disclosure when `citations` are
    present, attaches `diagnostics` when the orchestrator returns them.
  - `error` — terminal failure surfaced both inline and as a banner.
- A persona system that is both a UI skin and an LLM personality. The
  persona id is sent to the orchestrator on session creation as
  `app.personaId`. The UI applies the persona's theme as `--persona-*`
  custom properties on `:root` so swapping personas re-skins the whole
  app instantly. Adding a new persona is one new file plus one registry
  entry; no other code changes.
- Dark theme by default, with the background of the chat surface
  matched to the persona logo's background.
- Mobile-first layout: collapsible sidebar drawer with scrim on small
  viewports, inline sidebar on `≥768px` viewports, safe-area insets for
  iOS, hamburger toggle, virtual-keyboard-friendly composer.
- Capacitor wrapping for native iOS and Android.
- Accessibility: keyboard nav, sane focus rings, `aria-live` on the
  message stream, accessible names on icon buttons.

## Roadmap

In priority order:

1. **Persona switcher** in the topbar. Today the registry is read-only
   because there is only one persona.
2. **Additional personas** (Gigina Robotina, The English Teacher, The
   Italian Actor). Each is one TS file in `src/app/core/personas/` plus
   one logo asset in `public/personas/<id>/`.
3. **Settings panel** to override the bearer token and orchestrator
   base URL at runtime, instead of editing
   `src/app/core/config/runtime-config.ts`.
4. **Edit and regenerate** user messages, ChatGPT-style.
5. **File and image attachments**, after the orchestrator wires up
   `imageId` resolution via a deployment-provided media resolver.
6. **Capacitor wrappers** for iOS and Android. The contract stays the
   same; the only Capacitor-specific change is the bearer token coming
   from a secure preference instead of `RuntimeConfig`.
7. **Voice input/output**.
8. **Conversation search** across the locally-cached session list and
   the orchestrator's stored history.

## Current Implementation

Built as a thin v0.1 slice that already covers the full chat happy
path:

- Angular 21, signals, standalone components, zoneless change detection,
  control flow syntax. No unit tests. Separate `.ts` / `.html` /
  `.scss` files. TypeScript `strict`.
- Single hardcoded dev user (`dev-user`), single hardcoded bearer token
  (`dev-token-change-me`) — in lockstep with
  `swirlock-chat-orchestrator/service.config.cjs`. Both come from
  `DEFAULT_RUNTIME_CONFIG` in `src/app/core/config/runtime-config.ts`.
- One persona in the registry: **Gigi the Robot**. Its theme drives the
  whole UI. The browser tab title is set from the active persona's
  name via an `effect()` so adding personas later just works.
- Sidebar with sessions grouped by relative date (Today / Yesterday /
  Previous 7 days / Previous 30 days / Older), new-chat button,
  delete-on-hover affordance. Sessions list is cached in `localStorage`
  so the sidebar renders instantly on reload; full message history is
  rehydrated from the orchestrator on demand.
- Composer with auto-resize textarea, Enter-to-send / Shift+Enter for
  newline, send-and-stop button states, hint line on desktop.
- Message bubble renders Markdown via `marked` + `DOMPurify`, shows the
  persona avatar on assistant messages, displays:
  - Retrieval progress as a small inline label,
  - Thinking text as a collapsible block,
  - Streaming dots while waiting for the first chunk,
  - **Sources** disclosure when the `done` event includes citations.
- Mobile drawer for the sidebar with scrim, inline sidebar on
  `≥768px`. Safe-area insets respected for iOS notch / home indicator.
- Build verified clean against Angular 21 with the production builder.

What is not yet built:

- Persona switcher UI (the registry is read-only).
- Additional personas.
- Settings panel.
- Edit / regenerate user messages.
- Image / file attachments.
- Capacitor wrappers.
- Voice.
- Conversation search.

## Notes for orchestrator implementers

The Chatbot UI is forgiving but expects the orchestrator's stream
contract verbatim:

- The `done` event must carry the persisted `turnId`,
  `assistantMessage.messageId`, and `assistantMessage.createdAt`. The UI
  swaps its provisional client-side ids for these so deletion and
  rehydration line up with what the server actually persisted.
- `retrieval` event types are passed through verbatim. The UI maps the
  known RAG event-type strings to friendly labels in
  `src/app/core/services/session.service.ts`'s `RETRIEVAL_LABELS`. New
  RAG event types not in the map fall back to the raw type string.
- `error` is always terminal — the UI marks the message and stops the
  stream regardless of `retryable`. Retry is the user's decision.
