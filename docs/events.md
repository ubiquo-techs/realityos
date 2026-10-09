# RealityOS — Event Reference

All events are validated against the [JSON Schema](../schema/realityos-schema.json). Validation is **soft** — failures are logged but messages are never dropped.

---

## Standard Events (`realityos.*`)

### `realityos.presence`

Fired when a person enters or leaves the phygital interaction zone.

```json
{ "name": "realityos.presence", "type": "enter", "user_id": "u1" }
{ "name": "realityos.presence", "type": "leave", "user_id": "u1" }
```

| Field | Required | Values |
|-------|----------|--------|
| `type` | Yes | `"enter"` \| `"leave"` |
| `user_id` | No | String identifier for multi-user tracking |

---

### `realityos.navigate`

Directional intent expressed physically — swipe, sweep, step direction.

```json
{ "name": "realityos.navigate", "direction": "right", "hand": "right", "user_id": "u1" }
```

| Field | Required | Values |
|-------|----------|--------|
| `direction` | Yes | `"left"` \| `"right"` \| `"up"` \| `"down"` |
| `hand` | No | `"right"` \| `"left"` |
| `user_id` | No | String |

---

### `realityos.select`

Physical confirmation of intent — the phygital equivalent of a click.

```json
{ "name": "realityos.select", "type": "grip",    "hand": "right", "user_id": "u1" }
{ "name": "realityos.select", "type": "release",  "hand": "right", "user_id": "u1" }
{ "name": "realityos.select", "type": "push",     "hand": "right", "user_id": "u1" }
{ "name": "realityos.select", "type": "pull",     "hand": "right", "user_id": "u1" }
```

| Field | Required | Values |
|-------|----------|--------|
| `type` | Yes | `"grip"` \| `"release"` \| `"push"` \| `"pull"` |
| `hand` | No | `"right"` \| `"left"` |
| `user_id` | No | String |

---

### `realityos.gesture`

Named physical gesture with additional context fields depending on type.

```json
{ "name": "realityos.gesture", "type": "zoom",  "direction": "in" }
{ "name": "realityos.gesture", "type": "zoom",  "direction": "out" }
{ "name": "realityos.gesture", "type": "kick",  "side": "left", "angle": 45.2, "distance": 312.0 }
{ "name": "realityos.gesture", "type": "jump",  "intensity": 85.0 }
{ "name": "realityos.gesture", "type": "lean",  "x": -0.3, "y": 0.1 }
{ "name": "realityos.gesture", "type": "wave",  "side": "right" }
{ "name": "realityos.gesture", "type": "point", "side": "right" }
```

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | Gesture name string |
| `direction` | For zoom | `"in"` \| `"out"` |
| `side` | For kick/wave/point | `"left"` \| `"right"` |
| `angle` | For kick | Degrees (float) |
| `distance` | For kick | Millimetres (float) |
| `intensity` | For jump | 0–100 (float) |
| `x`, `y` | For lean | -1.0–1.0 (float) |
| `user_id` | No | String |

---

### `realityos.hand.active`

Which hand is currently the active (controlling) hand.

```json
{ "name": "realityos.hand.active", "side": "right", "user_id": "u1" }
{ "name": "realityos.hand.active", "side": "none",  "user_id": "u1" }
```

| Field | Required | Values |
|-------|----------|--------|
| `side` | Yes | `"right"` \| `"left"` \| `"none"` |
| `user_id` | No | String |

---

### `realityos.hand.state`

Open/closed/pointing state of a specific hand.

```json
{ "name": "realityos.hand.state", "side": "right", "state": "open",   "user_id": "u1" }
{ "name": "realityos.hand.state", "side": "right", "state": "closed", "user_id": "u1" }
{ "name": "realityos.hand.state", "side": "left",  "state": "point",  "user_id": "u1" }
```

| Field | Required | Values |
|-------|----------|--------|
| `side` | Yes | `"right"` \| `"left"` |
| `state` | Yes | `"open"` \| `"closed"` \| `"point"` |
| `user_id` | No | String |

---

### `realityos.cursor` _(stream)_

