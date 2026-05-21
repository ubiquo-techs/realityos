# RealityOS — App Developer Reference

> **Source of truth**: `docs/realityos-protocol.md` (protocol spec) and `core/dashboard/uboxclient.js` (browser API).
> Update this file whenever either changes.

---

## Overview

RealityOS is the phygital event layer. Sensors (Kinect, LiDAR, MIDI keyboard, button, …) publish structured events to the local hub. Your browser app subscribes through `uboxclient.js`. Actuators (DMX lights) receive commands the same way — through the hub — via `node.action` messages.

All events arrive as JSON over WebSocket. `uboxclient.js` parses and dispatches them so you only deal with plain JavaScript callbacks.

---

## Quick Start

```html
<!DOCTYPE html>
<html>
<head><title>My Ubox App</title></head>
<body>

<!-- 1. Load ubox bridge (served by the player at this exact URL) -->
<script src="http://127.0.0.1:8080/uboxclient.js"></script>

<!-- 2. Your app logic -->
<script>
  ubox.onPresence(function (type, userId) {
    if (type === 'enter') document.body.style.background = '#111';
    if (type === 'leave') document.body.style.background = '#000';
  });

  ubox.onNavigate(function (direction, hand, userId) {
    console.log('navigate', direction, hand);
  });
</script>
</body>
</html>
```

`uboxclient.js` auto-connects to `ws://127.0.0.1:8181/KinectHtml5` on load. No manual setup needed.

---

## Standard Events (`realityos.*`)

### Presence — `ubox.onPresence(fn)`

Fired when someone enters or leaves the interaction zone.

```js
ubox.onPresence(function (type, userId) {
  // type: 'enter' | 'leave'
  // userId: string | undefined
});
```

**Sources**: Kinect v2 (skeleton tracking), LiDAR (cluster detection).

**Wire payload**:
```json
{ "name": "realityos.presence", "type": "enter", "user_id": "u1" }
```

---

### Navigate — `ubox.onNavigate(fn)`

Directional intent — swipe, sweep, kick direction.

```js
ubox.onNavigate(function (direction, hand, userId) {
  // direction: 'left' | 'right' | 'up' | 'down'
  // hand: 'right' | 'left'
  // userId: string | undefined
  if (direction === 'right') showNextSlide();
  if (direction === 'left')  showPrevSlide();
});
```

**Wire payload**:
```json
{ "name": "realityos.navigate", "direction": "right", "hand": "right", "user_id": "u1" }
```

---

### Select — `ubox.onSelect(fn)`

Physical confirmation of intent — grip, push, pull (phygital click).

```js
ubox.onSelect(function (type, hand, userId) {
  // type: 'grip' | 'release' | 'push' | 'pull'
  if (type === 'grip') confirmSelection();
});
```

**Wire payload**:
```json
{ "name": "realityos.select", "type": "grip", "hand": "right", "user_id": "u1" }
```

---

### Gesture — `ubox.onGesture(fn)`

Named physical gesture with additional context.

```js
ubox.onGesture(function (type, data, userId) {
  // type: 'zoom' | 'kick' | 'jump' | 'lean' | 'wave' | 'point'
  // data: full event payload (type-specific fields below)
  switch (type) {
    case 'zoom':  handleZoom(data.direction);         break; // direction: 'in'|'out'
    case 'kick':  handleKick(data.side, data.angle);  break; // side: 'left'|'right', angle: degrees, distance: mm
    case 'jump':  handleJump(data.intensity);         break; // intensity: 0–100
    case 'lean':  handleLean(data.x, data.y);         break; // x/y: -1.0..1.0
    case 'wave':  handleWave(data.side);              break; // side: 'left'|'right'
    case 'point': handlePoint(data.side);             break; // side: 'left'|'right'
  }
});
```

**Wire payloads**:
```json
{ "name": "realityos.gesture", "type": "zoom",  "direction": "in" }
{ "name": "realityos.gesture", "type": "kick",  "side": "left", "angle": 45.2, "distance": 312.0 }
{ "name": "realityos.gesture", "type": "jump",  "intensity": 85.0 }
{ "name": "realityos.gesture", "type": "lean",  "x": -0.3, "y": 0.1 }
{ "name": "realityos.gesture", "type": "wave",  "side": "right" }
{ "name": "realityos.gesture", "type": "point", "side": "right" }
```

---

### Cursor — `ubox.onCursor(fn)`

