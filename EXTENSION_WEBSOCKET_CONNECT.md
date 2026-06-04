# Extension WebSocket Connectivity Plan

Planning document for adding extension-side WebSocket connectivity to a local bridge/gateway process, while keeping this repository focused on the browser extension only.

This document defines scope, protocol, constraints, and acceptance criteria before coding.

## 1) Purpose

- Add a new optional capability: connect the extension to a local bridge via WebSocket.
- Enable bridge-driven conversation enumeration and snapshot message iteration for the selected Teams conversation.
- Keep the existing export workflow intact and unchanged for users who do not use the new feature.

## 2) Guiding Principles

- Minimal implementation for stated goals.
- Additive changes only; no breaking behavior changes.
- Respect upstream architecture and conventions.
- Local-first trust boundary aligned with project philosophy (no third-party server dependency in v1).
- Keep protocol explicit, stable, and small.
- Keep runtime artifacts in memory by default.
- Treat scope loss as normal lifecycle, not an edge case.

## 3) Scope (In)

- Extension-side WebSocket client in background runtime.
- Popup action to connect/disconnect and view connection status.
- Bridge URL option using existing defaults/options methodology.
- Conversation listing operation over WebSocket.
- Snapshot iterator operation over WebSocket (pull-based chunking).
- Explicit protocol lifecycle and terminal events.
- V1 is single-instance and single-session: one bridge connection and one active stream/request at a time.

## 4) Non-Goals (Out)

- Implementing MCP server logic in this repository.
- Implementing bridge/gateway server in this repository.
- Remote endpoint support in v1.
- Multi-session orchestration, advanced routing, or high-throughput tuning.
- Persisting snapshots/chunks to disk or storage by default.

## 5) Upstream Mergeability Criteria

- No changes to existing export semantics, status flow, or file outputs.
- No behavior changes unless user explicitly invokes Connect MCP.
- New option keys and runtime messages are additive.
- Existing storage keys and message contracts remain compatible.
- UI changes are contained and do not regress current interaction patterns.
- Any new docs describe behavior plainly without over-prescribing implementation details.

## 6) Naming and Configuration

### 6.1 Option Naming

- Configuration key name: `mcpBridgeUrl`.
- Rationale: endpoint represents the local bridge/gateway, not an MCP server endpoint.

### 6.2 Default Value

- Default bridge URL: `ws://127.0.0.1:8765/ws`.
- Port choice avoids common conflicts with frequently used local ports such as 8000.

### 6.3 Defaults Methodology

- Follow existing options patterns in `src/utils/options.ts`:
  - define default constant(s),
  - include in `Options` type,
  - include in `DEFAULT_OPTIONS`,
  - normalize during options load.

## 7) Security and Trust Model (v1)

- Local-only in v1.
- `ws://` allowed only for loopback hosts (`127.0.0.1`, `localhost`, `::1`).
- Non-loopback endpoints are out of scope for v1.
- No auth token/header in v1 by design because deployment is local-only and first-party.
- User-initiated connect/disconnect only.
- Do not log secrets or sensitive payloads.
- Outbound-only extension integration; no inbound server hosted by the extension.
- No silent background exfiltration: bridge traffic starts only after explicit user connect.

## 8) Runtime Placement and Responsibilities

### 8.1 Background Runtime

- Owns WebSocket session lifecycle.
- Owns reconnect loop and queueing.
- Owns active stream lock and busy handling.
- Bridges popup controls to protocol operations.
- Uses existing extension scrape pipeline for snapshot generation.
- Owns tab/scope binding for a session (`tabId` + conversation scope).

### 8.1.1 Tab Affinity (Mandatory)

- A bridge session is bound to exactly one Teams tab at bind/start time.
- Binding is explicit and stable for the session (`tabId` + conversation scope).
- No implicit tab switching or "best guess" tab replacement mid-session.
- If the bound tab closes, navigates away from Teams scope, or no longer matches required conversation scope, the session enters `CONTEXT_LOST`.

### 8.2 Popup Runtime

- Sends intent messages only (connect, disconnect, status).
- Renders status and errors.
- Does not own WebSocket socket state.

### 8.3 Content Runtime

- No new protocol ownership.
- Existing scrape mechanisms remain source of conversation and message data.

## 9) Protocol Contract (v1)

Protocol identifier: `teams-exporter-bridge/v1`.

### 9.1 Envelope Requirements

Every frame includes:

- `v`: protocol version number.
- `type`: event name.
- `sessionId`: stable for the connection/session.
- `requestId`: unique per operation/stream.
- `ts`: timestamp.
- `payload`: operation data when applicable.
- `error`: structured error object for terminal failures.

Unknown `type` values are ignored safely.

### 9.2 Lifecycle and Health Events

- Extension to bridge:
  - `HELLO`.
  - `PONG`.
- Bridge to extension:
  - `HELLO_ACK`.
  - `PING`.

### 9.3 Conversation Enumeration

- Bridge requests listing with `LIST_CONVERSATIONS`.
- V1 request shape is minimal and has no mode toggles.
- Extension replies with `CONVERSATIONS` or terminal `ERROR`.
- Conversation and folder payload shapes align with extension domain types:
  - `ConversationSummary`.
  - `FolderSummary`.

### 9.4 Snapshot Iterator (Selected Conversation)

- Bridge starts operation with `START_SNAPSHOT` including:
  - target conversation id,
  - optional batch size,
  - optional filter/include options.
- Extension acknowledges with `SNAPSHOT_STARTED` including:
  - resolved conversation context,
  - total message count,
  - iterator starting cursor.