Continuous position. `x`/`y` are normalized `0.0–1.0` regardless of sensor type.

```json
{ "name": "realityos.cursor", "x": 0.52, "y": 0.31, "user_id": "u1" }
```

| Field | Required | Values |
|-------|----------|--------|
| `x` | Yes | 0.0 (left) – 1.0 (right) |
| `y` | Yes | 0.0 (top/far) – 1.0 (bottom/near) |
| `user_id` | No | String |

> **High-frequency.** Cloud forwarding is gated by `realityos.op.stream.level` (default: `none`).

---

### `realityos.body` _(stream)_

Full skeleton frame. All joint `x`/`y` normalized 0–1; `z` in metres.

```json
{
  "name": "realityos.body",
  "skeletons": [{
    "id": "u1",
    "joints": [
      { "name": "hand_right", "x": 0.62, "y": 0.45, "z": 1.23, "state": "tracked" },
      { "name": "head",       "x": 0.50, "y": 0.15, "z": 1.20, "state": "tracked" }
    ],
    "hand_right": "open",
    "hand_left": "closed",
    "depth_mm": 1450
  }]
}
```

**joint.state**: `"tracked"` | `"inferred"` | `"not_tracked"`

**Canonical joint names**:
```
head, neck, spine_shoulder, spine_mid, spine_base,
shoulder_left,  elbow_left,  wrist_left,  hand_left,  hand_tip_left,  thumb_left,
shoulder_right, elbow_right, wrist_right, hand_right, hand_tip_right, thumb_right,
hip_left,  knee_left,  ankle_left,  foot_left,
hip_right, knee_right, ankle_right, foot_right
```

> **High-frequency.** Gated by `realityos.op.stream.level`.

---

### `realityos.face` _(stream)_

Face tracking data for a single user.

```json
{
  "name": "realityos.face",
  "user_id": "u1",
  "yaw": 12.5, "pitch": -3.2, "roll": 0.8,
  "happy": true, "engaged": false, "mouth_open": false,
  "eye_left_closed": false, "eye_right_closed": false,
  "looking_away": false, "glasses": false
}
```

> **High-frequency.** Gated by `realityos.op.stream.level`.

---

## Lights Namespace (`realityos.lights.*`)

An actuator node bridges Art-Net/DMX512 hardware. Apps send `node.action` commands; the node emits the following events back.

### `realityos.lights.discovered`

Emitted after every Art-Net scan.

```json
{
  "name":      "realityos.lights.discovered",
  "node_id":   "lights-node-name",
  "target_ip": "2.0.0.100",
  "devices": [
    {
      "ip":            "2.0.0.100",
      "short_name":    "My Art-Net Node",
      "long_name":     "My Art-Net Node (full name)",
      "mac":           "AA:BB:CC:DD:EE:FF",
      "num_ports":     1,
      "out_universes": [0]
    }
  ]
}
```

### `realityos.lights.state`

Emitted periodically and after any fixture list change.

```json
{
  "name":      "realityos.lights.state",
  "node_id":   "lights-node-name",
  "target_ip": "2.0.0.100",
  "fixtures": [
    {
      "id":      "fixture-name",
      "type":    "led-par-6ch",
      "address": 1,
      "channels": { "dim": 200, "red": 255, "green": 0, "blue": 0, "white": 0, "strobe": 0 }
    }
  ]
}
```

---

## Lights Actions (`lights.*`)

All sent as `node.action` to the lights actuator node.

| Command | Required `data` fields | Effect |
|---------|------------------------|--------|
| `lights.discover` | — | Scan Art-Net network |
| `lights.set_target` | `ip: string` | Point to a specific Art-Net device |
| `lights.set` | `fixture_id`, `channels: {dim,red,green,blue,white,strobe}` | Set fixture channels |
| `lights.preset` | `fixture_id`, `preset: string` | Apply named preset |
| `lights.blackout` | — | All channels → 0 |
| `lights.set_raw` | `address: int`, `value: int` | Set a single DMX channel by address |
| `lights.add_fixture` | `id`, `type`, `address` | Add a fixture at runtime |
| `lights.remove_fixture` | `fixture_id` | Remove a fixture |
| `lights.clear_fixtures` | — | Remove all fixtures |

