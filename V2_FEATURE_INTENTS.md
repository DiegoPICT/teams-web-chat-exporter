# v2 Feature Intents (Remaining Scope)

This document defines the objective and specific intent of the two remaining v2 features.

It is intentionally explicit about behavior, determinism, and error semantics so bridge and extension implementations can converge without ambiguity.

## Feature 1: Deterministic Chat Targeting for Message Iteration

### Objective

The bridge can already request the extension to enumerate all existing chats. This needs to be a canonical transaction.
The transaction that exports the chat should support two mechanisms, enumerating the currently-gui selected chat, but also when specified iterate messages by specifying any one of the discovered chats, regardless of the selected chat in the gui.

### Specific Intent

- `LIST_CONVERSATIONS` remains the canonical inventory transaction.
- `START_SNAPSHOT` must support two deterministic modes:
  - **Targeted mode**: `payload.conversationId` is provided. The extension must iterate messages for that exact chat id.
  - **Active mode**: `payload.conversationId` is omitted. The extension must iterate messages for the currently GUI-selected chat.
- The extension must never silently return messages for a different chat than requested in targeted mode.
- `SNAPSHOT_STARTED.payload.conversationId` and `conversationTitle` are the source of truth for what actually started.

### Determinism Rules

- If targeted mode is requested and chat id is valid/reachable, start and stream that chat.
- If targeted mode is requested and chat id is invalid/unreachable/nonexistent, return terminal `ERROR` with clear code/message and stop.
- If active mode is requested, resolve only the current GUI-selected chat and report that id/title in `SNAPSHOT_STARTED`.
- No silent fallback from targeted mode to active mode.

### Error Semantics (minimum)

- `NOT_FOUND` (or equivalent): requested `conversationId` does not exist.
- `CONTEXT_LOST`: bound context is no longer valid.
- `BUSY`: concurrent operation attempted.
- `UNSUPPORTED`: frame or parameter not supported by runtime.

### Acceptance Criteria

- Chat inventory count is stable and reproducible for a given connected session.
- Targeted mode returns `SNAPSHOT_STARTED.conversationId == requested conversationId` or explicit terminal error.
- Active mode always reports and streams the GUI-selected conversation.
- Bridge can trust `SNAPSHOT_STARTED` to label exports and downstream summaries.

## Feature 2: Generic Teams API Transaction (`API_CALL` / `API_RESULT`)

### Objective

Expose a controlled general-purpose transaction so the bridge can ask the extension to execute specific Teams API calls and return raw result data, status, and/or error details.

### Specific Intent

- Introduce `API_CALL` request and `API_RESULT` response frames.
- Keep bridge logic generic while extension performs authenticated in-browser API calls.
- Reduce need to add one bespoke protocol command per new data need.

### Transaction Contract (minimum)

- **Request**: `API_CALL`
  - `method` (start with `GET`; expand later if needed)
  - `endpoint` (relative/allowlisted path)
  - optional `query`
  - optional `body`
- **Response**: `API_RESULT`
  - `status` (HTTP-like numeric status when available)
  - `data` (raw object/array/text)
  - `error` (structured details when call fails)

### Guardrails

- Allowlist hosts/origins and endpoint prefixes.
- Timeout cap and response size cap. For simplicity not immediately but in the future; deterministic, controlled iterative paging for potentially large scopes.
- Preserve request correlation with `requestId`.

### Acceptance Criteria

- For an allowed valid request, extension returns deterministic `API_RESULT` with status and raw payload.
- For denied/invalid request, extension returns deterministic `ERROR`/`API_RESULT.error` with reason.
- Bridge can surface extension-side API failures without losing fidelity.

## Out of Scope for This Document

- Auto-reconnect policy and runtime lifecycle tuning.
- Proactive GUI context push events.
- Paging on new transactions

Those are tracked separately as reliability/lifecycle follow-up work.
