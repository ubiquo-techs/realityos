# RealityOS — App Developer Reference

Build phygital browser experiences using the RealityOS event protocol. The reference implementation is [Ubox](https://ubox.world).

---

## Overview

Sensor nodes publish structured RealityOS events to a local hub. Your browser app subscribes through `uboxclient.js` and reacts to physical events — presence, movement, gestures, MIDI, lights — with plain JavaScript callbacks.

---

## Quick Start

```html
<!DOCTYPE html>
<html>
<head><title>My Experience</title></head>
<body>

<!-- Load the RealityOS browser client -->
<script src="http://127.0.0.1:8080/uboxclient.js"></script>

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

`uboxclient.js` auto-connects to the local hub on load. No manual setup needed.

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

**Wire payload**:
```json
{ "name": "realityos.presence", "type": "enter", "user_id": "u1" }
```

---

### Navigate — `ubox.onNavigate(fn)`

Directional intent — swipe, sweep, step direction.

```js
ubox.onNavigate(function (direction, hand, userId) {
  // direction: 'left' | 'right' | 'up' | 'down'
  // hand: 'right' | 'left'
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

Physical confirmation of intent — grip, push, pull.

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
  switch (type) {
    case 'zoom':  handleZoom(data.direction);        break; // direction: 'in'|'out'
    case 'kick':  handleKick(data.side, data.angle); break; // side: 'left'|'right', angle: degrees, distance: mm
    case 'jump':  handleJump(data.intensity);        break; // intensity: 0–100
    case 'lean':  handleLean(data.x, data.y);        break; // x/y: -1.0..1.0
    case 'wave':  handleWave(data.side);             break; // side: 'left'|'right'
    case 'point': handlePoint(data.side);            break; // side: 'left'|'right'
  }
});
```

---

### Cursor — `ubox.onCursor(fn)`

Continuous position stream. `x`/`y` normalized `0.0–1.0` regardless of sensor type.

```js
ubox.onCursor(function (x, y, userId) {
  // x: 0.0 (left) – 1.0 (right)
  // y: 0.0 (top/far) – 1.0 (bottom/near)
  moveCursor(x * window.innerWidth, y * window.innerHeight);
});
```

> **Stream level**: high-frequency. Not forwarded to the cloud by default. Call `ubox.setStreamLevel('medium')` to enable.

---

### Body (skeleton stream) — `ubox.onBody(fn)`

Full skeleton data per frame.

```js
ubox.onBody(function (skeletons) {
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

```js
ubox.onHandActive(function (side, userId) {
  // side: 'right' | 'left' | 'none'
  setActiveSide(side);
});
```

---

### Hand State — `ubox.onHandState(fn)`

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
      setPitchBend(event.value / 8192);
      break;

    case 'realityos.midi.device_list':
      // event.devices: [{index, name}, …]
      populateDevicePicker(event.devices);
      break;
  }
});
```

### MIDI Node Actions

```js
// Switch MIDI input device without restarting the node
ubox.sendNodeAction('midi-node-name', 'set_device', { index: 1 });

// Re-emit device list
ubox.sendNodeAction('midi-node-name', 'list_devices', {});
```

---

## Lights Namespace (`realityos.lights.*`)

A lights actuator node bridges Art-Net/DMX512 hardware. Your app sends commands, the node drives the physical lights.

### Events (Node → App)

```js
ubox.onDMXDiscovered(function (event) {
  // event.node_id, event.target_ip, event.devices: [{ip, short_name, long_name, mac, num_ports, out_universes}]
  event.devices.forEach(d => console.log('found', d.short_name, 'at', d.ip));
});

