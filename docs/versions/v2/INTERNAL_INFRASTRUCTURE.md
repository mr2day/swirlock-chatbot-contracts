# Internal Infrastructure v2

## Purpose

This document defines guidance for infrastructure services that support the Swirlock ecosystem but do not own product semantics.

## Runtime Configuration Source Of Truth

Every deployable service must have exactly one obvious source of truth for runtime configuration.

Runtime configuration means deploy-time or machine-specific values such as:

- host and port
- upstream service URLs
- model IDs and model allow-lists
- model keep-alive settings
- feature flags such as image input or thinking defaults
- request body size limits
- process-manager environment values

These values must not be duplicated across `.env`, `.env.example`, source-code defaults,
process-manager config, README examples, and ad hoc launch commands.

For Node services, the preferred local pattern is:

- put a committed root config file in the service repo, such as `host.config.cjs` for a model host
  or `service.config.cjs` for a domain service
- export one object containing the runtime values, commonly named `env`
- import that same object from the app bootstrap/config loader
- import that same object from the process-manager config, such as `ecosystem.config.cjs`
- make README setup instructions point to that file as the place to edit runtime settings
- fail fast during startup if required config values are missing or internally inconsistent

Service implementations should avoid source-code fallback values for deploy-time settings. A missing
model ID, upstream URL, host, port, or equivalent runtime setting should be treated as a
configuration error, not silently replaced with a hidden default.

Secrets may still come from a private secret store or an uncommitted local file when needed, but the
service must still have one named config-loading path. Secret loading must not reintroduce multiple
competing places for the same parameter.

The goal is operational clarity: changing a model, port, upstream URL, or equivalent runtime setting
must require one edit in one predictable place.

## Local Node/Nest Process Management

Local Node/Nest services that are intended to keep running on LAN machines should run under PM2,
not under an open development terminal.

This applies to local Windows or workstation/server machines that host Swirlock services directly,
such as:

- Primary LLM Host
- Utility LLM Host
- local RAG Engine instances
- local Context Fragmenter instances
- other local NestJS services that are not managed by a cloud platform

Cloud-hosted services are different. Services deployed to Google Cloud should use the native Google
Cloud runtime and process supervision for that product, such as Cloud Run, GKE, App Engine, Compute
Engine service management, or another explicitly selected Google-managed deployment path. Do not
add PM2 inside a Google-managed runtime unless the deployment target is intentionally a plain VM and
PM2 is the chosen process manager for that VM.

The standard PM2 pattern for local NestJS services is:

```powershell
npm install -g pm2
npm install
npm run build
pm2 start ecosystem.config.cjs
pm2 status
pm2 save
```

Each local Node/Nest service should provide an `ecosystem.config.cjs` file. That file must import
the same root runtime config object used by the application itself, as described in
`Runtime Configuration Source Of Truth`.

The PM2 ecosystem file should run compiled production output, not the Nest watch server. For NestJS
apps, that usually means:

```js
module.exports = {
  apps: [
    {
      name: 'service-name',
      script: 'dist/main.js',
      instances: 1,
      autorestart: true,
      watch: false,
      env,
    },
  ],
};
```

Model hosts should normally use one process instance unless a host explicitly documents that the
machine and model runtime can safely run more than one process. Model concurrency should be
controlled inside the service by model-slot and queue rules, not accidentally multiplied by a process
manager.

Useful operational commands:

```powershell
pm2 status
pm2 logs <service-name>
pm2 restart <service-name>
pm2 stop <service-name>
pm2 delete <service-name>
```

After changing runtime config or rebuilding code, local operators should restart through the
ecosystem file and save the process list:

```powershell
npm run build
pm2 restart ecosystem.config.cjs --update-env
pm2 save
```

For restart after machine reboot, the saved PM2 process list is restored with:

```powershell
pm2 resurrect
```

On Linux and macOS, PM2 startup hooks may be used where appropriate. On Windows, PM2's built-in
startup hook may not be available; a Windows service wrapper, Task Scheduler entry, or current-user
Startup-folder script may be used to run `pm2 resurrect`. Use a service or elevated scheduled task
when the service must start before user logon. A current-user startup script is acceptable only when
the machine's operational model includes a logged-in service user.

