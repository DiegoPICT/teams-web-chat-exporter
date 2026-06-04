# Extension WebSocket Connectivity Plan (v1)

This document defines the minimum v1 design for extension-side WebSocket connectivity to a local bridge/gateway. It is intentionally implementation-first and additive to upstream behavior.

Deferred items are listed in `WEBSOCKET_PARKED.md`.

## Goal

- Add optional `Connect MCP` capability.
- Support bridge-driven conversation enumeration.
- Support snapshot message streaming for one selected Teams conversation.
- Keep existing export behavior unchanged unless user explicitly uses `Connect MCP`.

## Hard v1 Constraints

- Single extension instance.
- Single bridge connection.
- Single active operation.
- No internal operation queue.
- Local-only endpoint (`ws://127.0.0.1:8765/ws` by default).
- No bridge or MCP server implementation in this repository.
- Snapshot/chunk data remains in memory only.

## Guiding Principles

- Minimal implementation for stated goals.
- Additive changes only; no breaking behavior changes.
- Respect upstream architecture and conventions.
- Implementability is the top criterion for v1 decisions.
- Scope loss is a normal lifecycle outcome.
- If a plan item conflicts with implementation reality and becomes a blocker, stop and reassess.

## Naming and Defaults

- Option key: `mcpBridgeUrl`.
- Default: `ws://127.0.0.1:8765/ws`.
- Follow existing defaults/options methodology in `src/utils/options.ts`:
  - add to `Options`,
  - set in `DEFAULT_OPTIONS`,
  - normalize on load.

## Security Posture (v1)

- Local-first by design.
- `ws://` allowed only for loopback hosts (`127.0.0.1`, `localhost`, `::1`).
- Non-loopback endpoints are out of scope.
- No auth token/header in v1.
- User-initiated connect/disconnect only.
- Outbound-only extension integration; no inbound server hosted by the extension.
- No silent background exfiltration.

## Runtime Ownership

### Background

- Owns WebSocket lifecycle.
- Owns busy gate (`BUSY` when already processing).
- Owns tab/scope binding (`tabId` + conversation scope).
- Uses existing upstream scrape path for snapshot generation.

### Popup

- Sends connect/disconnect/status intents.
- Renders connection + operation state.

### Tab Affinity Rule

- Session binds to exactly one Teams tab.
- No implicit tab switching.
- If scope becomes invalid, emit `CONTEXT_LOST` and require explicit restart.

## Protocol (v1)

Protocol id: `teams-exporter-bridge/v1`.

### Envelope

Each frame includes:

- `v`, `type`, `sessionId`, `requestId`, `ts`
- `payload` when applicable
- `error` only for failure frames

### Lifecycle

- Extension -> Bridge: `HELLO`
- Bridge -> Extension: `HELLO_ACK`

### Operations

- `LIST_CONVERSATIONS` -> `CONVERSATIONS` or `ERROR`
- `START_SNAPSHOT` -> `SNAPSHOT_STARTED`, then zero or more `CHUNK`, then `DONE` (or `ERROR`)
- `CANCEL` -> `DONE` (cancelled reason)

### Snapshot Streaming Model

- V1 uses extension push-streaming chunks.
- Implementation note: current upstream scrape path returns a full snapshot before chunking.
- V1 chunking is transport-level paging over in-memory results.

### State Vocabulary

- `DISCONNECTED`
- `CONNECTING`
- `CONNECTED_IDLE`
- `BUSY`
- `CONTEXT_LOST`
- `ERROR`

### Semantics

- `BUSY`: connected, valid scope, active operation.
- `CONTEXT_LOST`: connected transport, invalid bound scope, operation state cleared, explicit restart required.
- Scope-loss detection may be lazy (detected on next scope validation).

### Terminal Events

- `DONE`
- `ERROR`
- `DISCONNECTED`

## Reliability (v1)

- Single message handler with a single-operation busy gate.
- One active operation at a time.
- Use native WebSocket open/close/error events as the transport source of truth.
- Chrome MV3 worker suspension is expected; v1 does not guarantee continuous in-memory continuity.
- Recovery in v1 is explicit user reconnect from popup.

## Compatibility Gate (Pre-merge)

- Loopback WebSocket behavior is a required compatibility gate, not a late integration check.
- Validate v1 flow on supported targets (Chrome MV3 and Firefox MV2 at minimum).
- Confirm manifest/runtime behavior needed for loopback connectivity before merge.

## UX (v1)

- Add `Connect MCP` beside existing export action.
- Keep current export button behavior unchanged.
- Preserve onboarding/export-stop behavior and existing tour targets.
- Show compact connection state and last error text.
- Distinguish `CONNECTED_IDLE` (ready) from `CONTEXT_LOST` (connected but requires restart).
- Keep settings surface minimal (bridge URL only).

## Implementation Boundaries

Additive touch points only:

- `src/utils/options.ts`
- `src/types/messaging.ts`
- `src/entrypoints/background.ts`
- `src/entrypoints/popup/App.svelte` (or one small helper component)
- i18n keys for new UI text

Implementation discipline:

- Prefer strategic hook points over broad refactors.
- Keep churn in `background.ts` and `App.svelte` minimal.
- Avoid exporting broad internal surfaces unless a narrow blocker requires it.

No bridge/server/MCP backend code is added to this repository.

## Acceptance Criteria

- User can connect/disconnect to local bridge from popup.
- Session binds to one Teams tab + conversation scope.
- No implicit tab reassignment.
- `LIST_CONVERSATIONS` works.
- Snapshot stream works with extension push-streaming chunks.
- `BUSY` and `CONTEXT_LOST` semantics are implemented.
- Existing export behavior is unchanged.
- No default snapshot persistence.
- Security guardrails enforce loopback-only plaintext WS.
- New user-facing text follows locale parity requirements.

## Rollback and Risk Control

- Feature is isolated behind explicit `Connect MCP` action.
- If regressions appear, MCP connectivity can be disabled without touching export path.
- No migration of existing storage keys required.
- Non-blocking conflicts become deferred improvements.
- If a conflict is blocker-level and requires high-risk bandaids, stop and reassess.
