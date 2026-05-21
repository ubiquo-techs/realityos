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

**Sources**: Kinect v2 (skeleton enter/leave), LiDAR (cluster detection timeout).

---

### `realityos.navigate`

Directional intent expressed physically — swipe, sweep, kick direction.

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

Physical confirmation of intent — the phygital equivalent of a mouse click.

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

> **High-frequency.** Xspace forwarding is gated by `realityos.op.stream.level` (default: `none`).

**Sources**: Kinect v2 (active hand position normalized from 640×480), LiDAR (cluster centroid normalized from configured zone).

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

The DMX node bridges Art-Net/DMX512 hardware. It is an **actuator**: apps send `node.action` commands (see Actions section), and the node emits the following events back.

### `realityos.lights.discovered`

Emitted after every Art-Net scan.

```json
{
  "name":      "realityos.lights.discovered",
  "node_id":   "dmx-01",
  "target_ip": "2.0.0.100",
  "devices": [
    {
      "ip":            "2.0.0.100",
      "short_name":    "ODE Mk2",
      "long_name":     "Enttec ODE Mk2 Node",
      "mac":           "AA:BB:CC:DD:EE:FF",
      "num_ports":     1,
      "out_universes": [0]
    }
  ]
}
```

### `realityos.lights.state`

Emitted every 10 s and after any fixture list change.

```json
{
  "name":      "realityos.lights.state",
  "node_id":   "dmx-01",
  "target_ip": "2.0.0.100",
  "fixtures": [
    {
      "id":      "par-01",
      "type":    "led-par-6ch",
      "address": 1,
      "channels": { "dim": 200, "red": 255, "green": 0, "blue": 0, "white": 0, "strobe": 0 }
    }
  ]
}
```

---

## Lights Actions (`lights.*`)

All sent as `node.action` to `dmx-01` (or the configured node ID).

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

Emitted by the MIDI sensor node from any connected MIDI input device.

### `realityos.midi.note_on`

```json
{ "name": "realityos.midi.note_on", "channel": 1, "note": 60, "velocity": 100, "note_name": "C4", "device": "KeyLab 61" }
```

### `realityos.midi.note_off`

```json
{ "name": "realityos.midi.note_off", "channel": 1, "note": 60, "velocity": 0, "note_name": "C4", "device": "KeyLab 61" }
```

> Velocity-0 NoteOn messages are normalized to `note_off` by the MIDI node.

### `realityos.midi.control_change`

```json
{ "name": "realityos.midi.control_change", "channel": 1, "control": 64, "value": 127, "device": "KeyLab 61" }
```

Common CC numbers: `64` = sustain pedal, `1` = modulation wheel, `7` = volume.

### `realityos.midi.program_change`

```json
{ "name": "realityos.midi.program_change", "channel": 1, "program": 5, "device": "KeyLab 61" }
```

### `realityos.midi.pitch_bend`

```json
{ "name": "realityos.midi.pitch_bend", "channel": 1, "value": 4096, "device": "KeyLab 61" }
```

`value` range: -8192 (full down) to 8191 (full up), 0 = center.

### `realityos.midi.device_list`

```json
{ "name": "realityos.midi.device_list", "devices": [{ "index": 0, "name": "KeyLab 61" }, { "index": 1, "name": "USB MIDI Interface" }] }
```

Emitted on startup and in response to a `list_devices` node action.

### MIDI Node Actions

| Command | Data | Effect |
|---------|------|--------|
| `set_device` | `{ "index": N }` | Hot-swap to MIDI device N without restarting |
| `list_devices` | — | Re-emit `realityos.midi.device_list` |

---

## Operational Events (`realityos.op.*`)

### `realityos.op.stream.level`

Controls how much high-frequency stream data (cursor, body, face) is forwarded to xspace (cloud). Interaction events are always forwarded regardless of this setting.

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

**Direction**: App → Player (local). Sent via `uboxclient.js`:
```js
ubox.setStreamLevel('high');
```

---

## Custom Events (`realityos.custom.*`)

Developer-defined events with free-form JSON body. Name must start with `realityos.custom.`.

```json
{ "name": "realityos.custom.score_updated", "points": 150, "player": "p1" }
{ "name": "realityos.custom.zone_trigger",  "zone": "A",   "active": true }
```

**Direction**: bidirectional — apps send them to xspace; xspace can send them back.

No body schema is enforced beyond the name prefix.

---

## Kinect v2 → RealityOS Translation Table

The hub translates these legacy event names automatically. Both the original and translated events are delivered to browser apps.

| Kinect name | Canonical event | Fields added |
|-------------|----------------|--------------|
| `RIGHT` | `realityos.navigate` | direction=right, hand=right |
| `LEFT` | `realityos.navigate` | direction=left, hand=right |
| `LRIGHT` | `realityos.navigate` | direction=right, hand=left |
| `RLEFT` | `realityos.navigate` | direction=left, hand=left |
| `GRIP` | `realityos.select` | type=grip |
| `RELEASE` | `realityos.select` | type=release |
| `CLICKDOWN` | `realityos.select` | type=push |
| `CLICKUP` | `realityos.select` | type=pull |
| `ZOOM_IN` | `realityos.gesture` | type=zoom, direction=in |
| `ZOOM_OUT` | `realityos.gesture` | type=zoom, direction=out |
| `POINT` | `realityos.gesture` | type=point, side=right |
| `Wave` | `realityos.gesture` | type=wave, side=right |
| `Kick` | `realityos.gesture` | type=kick (angle, side, distance passed through) |
| `Jump` | `realityos.gesture` | type=jump (intensity passed through) |
| `Lean` | `realityos.gesture` | type=lean (x, y passed through) |
| `NewUser` | `realityos.presence` | type=enter |
| `NoUser` / `UserLeft` | `realityos.presence` | type=leave |
| `ChangedHandRight` | `realityos.hand.active` | side=right |
| `ChangedHandLeft` | `realityos.hand.active` | side=left |
| `ChangedHandNone` | `realityos.hand.active` | side=none |
| `active_hand` | `realityos.cursor` | x/y normalized from 640×480 to 0–1 |
| `skeleton` | `realityos.body` | name changed; skeletons passed through |
| `face` | `realityos.face` | name changed; all fields passed through |