The local production rule is: long-running services are started, monitored, restarted, and resurrected
by the process manager. Development commands such as `nest start --watch`, `npm run start:dev`, or a
visible terminal running `node dist/main.js` are not the steady-state production run mode.

## Model Host Principle

A model host is an agnostic model appliance.

It owns:

- model loading
- model keep-alive
- model unloading
- inference transport
- health/readiness reporting
- model-health protection
- model-slot serialization and queue reporting

It does not own:

- chat semantics
- RAG semantics
- memory semantics
- image storage semantics
- business task interpretation
- prompt policy beyond local safety/system guardrails

Two model hosts may run the same model and advertise the same capabilities. They remain separate
infrastructure endpoints because callers use them for different operational roles.

## Model Host Profiles

### Primary LLM Host

Profile:

- text input
- image input when advertised by the hosted model
- text output
- no image output

Typical caller:

- Chat Orchestrator

Common usage:

- final user-facing response inference from prepared input
- direct interpretation of original user images when final answering benefits from vision

### Utility LLM Host

Profile:

- text input
- image input when advertised by the hosted model
- text output
- no image output

Typical callers:

- RAG Engine
- Context Fragmenter
- Chat Orchestrator for support cognition or explicitly allowed degraded final-answer fallback

Common usage:

- image-derived retrieval support
- query rewriting
- classification
- compression
- evidence shaping
- memory support
- background memory and sleep jobs

The Utility LLM Host does not know which of these tasks it is serving. The caller owns the prompt and interpretation.

Primary LLM Host and Utility LLM Host may be implemented by the same codebase, expose the same
Model Host API, and run the same model ID, such as `gemma4:e4b`. They should still be configured and
documented as separate deployed services because the Primary host protects the live final-answer
path, while the Utility host absorbs support and background cognition.

## Responsibility-Based Routing

Routing must be based on service responsibility, not just on which host can technically process an
input.

Normal routing:

- Chat Orchestrator sends final-answer inference to Primary LLM Host.
- Chat Orchestrator may pass original user images directly to Primary LLM Host when it advertises
  `imageInput: true`.
- RAG Engine sends retrieval-support inference to Utility LLM Host.
- Context Fragmenter sends memory-support and sleep inference to Utility LLM Host.
- Background jobs should use low or omitted model-host priority.

Avoid these patterns:

- forcing every image through Utility LLM Host before final answering
- moving retrieval or memory decisions into either model host
- using Primary LLM Host for background support work unless explicitly elevated
- using Utility LLM Host for normal final answers unless the Orchestrator is in an explicit degraded
  fallback mode

## Safety Requirements

Every model host should protect the hosted model and local machine through:

- keep-alive behavior
- preload/unload lifecycle behavior
- one active model request at a time unless the host explicitly documents more slots
- a visible queue for additional requests
- process-level request body protection

Model hosts should not impose task-level limits such as output token caps or elapsed-time caps.
Callers own task policy and may pass model-runtime options when the host contract allows it.

## Queue Requirements

Model hosts with a single model slot should queue additional requests. Queue order is:

1. higher numeric `requestContext.priority` first
2. earlier arrival first when priorities are equal

If `requestContext.priority` is omitted, the request is treated as lower priority than every request
that provides a finite priority number.

Streaming model-host APIs should report queue position, queue depth, requests ahead, and estimated
start time when enough recent runtime samples exist. Queue estimates are advisory.

## Health Requirements

Every model host should expose:

- liveness
- readiness
- configured model ID
- loaded/running state
- capabilities
- active request count
- model slot count
- queue depth
- average request duration when available

## Media Handling

Domain services exchange media through `imageId` or `imageUrl`.

Model hosts may accept:

- `imageUrl`
- direct base64 image payloads

Model hosts should not become durable media stores. If a caller has an `imageId`, the caller should resolve it to bytes or URL before model inference unless the deployment explicitly provides a shared media resolver.