Continuous position stream. `x`/`y` are normalized `0.0–1.0` regardless of sensor type.

```js
ubox.onCursor(function (x, y, userId) {
  // x: 0.0 (left) – 1.0 (right)
  // y: 0.0 (top/far) – 1.0 (bottom/near)  ← axis varies by sensor
  moveCursor(x * window.innerWidth, y * window.innerHeight);
});
```

> **Stream level**: cursor is high-frequency. By default it is not forwarded to xspace. Call `ubox.setStreamLevel('medium')` to enable cloud forwarding.

**Sources**: Kinect v2 (active hand → cursor), LiDAR (cluster centroid).

**Wire payload**:
```json
{ "name": "realityos.cursor", "x": 0.52, "y": 0.31, "user_id": "u1" }
```

---

### Body (skeleton stream) — `ubox.onBody(fn)`

Full skeleton data per frame.

```js
ubox.onBody(function (skeletons) {
  // skeletons: array of skeleton objects
  skeletons.forEach(function (sk) {
    var handR = sk.joints.find(j => j.name === 'hand_right');
    if (handR && handR.state === 'tracked') {
      drawHand(handR.x, handR.y); // normalized 0–1
    }
  });
});
```

**Skeleton object**:
```json
{
  "id": "u1",
  "joints": [
    { "name": "hand_right", "x": 0.62, "y": 0.45, "z": 1.23, "state": "tracked" }
  ],
  "hand_right": "open",
  "hand_left": "closed",
  "right_hand_raised": 0,
  "left_hand_raised": 0,
  "depth_mm": 1450
}
```

**Canonical joint names** (x/y: 0–1, z: metres):
```
head, neck, spine_shoulder, spine_mid, spine_base,
shoulder_left,  elbow_left,  wrist_left,  hand_left,  hand_tip_left,  thumb_left,
shoulder_right, elbow_right, wrist_right, hand_right, hand_tip_right, thumb_right,
hip_left,  knee_left,  ankle_left,  foot_left,
hip_right, knee_right, ankle_right, foot_right
```

**joint.state**: `tracked | inferred | not_tracked`

---

### Face (stream) — `ubox.onFace(fn)`

```js
ubox.onFace(function (data) {
  if (data.happy) showSmileEffect();
  console.log('yaw', data.yaw, 'pitch', data.pitch);
});
```

**Wire payload**:
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

---

### Hand Active — `ubox.onHandActive(fn)`

Which hand is currently the controlling hand.

```js
ubox.onHandActive(function (side, userId) {
  // side: 'right' | 'left' | 'none'
  setActiveSide(side);
});
```

---

### Hand State — `ubox.onHandState(fn)`

Open/closed/pointing state of a specific hand.

```js
ubox.onHandState(function (side, state, userId) {
  // side: 'right' | 'left'
  // state: 'open' | 'closed' | 'point'
  if (side === 'right' && state === 'closed') grabObject();
});
```

---

## MIDI Namespace (`realityos.midi.*`)

### `ubox.onMidi(fn)`

All MIDI events arrive through a single callback. Discriminate by `event.name`.

```js
ubox.onMidi(function (event) {
  switch (event.name) {

    case 'realityos.midi.note_on':
      // event.channel (1–16), event.note (0–127), event.velocity (1–127),
      // event.note_name ('C4', 'F#3', …), event.device (string)
      playNote(event.note, event.velocity);
      break;

    case 'realityos.midi.note_off':
      // event.channel, event.note, event.velocity (0), event.note_name, event.device
      releaseNote(event.note);
      break;

    case 'realityos.midi.control_change':
      // event.channel, event.control (0–127), event.value (0–127), event.device
      if (event.control === 64) setSustainPedal(event.value > 63);
      break;

    case 'realityos.midi.program_change':
      // event.channel, event.program (0–127), event.device
      setInstrument(event.program);
      break;

    case 'realityos.midi.pitch_bend':
      // event.channel, event.value (-8192..8191, 0 = center), event.device
      setPitchBend(event.value / 8192); // normalize to -1..1
      break;

    case 'realityos.midi.device_list':
      // event.devices: [{index, name}, …]
      populateDevicePicker(event.devices);
      break;
  }
});
```

