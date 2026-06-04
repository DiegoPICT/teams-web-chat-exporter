# WebSocket Parked Items (Post-v1)

These items were discussed and are intentionally deferred to keep v1 simple, implementable, and upstream-friendly.

## 1) Internal Queueing and Queue Draining

- Parked because v1 is single-operation only.
- v1 behavior is a busy gate (`BUSY`) rather than queued execution.

## 2) Application-Level Ping/Pong

- Parked because v1 is local loopback and can rely on native WebSocket lifecycle events.
- v1 uses socket open/close/error as transport truth.

## 3) Automatic Reconnect Loop

- Parked to avoid reconnect state-machine complexity in v1.
- v1 reconnect is explicit user action from popup.

## 4) Layered Custom Timeouts

- Parked to avoid brittle overlap with upstream scrape and transport error handling.
- v1 relies on existing upstream operation behavior + native transport lifecycle.

## 5) Pull-Based `NEXT` Iterator Flow

- Parked because current upstream scrape path yields a full snapshot before iteration.
- v1 uses extension push-streaming chunks from in-memory snapshot.

## Revisit Triggers

Re-open parked items only when one or more is true:

- measured reliability issue in v1,
- measured memory/latency issue requiring backpressure,
- explicit upstream review feedback requesting the capability,
- scope expansion beyond v1 single-operation model.
