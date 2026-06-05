# Extension v2 Implementation Plan (MCP)

This plan translates `EXTENSIONSv2.md` into concrete implementation steps
using the current codebase on `exporter-live-MCP`.

Primary constraint: keep changes additive and minimal to upstream-critical
paths (`background.ts`, popup MCP controls, existing scrape pipeline).

## 1) Scope for This Cycle

In scope:

- Targeted snapshot semantics for `START_SNAPSHOT` (`conversationId` handling)
- MCP-accessible diagnostic logs (`GET_LOGS` / `LOGS_RESULT`)
- Guardrailed generic pass-through fetch (`API_CALL` / `API_RESULT`)

Out of scope (parked):

- Auto-reconnect loop
- Proactive context sync push frames

## 2) Current Hooks We Reuse (No New Scraper Architecture)

### A. MCP transport and operation dispatch

- `src/entrypoints/background.ts`
  - MCP frame routing: `handleMcpSocketMessage`
  - MCP operations: `handleMcpListConversations`, `handleMcpStartSnapshot`
  - status/state: `McpConnectionState`, `getMcpStatusPayload`, `setMcpState`

### B. Conversation-targeted scraping already exists

- `src/types/shared.ts`
  - `ScrapeOptions.conversationId?: string | null`
  - `ScrapeOptions.noDomFallback?: boolean`
- `src/entrypoints/content.ts`
  - passes `conversationId` to `apiScrape(...)`
  - supports `noDomFallback` guard
- `src/content/api-client.ts`
  - `apiScrape(..., { conversationId })` prefers explicit target id

### C. Logs already captured internally

- `src/entrypoints/background.ts`
  - merged diagnostics buffer is already available for popup diagnostics
  - existing message handlers: `GET_DIAGNOSTICS_BG`, `DIAG_LOG_FORWARD`, etc.

### D. Network fetch baseline exists

- background already handles `FETCH_BLOB` and `FETCH_BLOB_DIRECT` with timeout,
  error mapping, and size checks. This gives us patterns for constrained I/O.

## 3) Design Principles for v2 Changes

- Additive-only protocol changes; do not break v1 bridge clients.
- No broad refactors in scraper/content architecture.
- Keep MCP single-operation model (`BUSY`) unchanged.
- Prefer explicit, deterministic errors over implicit fallback behavior.
- Reuse existing options/types/storage model; avoid new persistent state unless
  unavoidable.

## 4) Protocol Changes

Keep protocol id at v1 for compatibility initially, while introducing optional
new frame types that v1 clients can ignore safely. Move to `/v2` only if a
breaking semantic change is required.

New/updated frames:

- Request: `START_SNAPSHOT` payload may include `conversationId`.
- Response on mismatch: `ERROR` with `{ code: 'CONTEXT_MISMATCH' }`.
- Request: `GET_LOGS` payload `{ limit?: number, levels?: string[] }`.
- Response: `LOGS_RESULT` payload `{ entries: [...] }`.
- Request: `API_CALL` payload:
  - `method`
  - `endpoint` (relative path only)
  - optional `query`
  - optional `body`
- Response: `API_RESULT` payload:
  - `status`
  - `data` (parsed JSON when possible)
  - `error` (structured)

## 5) Workstream A: `START_SNAPSHOT` Target Semantics

### Goal

Prevent silent mismatch between requested conversation and streamed result.

### Minimal implementation sequence

1. Parse optional `payload.conversationId` in `handleMcpStartSnapshot`.
2. If absent, keep current behavior (bound conversation).
3. If present and different from bound conversation:
   - return `ERROR` `CONTEXT_MISMATCH` (first step), OR
   - if enabled by flag, run scrape using requested id.
4. When scraping by explicit requested id, set `noDomFallback: true` so API
   failure does not scrape active UI chat by accident.
5. Ensure `SNAPSHOT_STARTED` echoes the actual conversation id used.

### Why this is minimal

- Uses existing `conversationId` hook already threaded to API scrape.
- Avoids any UI tab switching, DOM navigation, or new content-script channels.
- Keeps MCP operation model intact.

