# Extension for MCP

This document describes the extension-side MCP behavior currently implemented in
this repository.

It focuses on end-state behavior: what the extension does, how key transactions
work, and which reliability items remain intentionally parked.

## Scope and intent

The extension provides a local WebSocket client that connects to a bridge and
exposes deterministic MCP transactions for Teams data access.

Current scope includes:

- MCP connect/disconnect/status control from popup through background.
- Conversation inventory (`LIST_CONVERSATIONS`).
- Snapshot streaming (`START_SNAPSHOT` + `CHUNK` + `DONE`) with deterministic
  targeting modes.
- Extension health reporting (`HEALTH` / `HEALTH_RESULT`).
- Extension internal log retrieval (`GET_LOGS` / `LOGS_RESULT`).
- Generic Teams API transaction (`API_CALL` / `API_RESULT`).

## Runtime ownership model

### Popup context

- Sends user intents to background (`MCP_CONNECT`, `MCP_DISCONNECT`,
  `MCP_STATUS`).
- Renders MCP state updates (`MCP_STATUS_UPDATE`).
- Does not own WebSocket transport.

### Background context

- Owns WebSocket lifecycle and MCP frame routing.
- Owns single-operation busy gate (`BUSY`).
- Owns bound Teams tab scope.
- Orchestrates scrape execution and stream framing.

### Content context

- Executes page-scoped Teams operations when requested by background.
- Runs API scraping and generic API transaction execution under Teams auth
  context.

## Health and version reporting

The extension supports `HEALTH` and replies with `HEALTH_RESULT`.

Health/status payload includes:

- `state`, `bridgeUrl`, `connected`
- `protocol`
- `extensionVersion` (authoritative, from extension manifest)
- `sessionId`, `tabId`, `conversationId`, `conversationTitle`, `lastError`

`HELLO` payload also includes `protocol` and `extensionVersion` so bridge can
verify runtime identity at handshake time.

## Logs transaction

`GET_LOGS` returns `LOGS_RESULT` with bounded recent entries.

Entry shape:

- `ts`
- `src` (`bg` or `content`)
- `level`
- `line`

Supported request controls:

- `limit` (bounded)
- optional `levels` filter

The returned lines apply conservative redaction for obvious token-like patterns.

## Deterministic snapshot targeting

`START_SNAPSHOT` supports two deterministic modes.

### Targeted mode

- Input: `payload.conversationId` provided.
- Behavior: snapshot iterates that exact chat id.
- Safety: targeted mode uses `noDomFallback` so API failure does not silently
  scrape the currently visible GUI chat.

### Active mode

- Input: `payload.conversationId` omitted.
- Behavior: snapshot resolves and iterates the currently selected GUI chat.

`SNAPSHOT_STARTED.payload.conversationId` and `conversationTitle` are the
source of truth for what actually started.

No silent targeted-to-active fallback is allowed.

## Generic API transaction

The extension supports `API_CALL` and replies with `API_RESULT`.

Request contract:

- `method`
- `endpoint`
- optional `query`
- optional `body`

Response contract:

- `status`
- `data`
- `error`

Extension-side rails are intentionally basic (bridge remains the primary
policy layer): payload validation, timeout cap, response-size cap, and stable
error mapping.

## Error semantics

Core deterministic error codes used by extension-side MCP flow:

- `BUSY` for concurrent operation attempts.
- `CONTEXT_LOST` when bound Teams tab scope is no longer valid.
- `NOT_FOUND` for targeted snapshot requests whose conversation cannot be
  resolved.
- `UNSUPPORTED` for unsupported frame types or invalid parameter shapes.

## Parked items

The following remain intentionally out of scope in the current implementation:

- Auto-reconnect state machine.
- Proactive GUI context push events.
- Queueing / iterator transport redesign.
