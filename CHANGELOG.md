# Changelog

## v1.1.0 — 2026-05-21

**Schema bug fixes** — correcting divergences between the schema and the actual runtime behavior of the Ubox Physical Player hub:

- `realityos.navigate`: `direction` enum corrected from `"next"|"prev"` to `"left"|"right"|"up"|"down"`
- `realityos.select`: required field renamed from `state` to `type`; values corrected from `"start"|"end"` to `"grip"|"release"|"push"|"pull"`
- `realityos.gesture`: required field renamed from `gesture` to `type`
- `realityos.op.stream.level`: `level` field type changed from `number (0–1)` to `string enum: "none"|"low"|"medium"|"high"|"smart"`
- `realityos.hand.state`: `hand` field renamed to `side`; `state` enum updated from `"open"|"closed"|"lasso"` to `"open"|"closed"|"point"`
- Added `$id` pointing to the canonical published URL
- Added `x-version` metadata field

## v1.0.0 — 2025-01-01

Initial schema covering: presence, navigate, select, gesture, hand.active, hand.state, cursor, body, face, lights.discovered, lights.state, op.stream.level, midi.*, custom.*
