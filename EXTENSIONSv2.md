# Extension v2 Recommendations & Discovered Issues

This document captures MCP-extension gaps discovered after shipping v1 on
`exporter-live-MCP`.

Implementation delta summary for this branch is tracked in
`EXTENSION_MCP_IMPROVEMENTS.md`.

Decision for the next cycle: keep v2 additive and minimal, and avoid broad
state-machine churn in upstream-critical paths.

## Scope Decision

In this cycle, we actively pursue:

- #1 Targeted snapshot semantics (`START_SNAPSHOT` + explicit `conversationId`)
- #4 MCP-level log retrieval (`GET_LOGS` / `LOGS_RESULT`)
- #5 Guardrailed generic API pass-through (`API_CALL` / `API_RESULT`)

We intentionally park for now:

- #2 Auto-reconnect loop
- #3 Proactive context sync push frames

Parking rationale: both require persistent lifecycle and state-sync complexity
that conflicts with our current "minimal upstream change" policy for MCP v2.

## 1. Ignored `conversationId` during `START_SNAPSHOT`

**Current behavior (code-accurate):**

- `START_SNAPSHOT` payload `conversationId` is currently ignored.
- Snapshot execution uses the MCP-bound conversation established on connect.

This causes a protocol mismatch: the bridge can request conversation X and
still receive data for bound conversation Y.

**v2 recommendation:**

- Honor payload `conversationId` when provided, using the existing API scrape
  hooks that already accept explicit `conversationId`, OR
- Reject with `ERROR` + `code: CONTEXT_MISMATCH` when requested and bound scope
  differ.

Preference for minimal churn: start with explicit reject-on-mismatch semantics,
then add direct targeting once fully verified.

## 2. Lack of Auto-Reconnect Mechanism (Parked)

**Status:** Parked for this cycle.

**Reason:** reconnect introduces a separate lifecycle state machine, retry
backoff policy, and coupling with scope validity and popup state. That is a
larger behavioral change than we want in the next upstream-aligned step.

## 3. Lack of Proactive Context Sync (Parked)

**Status:** Parked for this cycle.

**Reason:** push-based context updates (`CONTEXT_UPDATED`) require durable chat
change detection, dedupe, and race handling across popup/background/content.
We keep lazy validation-on-operation for now.

## 4. MCP-Level Internal Log Retrieval for Troubleshooting

**Current behavior:**

- The extension already maintains diagnostics logs internally.
- MCP protocol does not expose those logs directly.

**v2 recommendation:**

- Add `GET_LOGS` request with capped `limit`.
- Return `LOGS_RESULT` with timestamp, severity, and message fields.
- Keep payload bounded and sanitize obvious sensitive artifacts.

## 5. Generic API Call Pass-Through

**Current behavior:**

- MCP v1 exposes only fixed operations (`LIST_CONVERSATIONS`,
  `START_SNAPSHOT`).

**v2 recommendation:**

- Add `API_CALL` / `API_RESULT` for bridge-driven fetches through extension
  auth context.
- Keep it tightly constrained in v2:
  - allowlist hosts and endpoint prefixes,
  - method restrictions (start with `GET`),
  - timeout and response-size caps,
  - explicit structured errors.

This preserves flexibility while staying compatible with the minimal-change
policy.