**Presets**: `full_white` · `red` · `green` · `blue` · `blackout`

**Fixture types**:

| Type | Channels | Layout |
|------|----------|--------|
| `led-par-6ch` | 6 | DIM · R · G · B · W · STROBE |
| `pixel-bar-4ch` | 4 | R · G · B · W |
| `pixel-bar-8ch` | 8 | DIM · R · G · B · W · STROBE · PROG · SPEED |
| `pixel-bar-96ch` | 96 | 24 pixels × (R · G · B · W) |
| `pixel-bar-100ch` | 100 | DIM · STROBE · PROG · SPEED + 24 × (R · G · B · W) |

---

## MIDI Namespace (`realityos.midi.*`)

Emitted by a MIDI sensor node from any connected MIDI input device.

### `realityos.midi.note_on`

```json
{ "name": "realityos.midi.note_on", "channel": 1, "note": 60, "velocity": 100, "note_name": "C4", "device": "My MIDI Controller" }
```

### `realityos.midi.note_off`

```json
{ "name": "realityos.midi.note_off", "channel": 1, "note": 60, "velocity": 0, "note_name": "C4", "device": "My MIDI Controller" }
```

> Velocity-0 NoteOn messages should be normalized to `note_off` by the node.

### `realityos.midi.control_change`

```json
{ "name": "realityos.midi.control_change", "channel": 1, "control": 64, "value": 127, "device": "My MIDI Controller" }
```

Common CC numbers: `64` = sustain pedal, `1` = modulation wheel, `7` = volume.

### `realityos.midi.program_change`

```json
{ "name": "realityos.midi.program_change", "channel": 1, "program": 5, "device": "My MIDI Controller" }
```

### `realityos.midi.pitch_bend`

```json
{ "name": "realityos.midi.pitch_bend", "channel": 1, "value": 4096, "device": "My MIDI Controller" }
```

`value` range: -8192 (full down) to 8191 (full up), 0 = center.

### `realityos.midi.device_list`

```json
{ "name": "realityos.midi.device_list", "devices": [{ "index": 0, "name": "My MIDI Controller" }, { "index": 1, "name": "My Second MIDI Device" }] }
```

Emitted on startup and in response to a `list_devices` node action.

### MIDI Node Actions

| Command | Data | Effect |
|---------|------|--------|
| `set_device` | `{ "index": N }` | Switch to MIDI device N without restarting |
| `list_devices` | — | Re-emit `realityos.midi.device_list` |

---

## SoC Namespace (`realityos.soc.*`)

A bidirectional node bridges a line of simple system-on-chip hardware — buttons, distance sensors, device-local LEDs, etc. — over a device-specific discovery/connect protocol (e.g. UDP broadcast + TCP for ESP32-class devices). `soc` is deliberately hardware-agnostic: it isn't tied to any one chip family, so new device lines can emit into it unchanged.

**Not to be confused with `realityos.lights.*`.** `realityos.soc.led_level`/`realityos.soc.button_led` describe simple GPIO-driven LEDs that live *on the SoC device itself* — never Art-Net/DMX fixtures. These are different hardware with different protocols; an app or node must never bridge one namespace into the other.

Only one device is "connected" at a time per node; discovery and the typed events below are independent of which device (if any) is currently connected.

### `realityos.soc.discovered`

Emitted after every network scan. `device_model` is optional (empty when a device's discovery reply doesn't report one).

```json
{ "name": "realityos.soc.discovered", "node_id": "soc-node-name", "devices": [{ "ip": "192.168.1.50", "device_id": "BTN01", "port": 8080, "device_type": "ESP32 HD", "device_model": "" }] }
```

### `realityos.soc.connected` / `realityos.soc.connect_failed` / `realityos.soc.disconnected`

Connection lifecycle for the currently-selected device.

```json
{ "name": "realityos.soc.connected", "node_id": "soc-node-name", "device_id": "BTN01", "ip": "192.168.1.50", "device_type": "ESP32 HD", "device_model": "" }
{ "name": "realityos.soc.connect_failed", "node_id": "soc-node-name", "device_id": "BTN01", "error": "auth rejected (wrong secret?)" }
{ "name": "realityos.soc.disconnected", "node_id": "soc-node-name", "device_id": "BTN01" }
```