**Wire payloads**:
```json
{ "name": "realityos.midi.note_on",       "channel": 1, "note": 60, "velocity": 100, "note_name": "C4",  "device": "KeyLab 61" }
{ "name": "realityos.midi.note_off",       "channel": 1, "note": 60, "velocity": 0,   "note_name": "C4",  "device": "KeyLab 61" }
{ "name": "realityos.midi.control_change", "channel": 1, "control": 64, "value": 127, "device": "KeyLab 61" }
{ "name": "realityos.midi.program_change", "channel": 1, "program": 5,               "device": "KeyLab 61" }
{ "name": "realityos.midi.pitch_bend",     "channel": 1, "value": 8192,              "device": "KeyLab 61" }
{ "name": "realityos.midi.device_list",    "devices": [{ "index": 0, "name": "KeyLab 61" }] }
```

> **Note**: velocity-0 NoteOn messages are normalized to `note_off` by the MIDI node automatically.

### MIDI Node Actions

```js
// Switch MIDI input device without restarting the node
ubox.sendNodeAction('midi-01', 'set_device', { index: 1 });

// Re-emit device list (triggers realityos.midi.device_list event)
ubox.sendNodeAction('midi-01', 'list_devices', {});
```

---

## Lights Namespace (`realityos.lights.*`)

The DMX node (`dmx-01`) bridges Art-Net/DMX512 hardware. It is an **actuator**: your app sends commands, the node drives the physical lights.

### Events (Node → App)

#### `ubox.onDMXDiscovered(fn)` — hardware scan result

```js
ubox.onDMXDiscovered(function (event) {
  // event.node_id, event.target_ip, event.devices: [{ip, short_name, long_name, mac, num_ports, out_universes}]
  event.devices.forEach(d => console.log('found', d.short_name, 'at', d.ip));
});
```

#### `ubox.onDMXState(fn)` — fixture channel state

```js
ubox.onDMXState(function (event) {
  // event.node_id, event.target_ip, event.fixtures: [{id, type, address, channels}]
  event.fixtures.forEach(f => {
    console.log(f.id, 'dim:', f.channels.dim, 'rgb:', f.channels.red, f.channels.green, f.channels.blue);
  });
});
```

Emitted every 10 s and immediately after any fixture list change.

---

### Actions (App → Node)

All sent via `ubox.sendNodeAction(nodeId, command, data)`.

```js
// Scan for Art-Net hardware on the network
ubox.sendNodeAction('dmx-01', 'lights.discover', {});

// Point the node at a specific Art-Net device
ubox.sendNodeAction('dmx-01', 'lights.set_target', { ip: '2.0.0.100' });

// Set individual fixture channels
ubox.sendNodeAction('dmx-01', 'lights.set', {
  fixture_id: 'par-01',
  channels: { dim: 200, red: 255, green: 0, blue: 0, white: 0, strobe: 0 }
});

// Apply a named preset
ubox.sendNodeAction('dmx-01', 'lights.preset', { fixture_id: 'par-01', preset: 'red' });
// presets: 'full_white' | 'red' | 'green' | 'blue' | 'blackout'

// Black out all fixtures
ubox.sendNodeAction('dmx-01', 'lights.blackout', {});

// Set a single DMX channel by address
ubox.sendNodeAction('dmx-01', 'lights.set_raw', { address: 3, value: 128 });

// Manage fixtures at runtime
ubox.sendNodeAction('dmx-01', 'lights.add_fixture',     { id: 'par-01', type: 'led-par-6ch', address: 1 });
ubox.sendNodeAction('dmx-01', 'lights.remove_fixture',  { fixture_id: 'par-01' });
ubox.sendNodeAction('dmx-01', 'lights.clear_fixtures',  {});
```

### Fixture Types

| Type | Channels | Layout |
|------|----------|--------|
| `led-par-6ch` | 6 | DIM · R · G · B · W · STROBE |
| `pixel-bar-4ch` | 4 | R · G · B · W |
| `pixel-bar-8ch` | 8 | DIM · R · G · B · W · STROBE · PROG · SPEED |
| `pixel-bar-96ch` | 96 | 24 pixels × (R · G · B · W) |
| `pixel-bar-100ch` | 100 | DIM · STROBE · PROG · SPEED + 24 × (R · G · B · W) |

### REST Alternative

Node actions can also be sent over HTTP (useful for server-side tooling or dashboards):

```
POST http://127.0.0.1:8080/api/node/action
Content-Type: application/json

{ "node_id": "dmx-01", "command": "lights.blackout", "data": {} }
```

Response: `200 {"ok":true}` or `503 {"error":"node not connected"}`.

---

## Custom Events (`realityos.custom.*`)

