# Context Fragmenter v4

Endpoint: `WS /v4/context`

The Context Fragmenter is not implemented yet. Its v4 transport rules are still
fixed:

- all memory selection and memory recording messages use the shared envelope
- one persistent socket per caller-service relationship
- cancellation and heartbeat use the common `cancel` and `heartbeat` messages