### `realityos.soc.line`

Raw, unparsed passthrough — one event per line the connected device sends. Always emitted regardless of device type; the fallback for any device without a dedicated translation yet.

```json
{ "name": "realityos.soc.line", "node_id": "soc-node-name", "device_id": "BTN01", "line": "BUTTON_PRESSED" }
```

### `realityos.soc.distance`

Emitted **in addition to** `realityos.soc.line` when the connected device's type reports a distance reading (e.g. an ultrasonic sensor).

```json
{ "name": "realityos.soc.distance", "node_id": "soc-node-name", "device_id": "US01", "distance_cm": 42.5 }
```

### `realityos.soc.led_level`

Emitted **in addition to** `realityos.soc.line` when the connected device's type reports the current lit-LED count on its own LED strip. This is the device's own hardware state, not an Art-Net/DMX fixture.

```json
{ "name": "realityos.soc.led_level", "node_id": "soc-node-name", "device_id": "TB01", "lit_count": 9 }
```

### `realityos.soc.buttons_list`

Emitted for button-panel devices, listing every physical button id the device reports. **There is no fixed or expected count** — `buttons` is whatever length the device itself has; nodes and apps must treat it as variable.

```json
{ "name": "realityos.soc.buttons_list", "node_id": "soc-node-name", "device_id": "BTN01", "buttons": [1, 2, 3, 4, 5, 6, 7, 8] }
```

### `realityos.soc.button_pressed`

Emitted spontaneously (not a reply to any command) when a physical button is pressed.

```json
{ "name": "realityos.soc.button_pressed", "node_id": "soc-node-name", "device_id": "BTN01", "button": 3 }
```

### `realityos.soc.button_led`

Emitted whenever a button's own LED state changes, by command or any other cause the firmware reports. This is the button's own indicator light, not an Art-Net/DMX fixture.

```json
{ "name": "realityos.soc.button_led", "node_id": "soc-node-name", "device_id": "BTN01", "button": 3, "on": true }
```

### SoC Node Actions

All sent as `node.action` to the SoC node, same pattern as `lights.*`.

| Command | Data | Effect |
|---------|------|--------|
| `esp.scan` | — | Scan for devices on the network, emits `realityos.soc.discovered` |
| `esp.select` | `ip`, `port`, `device_id`, `device_type`, `device_model` (optional) | Disconnect any current device, connect + handshake to this one |
| `esp.send` | `cmd: string` | Raw command passthrough to the connected device — contents are device/firmware-specific, not constrained by this protocol |
| `esp.disconnect` | — | Close the current device connection |

**Direction**: App → Node (`esp.*`), Node → App (`realityos.soc.*`).

---

## Operational Events (`realityos.op.*`)

### `realityos.op.stream.level`

Controls how much high-frequency stream data (cursor, body, face) is forwarded to the cloud backend. Interaction events (`presence`, `navigate`, `select`, `gesture`, `hand.*`) are always forwarded regardless.

```json
{ "name": "realityos.op.stream.level", "level": "medium" }
```

| Level | Cursor | Body | Face |
|-------|--------|------|------|
| `none` (default) | ✗ | ✗ | ✗ |
| `low` | Every 10th frame | Every 10th frame | Every 10th frame |
| `medium` | Every 5th frame | Every 5th frame | Every 5th frame |
| `high` | Every frame | Every frame | Every frame |
| `smart` | Adaptive (≈ medium) | Adaptive | Adaptive |

**Direction**: App → Hub (local).

---

## Custom Events (`realityos.custom.*`)

Developer-defined events with free-form JSON body. Name must start with `realityos.custom.`.

```json
{ "name": "realityos.custom.score_updated", "points": 150, "player": "p1" }
{ "name": "realityos.custom.zone_trigger",  "zone": "A",   "active": true }
```

**Direction**: bidirectional — apps send them to the cloud backend; the backend can send them back.

No body schema is enforced beyond the name prefix.