Bidirectional developer-defined events with free-form JSON.

```js
// Listen for custom events coming from xspace or other apps
ubox.onCustom(function (event) {
  // event.name = 'realityos.custom.<something>'
  if (event.name === 'realityos.custom.score_updated') {
    updateScoreDisplay(event.points);
  }
});

// Send a custom event to xspace
ubox.sendCustom('score_updated', { points: 150, player: 'p1' });
// → sends: { name: 'realityos.custom.score_updated', points: 150, player: 'p1' }

// You can also pass the full name
ubox.sendCustom('realityos.custom.zone_trigger', { zone: 'A', active: true });
```

---

## Operational Events (`realityos.op.*`)

### Stream Level

Cursor, body, and face are high-frequency streams. By default they are **not forwarded to xspace** (cloud). Control this with stream level:

```js
ubox.setStreamLevel('medium'); // shorthand

// Or send the full op event
ubox.sendOp('realityos.op.stream.level', { level: 'high' });
```

| Level | Cursor | Body | Face |
|-------|--------|------|------|
| `none` (default) | ✗ | ✗ | ✗ |
| `low` | Every 10th frame | Every 10th frame | Every 10th frame |
| `medium` | Every 5th frame | Every 5th frame | Every 5th frame |
| `high` | Every frame | Every frame | Every frame |
| `smart` | Adaptive (≈ medium) | Adaptive | Adaptive |

Interaction events (`navigate`, `select`, `presence`, `gesture`, `hand.*`) are **always forwarded** regardless of stream level.

---

## `ubox.onStandard(fn)` — Incoming Standard Events from Xspace

Standard `realityos.*` events forwarded from the cloud (another player or server-side logic) arrive here:

```js
ubox.onStandard(function (data) {
  console.log('standard from xspace', data);
});
```

---

## Complete API Reference

```js
// ── Standard event callbacks ───────────────────────────────────────────────
ubox.onPresence(fn)    // fn(type, userId)            type: 'enter'|'leave'
ubox.onNavigate(fn)    // fn(direction, hand, userId) direction: 'left'|'right'|'up'|'down'
ubox.onSelect(fn)      // fn(type, hand, userId)      type: 'grip'|'release'|'push'|'pull'
ubox.onGesture(fn)     // fn(type, data, userId)      type: 'zoom'|'kick'|'jump'|'lean'|'wave'|'point'
ubox.onCursor(fn)      // fn(x, y, userId)            normalized 0–1
ubox.onBody(fn)        // fn(skeletons)               array of skeleton objects
ubox.onFace(fn)        // fn(data)                    yaw/pitch/roll + expression flags
ubox.onHandActive(fn)  // fn(side, userId)            side: 'right'|'left'|'none'
ubox.onHandState(fn)   // fn(side, state, userId)     state: 'open'|'closed'|'point'

// ── MIDI ──────────────────────────────────────────────────────────────────
ubox.onMidi(fn)        // fn(event)  event.name = 'realityos.midi.*'

// ── Lights / DMX ──────────────────────────────────────────────────────────
ubox.onDMXDiscovered(fn) // fn(event) — hardware scan result
ubox.onDMXState(fn)      // fn(event) — fixture channel state

// ── Custom / Standard from xspace ─────────────────────────────────────────
ubox.onCustom(fn)      // fn(event) — realityos.custom.*
ubox.onStandard(fn)    // fn(data)  — realityos.* forwarded from xspace

// ── Operational ───────────────────────────────────────────────────────────
ubox.setStreamLevel(level)        // 'none'|'low'|'medium'|'high'|'smart'
ubox.sendOp(name, payload)        // send any realityos.op.* event

// ── Outbound ──────────────────────────────────────────────────────────────
ubox.sendCustom(name, payload)             // send realityos.custom.* to xspace
ubox.sendNodeAction(nodeId, cmd, data)     // route node.action to an actuator node

// ── Legacy (backward compat — avoid in new apps) ─────────────────────────
ubox.onArduino(fn)     // fn(value) — raw Arduino-wrapped messages
ubox.onZoom(fn)        // fn(state, delta, distance) — legacy zoom stream
sendToArduino(msg)     // global; sends raw string via WebSocket
```

---

## Kinect Legacy → RealityOS Translation

The hub translates Kinect-specific event names automatically. New apps receive the canonical name; old apps continue to receive both.