### Edge cases

- Empty or non-string requested id -> treat as absent.
- Unknown/unavailable conversation -> return existing scrape error mapping.
- Cancel path remains unchanged.

## 6) Workstream B: `GET_LOGS` over MCP

### Goal

Expose extension-side diagnostics to bridge clients without opening DevTools.

### Minimal implementation sequence

1. Extend MCP frame router to accept `GET_LOGS`.
2. Reuse existing in-memory diagnostic buffer as the data source.
3. Apply a strict cap:
   - default limit (e.g., 100)
   - hard max (e.g., 500)
4. Optionally filter by level when provided.
5. Emit `LOGS_RESULT` with a compact entry shape:
   - `ts`, `level`, `src`, `line`
6. Return `ERROR` on malformed payload only; otherwise tolerate missing fields.

### Security and privacy guardrails

- Do not include storage snapshots or environment dumps in this frame.
- Keep to recent line entries only.
- Optionally redact obvious bearer/token substrings before emission.

## 7) Workstream C: Guardrailed `API_CALL`

### Goal

Allow bridge evolution without adding one hardcoded MCP operation per endpoint.

### Minimal implementation sequence

1. Add `API_CALL` handler in background MCP router.
2. Validate payload strictly:
   - `method` initially `GET` only
   - `endpoint` must be relative (no absolute URL)
3. Resolve endpoint against allowlisted base(s) already known in runtime
   context (Teams chat service origin discovered by existing flow).
4. Execute fetch with:
   - timeout
   - response size cap
   - JSON parse best-effort (fallback to text)
5. Return `API_RESULT` with normalized error details.

### Required constraints (non-negotiable)

- Host allowlist, not user-provided arbitrary domains.
- Endpoint-prefix allowlist, not free-form pathing.
- No custom auth headers accepted from bridge payload.
- Concurrency remains single active MCP operation.

### Rollout strategy

- Stage 1: read-only endpoints needed by bridge today.
- Stage 2: expand allowlist only when a concrete use case is validated.

## 8) Files Expected to Change (Minimal Touch Set)

- `src/entrypoints/background.ts`
  - MCP frame router + operation handlers
  - START_SNAPSHOT target semantics
  - `GET_LOGS` and `API_CALL` handlers
- `src/types/messaging.ts`
  - MCP frame payload/result typing updates where applicable
- `WEBSOCKET.md` (optional)
  - index update if new MCP frames are documented locally
- `EXTENSIONSv2.md`
  - scope/parking decisions (already updated)

Avoid changes unless required:

- `src/entrypoints/content.ts`
- `src/content/api-client.ts`
- popup UI components

## 9) Acceptance Criteria

### A. Snapshot target correctness

- A request with `conversationId != boundConversationId` never silently returns
  a different chat.
- If mismatch mode is reject-first, returns `ERROR` `CONTEXT_MISMATCH`.

### B. Logs over MCP

- `GET_LOGS` returns bounded recent logs with stable schema.
- Invalid payload does not crash socket handler.

### C. API pass-through safety

- Non-allowlisted targets are rejected.
- Large/slow responses return structured timeout/size errors.
- No arbitrary header injection from bridge payload.

## 10) Verification Plan

Manual protocol checks (bridge-side):

1. Connect MCP and send `START_SNAPSHOT` with a different `conversationId`.
2. Confirm deterministic mismatch behavior.
3. Send `GET_LOGS` and verify capped entries.
4. Send valid and invalid `API_CALL` requests.
5. Confirm `BUSY` gate still prevents parallel operations.

Repo checks after code changes:

- `pnpm check`
- `pnpm build`
- `pnpm build:firefox`

## 11) Defer List (Explicit)

Still parked after this plan:

- socket auto-reconnect loop
- context-change push frames (`CONTEXT_UPDATED`)
- operation queueing and pull-iterator flow

These remain parked until we intentionally take on lifecycle/state-machine
complexity in a dedicated reliability phase.
