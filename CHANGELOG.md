# Changelog

## v1.2.0 — 2026-10-09

**Added the SoC namespace** (`realityos.soc.*`) — previously implemented by nodes but never formalized in the schema, so every SoC event failed validation (soft/non-blocking, but noisy):

- `realityos.soc.discovered`, `connected`, `connect_failed`, `disconnected`, `line` — device discovery/connection lifecycle and raw line passthrough
- `realityos.soc.distance` — typed distance reading
- `realityos.soc.led_level` — typed lit-LED-count reading for a device-local LED strip
- `realityos.soc.buttons_list` — list of a button-panel device's physical button ids (no fixed/expected count — devices report their own cardinality)
- `realityos.soc.button_pressed` — spontaneous physical button press
- `realityos.soc.button_led` — a button's own LED state changed
- Added `device_model` (optional) to `soc_connected`

**Note**: `realityos.soc.led_level`/`button_led` describe GPIO LEDs living on the SoC device itself — unrelated to `realityos.lights.*` (Art-Net/DMX). Don't conflate the two namespaces.

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