| Kinect name | Canonical event | Notes |
|-------------|----------------|-------|
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
| `active_hand` | `realityos.cursor` | x/y passed through |
| `skeleton` | `realityos.body` | skeletons passed through |
| `face` | `realityos.face` | all fields passed through |

---

## Full App Example

A minimal interactive experience responding to presence, navigation, gestures, and MIDI:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>My Ubox Experience</title>
  <style>
    body { margin: 0; background: #000; color: #fff; font-family: sans-serif;
           display: flex; align-items: center; justify-content: center;
           height: 100vh; font-size: 3rem; }
    #msg  { opacity: 0; transition: opacity 0.4s; }
    #msg.visible { opacity: 1; }
    #cursor { position: fixed; width: 20px; height: 20px; border-radius: 50%;
               background: #00d4ff; pointer-events: none; transform: translate(-50%,-50%); }
  </style>
</head>
<body>
  <div id="msg">Welcome</div>
  <div id="cursor" style="display:none"></div>

  <script src="http://127.0.0.1:8080/uboxclient.js"></script>
  <script>
    var msg    = document.getElementById('msg');
    var cursor = document.getElementById('cursor');

    function show(text) {
      msg.textContent = text;
      msg.classList.add('visible');
      clearTimeout(show._t);
      show._t = setTimeout(function () { msg.classList.remove('visible'); }, 2000);
    }

    // ── Presence ─────────────────────────────────────────────────────────
    ubox.onPresence(function (type) {
      if (type === 'enter') show('Hello!');
      if (type === 'leave') show('Goodbye');
    });

    // ── Navigation ───────────────────────────────────────────────────────
    ubox.onNavigate(function (direction) {
      show('→ ' + direction);
    });

    // ── Select ───────────────────────────────────────────────────────────
    ubox.onSelect(function (type) {
      if (type === 'grip') show('Selected!');
    });

    // ── Cursor ───────────────────────────────────────────────────────────
    ubox.onCursor(function (x, y) {
      cursor.style.display = 'block';
      cursor.style.left = (x * 100) + 'vw';
      cursor.style.top  = (y * 100) + 'vh';
    });

    // ── MIDI ─────────────────────────────────────────────────────────────
    ubox.onMidi(function (event) {
      if (event.name === 'realityos.midi.note_on') {
        show(event.note_name + ' vel:' + event.velocity);
        document.body.style.background = 'hsl(' + (event.note * 3) + ',70%,15%)';
      }
      if (event.name === 'realityos.midi.note_off') {
        document.body.style.background = '#000';
      }
    });

    // ── Lights — react to presence with DMX ──────────────────────────────
    ubox.onPresence(function (type) {
      if (type === 'enter') {
        ubox.sendNodeAction('dmx-01', 'lights.preset', { fixture_id: 'par-01', preset: 'full_white' });
      }
      if (type === 'leave') {
        ubox.sendNodeAction('dmx-01', 'lights.blackout', {});
      }
    });
  </script>
</body>
</html>
```

---

## Configuration Reference

Nodes are declared in `config.yaml` (installed: `%LOCALAPPDATA%\Ubox Physical Player\config.yaml`).

```yaml
nodes:
  - id: midi-01
    runtime: native
    mode: detector
    script: midi-sensor.exe
    manifest_url: "https://raw.githubusercontent.com/ubiquo-techs/ubox-distribution/main/manifests/sensors/midi/manifest.json"
    args: ["--device-index", "0"]
    enabled: true

  - id: dmx-01
    runtime: native
    mode: actuator
    script: dmx-sensor.exe
    manifest_url: "https://raw.githubusercontent.com/ubiquo-techs/ubox-distribution/main/manifests/sensors/dmx/manifest.json"
    args: ["--universe", "0", "--local-ip", "2.0.0.2", "--hz", "25", "--fixtures", "fixtures.json"]
    enabled: true

  - id: lidar-01
    runtime: native
    mode: detector
    script: lidar-sensor.exe
    manifest_url: "https://raw.githubusercontent.com/ubiquo-techs/ubox-distribution/main/manifests/sensors/lidar/manifest.json"
    args: ["--port", "COM5", "--baud", "460800"]
    enabled: true
```

---

## Updating This Document

When the RealityOS protocol changes:

1. Update `docs/realityos-protocol.md` with the protocol-level spec.
2. Update `core/dashboard/uboxclient.js` with the new browser API.
3. Update this file to reflect the new events, callbacks, or actions.

This file is the entry point for agents and developers building Ubox experiences. Keep all three files in sync.
