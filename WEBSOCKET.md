# WebSocket Docs Index

This file is intentionally brief to avoid duplicated guidance.

Use these documents as the canonical sources:

- `EXTENSION_WEBSOCKET_CONNECT.md`
  - Extension-side v1 plan, constraints, acceptance criteria, and implementation boundaries.
- `WEBSOCKET_PARKED.md`
  - Explicitly deferred (non-v1) behaviors.

Detailed bridge/MCP developer protocol documentation is maintained outside this repository.

## Quick Start (Bridge Developer)

1. Connect to the configured loopback endpoint (default `ws://127.0.0.1:8765/ws`).
2. Receive `HELLO`; respond with `HELLO_ACK`.
3. Send one operation at a time (`LIST_CONVERSATIONS` or `START_SNAPSHOT`) with `requestId`.
4. Handle terminal frames (`DONE`, `ERROR`, socket close) explicitly.
5. Treat `CONTEXT_LOST` as requiring explicit user reconnect from popup.

For extension-side implementation and constraints, follow `EXTENSION_WEBSOCKET_CONNECT.md`.
