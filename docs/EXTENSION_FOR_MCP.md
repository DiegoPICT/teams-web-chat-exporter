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

## Known issues

- `CANCEL` semantics are currently snapshot-centric. `START_SNAPSHOT` is
  cancelled explicitly, but non-snapshot operations (including `API_CALL`) may
  emit a generic cancelled terminal signal while the in-flight operation can
  still return a result.
- Active snapshot mode can report a stale `conversationTitle` when the user has
  switched chats since initial MCP connect.
- `API_CALL` currently rejects absolute `http(s)://` endpoints, but a protocol-
  relative endpoint (`//host/path`) can still override host unless explicitly
  blocked.
- `API_CALL` is generic in frame shape but currently resolves against chat
  service discovery/auth flow. It is not yet a multi-service Teams surface.
- Error taxonomy is deterministic but split between transport-level `ERROR`
  codes and `API_RESULT.error.code` values.
- Targeted-conversation precheck (`LIST_CONVERSATIONS_QUICK`) can classify
  transient data/read failures as `NOT_FOUND`.
- MCP UI keys currently live in `en.json`; if locale parity is enforced for this
  feature surface, non-English locale files need synchronized keys.

## TODO

- Normalize `CANCEL` behavior across MCP operations, including deterministic
  cancel handling for `API_CALL`.
- Resolve active-mode snapshot titles from current GUI state instead of relying
  on connect-time title state.
- Explicitly reject protocol-relative `API_CALL` endpoints (`//...`) and keep
  endpoint normalization host-stable.
- Make `API_CALL` service scope explicit in contract/docs, and extend to other
  Teams service families only when intentionally designed.
- Consolidate error-code vocabulary so bridge handling is simpler and fully
  deterministic across frame-level and payload-level failures.
- Split targeted precheck failures into distinguishable outcomes (not-found vs
  transient read/runtime failure).
- Decide and apply locale policy for MCP UI keys (en-only by exception, or full
  locale parity).

## Parked items

The following remain intentionally out of scope in the current implementation:

- Auto-reconnect state machine.
- Proactive GUI context push events.
- Queueing / iterator transport redesign.
