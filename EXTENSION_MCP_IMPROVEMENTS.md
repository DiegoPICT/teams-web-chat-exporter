# Extension MCP Improvements (Branch Delta)

Focused summary of MCP extension improvements delivered on
`exporter-live-MCP`, relative to the baseline commit we started from:

- baseline: `6e2bfda` (`feat(mcp): add v1 extension websocket client controls`)

This is a downstream branch update only. It is intentionally additive and
minimal, and does not require rewriting upstream architecture.

## What Improved

### 1) Runtime Health + Authoritative Version Reporting

- Added MCP `HEALTH` request handling with `HEALTH_RESULT` response.
- Health/status payload now includes:
  - `protocol`
  - `extensionVersion` (from `runtime.getManifest().version`)
- `HELLO` payload also includes `extensionVersion` for handshake-level
  verification.

Why it matters: bridge can confirm the exact loaded extension runtime instead of
inferring from local build assumptions.

### 2) Extension Internal Log Access over MCP

- Added MCP `GET_LOGS` -> `LOGS_RESULT`.
- Returns bounded recent extension logs with:
  - `ts`, `src`, `level`, `line`
- Supports `limit` and optional `levels` filtering.
- Includes conservative redaction for obvious token-like artifacts.

Why it matters: bridge now has extension-internal visibility (background and
content) in addition to transport-level bridge logs.

### 3) Deterministic `START_SNAPSHOT` Targeting

- `START_SNAPSHOT` now supports two deterministic modes:
  - targeted: `payload.conversationId` provided
  - active: `payload.conversationId` omitted (resolve current GUI chat)
- Targeted mode sets `noDomFallback: true` to prevent silent drift to active
  chat when API fetch fails.
- `SNAPSHOT_STARTED` now reflects the effective chat (`conversationId` /
  `conversationTitle`) used for the run.
- Added explicit not-found handling path (`NOT_FOUND`) for targeted requests.

Why it matters: bridge can trust snapshot labeling and avoid exporting the wrong
conversation silently.

### 4) Generic Teams API Transaction

- Added MCP `API_CALL` request handling with `API_RESULT` response.
- Supports multi-method calls (not limited to GET-only).
- Returns normalized shape:
  - `status`
  - `data`
  - `error`
- Extension-side rails kept basic by design:
  - payload validation
  - timeout cap
  - response-size cap
  - deterministic error mapping

Why it matters: bridge can evolve without adding one bespoke MCP frame per new
Teams endpoint need.

## Design Constraints Preserved

- Single active MCP operation (`BUSY`) remains unchanged.
- No reconnect state-machine work introduced.
- No proactive context-push event system introduced.
- No popup UX redesign required.

## Still Parked (Intentional)

- Auto-reconnect reliability loop.
- Proactive GUI context sync frames.
- Queueing / iterator transport redesign.

These remain separate reliability/lifecycle follow-up work.

## Main Files Touched

- `src/entrypoints/background.ts`
- `src/entrypoints/content.ts`
- `src/content/api-client.ts`
- `src/types/messaging.ts`
- `package.json` / `wxt.config.ts` (version update already landed)