- Bridge requests chunks using `NEXT` with cursor.
- Extension returns `CHUNK` with:
  - current cursor,
  - next cursor or null,
  - count,
  - message items.
- Stream ends explicitly with `DONE`.
- Failure ends explicitly with `ERROR`.

### 9.5 Cancellation and Busy

- Bridge may send `CANCEL` for active request.
- Extension ends request with terminal `DONE` carrying cancelled completion reason.
- If a new snapshot request arrives while one is active, extension returns terminal `ERROR` with busy classification.

### 9.5.1 BUSY and CONTEXT_LOST Semantics

- `BUSY` means transport is connected, scope is valid, and extension is actively processing a request.
- `CONTEXT_LOST` means transport is still connected, but bound scope is invalid.
- On `CONTEXT_LOST`, extension stops streaming, clears in-memory iterator/request state, updates UI to degraded-but-connected, and waits for explicit rebind/restart.
- Bridge treats `CONTEXT_LOST` as non-fatal and idempotent (restart from clean baseline).

### 9.6 Terminal Signals

Terminal outcomes are explicit and never inferred from silence:

- `DONE`.
- `ERROR`.
- `DISCONNECTED`.

### 9.7 Connection and Operation State Vocabulary (v1)

- `DISCONNECTED`: no active transport.
- `CONNECTING`: transport handshake in progress.
- `CONNECTED_IDLE`: transport connected, scope valid, waiting.
- `BUSY`: transport connected, scope valid, actively listing/snapshotting/streaming.
- `CONTEXT_LOST`: transport connected, scope invalid, explicit rebind required.
- `ERROR`: terminal operation failure; transport may still be recoverable.

## 10) Error Model

Standard error classifications:

- invalid request.
- unsupported event type.
- no Teams tab available.
- content runtime unavailable.
- conversation not found.
- scrape failed.
- timeout.
- busy.
- internal error.

Error payload includes:

- stable error code,
- user-readable message,
- retriable flag,
- optional details for diagnostics.

## 11) Reliability Model

- Single reader model for inbound WebSocket frames.
- Internal queue between socket receiver and operation handlers.
- One active stream/request at a time for the extension instance.
- Queue drain/reset before accepting a new request after completion.
- One reconnect timer only; fixed interval; deduplicated scheduling.
- Chrome MV3 service-worker suspension is expected behavior; v1 does not promise continuous in-memory runtime continuity.
- Reconnect and deterministic status re-declaration are normal recovery paths.
- Layered timeouts:
  - connect timeout,
  - snapshot generation timeout,
  - per-request/stream timeout.

## 11.1 Compatibility Gate (Pre-merge)

- Loopback WebSocket behavior is a required compatibility gate, not a late integration check.
- Validate the v1 flow on supported targets (Chrome MV3 and Firefox MV2 at minimum).
- Confirm manifest/runtime behavior needed for loopback connectivity before merge.

## 12) Data Handling and Artifact Hygiene

- Snapshots/chunks remain in memory by default.
- No default persistence to files or storage.
- If optional diagnostics are added later:
  - explicit opt-in,
  - sensitive-value redaction,
  - local artifact paths excluded via gitignore rules.

## 13) UX Plan (Minimal)

- Split primary action area to include `Connect MCP` next to existing export action.
- Keep current export button behavior unchanged.
- Preserve onboarding/export-stop behavior and existing tour targets.
- Show compact connection state and last error text for MCP connectivity.
- Do not introduce additional complex settings in v1 beyond bridge URL.
- Distinguish `CONNECTED_IDLE` (ready) from `CONTEXT_LOST` (connected but requires rebind).

## 14) Implementation Boundaries

Expected additive touch points:

- `src/utils/options.ts` for `mcpBridgeUrl` default/normalization.
- `src/types/messaging.ts` for new popup/background messages and protocol event types.
- `src/entrypoints/background.ts` for WebSocket manager and operation handlers.
- `src/entrypoints/popup/App.svelte` (and possibly one small component) for connect/disconnect UI.
- i18n keys for any new user-facing labels/status text.

Implementation discipline for mergeability:

- Isolate WebSocket/session state logic in dedicated module(s) with thin integration points.
- Keep churn in existing hotspots (`background.ts`, `App.svelte`) minimal and additive.

No bridge/server/MCP backend code is added to this repository.

## 15) Acceptance Criteria

- User can connect and disconnect to local bridge from popup.
- Popup close/open does not by itself break the connection while background runtime remains alive.
- Session binding is explicit to one Teams tab and one conversation scope.
- No implicit tab reassignment when scope changes.
- Conversation enumeration works through protocol request/response.
- Snapshot stream works with pull-based chunking.
- Every frame includes session and request correlation fields.
- Terminal state is always explicit.
- `BUSY` and `CONTEXT_LOST` are emitted with the agreed semantics.
- Existing export functionality remains unchanged.
- No default persistence of snapshot data.
- Security guardrails enforce loopback-only plaintext WS.
- One active bridge connection and one active stream/request at a time.
- New user-facing text follows existing locale parity rules.
- Default v1 snapshot profile remains intentionally lean (not full export parity by default).

## 16) Rollback and Risk Control

- Feature remains isolated behind explicit Connect MCP action.
- If regressions appear, MCP connectivity can be disabled without touching export path.
- No migration of existing storage keys required; additive option only.

## 17) Deferred Items (Future, Not v1)

- Remote bridge support.
- Authentication and stronger endpoint trust model for remote scenarios.
- Multi-session parallelism and advanced routing.
- Rich telemetry dashboards or persisted protocol traces.