ubox.onDMXState(function (event) {
  // event.node_id, event.target_ip, event.fixtures: [{id, type, address, channels}]
  event.fixtures.forEach(f => console.log(f.id, 'dim:', f.channels.dim));
});
```

### Actions (App → Node)

```js
ubox.sendNodeAction('lights-node-name', 'lights.discover', {});
ubox.sendNodeAction('lights-node-name', 'lights.set_target', { ip: '2.0.0.100' });
ubox.sendNodeAction('lights-node-name', 'lights.set', {
  fixture_id: 'fixture-name',
  channels: { dim: 200, red: 255, green: 0, blue: 0, white: 0, strobe: 0 }
});
ubox.sendNodeAction('lights-node-name', 'lights.preset',  { fixture_id: 'fixture-name', preset: 'red' });
ubox.sendNodeAction('lights-node-name', 'lights.blackout', {});
ubox.sendNodeAction('lights-node-name', 'lights.set_raw',  { address: 3, value: 128 });
ubox.sendNodeAction('lights-node-name', 'lights.add_fixture',    { id: 'fixture-name', type: 'led-par-6ch', address: 1 });
ubox.sendNodeAction('lights-node-name', 'lights.remove_fixture', { fixture_id: 'fixture-name' });
ubox.sendNodeAction('lights-node-name', 'lights.clear_fixtures', {});
```

**Presets**: `full_white` · `red` · `green` · `blue` · `blackout`

| Fixture type | Channels | Layout |
|---|---|---|
| `led-par-6ch` | 6 | DIM · R · G · B · W · STROBE |
| `pixel-bar-4ch` | 4 | R · G · B · W |
| `pixel-bar-8ch` | 8 | DIM · R · G · B · W · STROBE · PROG · SPEED |
| `pixel-bar-96ch` | 96 | 24 pixels × (R · G · B · W) |
| `pixel-bar-100ch` | 100 | DIM · STROBE · PROG · SPEED + 24 × (R · G · B · W) |

---

## Custom Events (`realityos.custom.*`)

Bidirectional developer-defined events with free-form JSON.

```js
ubox.onCustom(function (event) {
  if (event.name === 'realityos.custom.score_updated') {
    updateScoreDisplay(event.points);
  }
});

ubox.sendCustom('score_updated', { points: 150, player: 'p1' });
// → sends: { name: 'realityos.custom.score_updated', points: 150, player: 'p1' }
```

---

## Operational Events (`realityos.op.*`)

### Stream Level

Cursor, body, and face are high-frequency streams. Control cloud forwarding:

```js
ubox.setStreamLevel('medium');
// or
ubox.sendOp('realityos.op.stream.level', { level: 'high' });
```

| Level | Forwarding |
|-------|-----------|
| `none` (default) | Off |
| `low` | Every 10th frame |
| `medium` | Every 5th frame |
| `high` | Every frame |
| `smart` | Adaptive (≈ medium) |

Interaction events are always forwarded regardless of stream level.

---

## Incoming Events from the Cloud

Standard events forwarded from the cloud backend arrive on `ubox.onStandard`. Custom events arrive on `ubox.onCustom`.

```js
ubox.onStandard(function (data) {
  var event = JSON.parse(data);
  console.log('from cloud', event);
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
ubox.onMidi(fn)          // fn(event)  event.name = 'realityos.midi.*'

// ── Lights / DMX ──────────────────────────────────────────────────────────
ubox.onDMXDiscovered(fn) // fn(event) — hardware scan result
ubox.onDMXState(fn)      // fn(event) — fixture channel state

// ── Cloud events ──────────────────────────────────────────────────────────
ubox.onCustom(fn)        // fn(event) — realityos.custom.* (bidirectional)
ubox.onStandard(fn)      // fn(data)  — realityos.* forwarded from cloud

// ── Operational ───────────────────────────────────────────────────────────
ubox.setStreamLevel(level)         // 'none'|'low'|'medium'|'high'|'smart'
ubox.sendOp(name, payload)         // send any realityos.op.* event

// ── Outbound ──────────────────────────────────────────────────────────────
ubox.sendCustom(name, payload)     // send realityos.custom.* to cloud
ubox.sendNodeAction(nodeId, cmd, data)  // route node.action to an actuator node
```

---

## Full App Example

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>My RealityOS Experience</title>
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

    ubox.onPresence(function (type) {
      if (type === 'enter') show('Hello!');
      if (type === 'leave') show('Goodbye');
    });

    ubox.onNavigate(function (direction) { show('→ ' + direction); });

    ubox.onSelect(function (type) {
      if (type === 'grip') show('Selected!');
    });

    ubox.onCursor(function (x, y) {
      cursor.style.display = 'block';
      cursor.style.left = (x * 100) + 'vw';
      cursor.style.top  = (y * 100) + 'vh';
    });

    ubox.onMidi(function (event) {
      if (event.name === 'realityos.midi.note_on') {
        show(event.note_name + ' vel:' + event.velocity);
        document.body.style.background = 'hsl(' + (event.note * 3) + ',70%,15%)';
      }
      if (event.name === 'realityos.midi.note_off') {
        document.body.style.background = '#000';
      }
    });

    ubox.onPresence(function (type) {
      if (type === 'enter') ubox.sendNodeAction('lights-node-name', 'lights.preset', { fixture_id: 'fixture-name', preset: 'full_white' });
      if (type === 'leave') ubox.sendNodeAction('lights-node-name', 'lights.blackout', {});
    });
  </script>
</body>
</html>
```
