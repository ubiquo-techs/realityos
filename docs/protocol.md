# RealityOS — Node Wire Protocol

This document describes how sensor and actuator nodes connect to a RealityOS-compatible hub and exchange messages.

---

## Transport

Nodes connect via WebSocket. All messages are JSON text frames.

---

## Hub Envelope

Every message is wrapped in a hub envelope:

```json
{
  "type": "node.event",
  "level": 1,
  "timestamp": "2026-01-01T00:00:00Z",
  "payload": { ... }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Message type (see table below) |
| `level` | int | Log level: 0=register, 1=event, 2=log, 3=heartbeat |
| `timestamp` | string | ISO 8601 UTC |
| `payload` | object | Type-specific payload |

---

## Message Types

| Type | Direction | When | Description |
|------|-----------|------|-------------|
| `node.register` | Node → Hub | On connect | Declares node identity, mode, and capabilities |
| `node.event` | Node → Hub | On phygital event | RealityOS event payload |
| `node.heartbeat` | Node → Hub | Every 10 s | Keepalive with uptime |
| `node.log` | Node → Hub | Operational | Human-readable log message |
| `node.action` | App → Hub → Node | To control actuators | Command routed to a named node |

---

## `node.register`

Sent immediately after the WebSocket connection is established.

```json
{
  "type": "node.register",
  "level": 0,
  "timestamp": "2026-01-01T00:00:00Z",
  "payload": {
    "node_id":   "depth-node-name",
    "node_type": "depth-camera",
    "mode":      "detector",
    "version":   "1.2.0"
  }
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `node_id` | Yes | Unique identifier for this node instance — set by the user in configuration |
| `node_type` | Yes | Hardware type string (e.g. `"depth-camera"`, `"lidar"`, `"dmx"`, `"midi"`, `"esp32"`) |
| `mode` | Yes | `"detector"` \| `"actuator"` \| `"bidirectional"` |
| `version` | Yes | Node binary/software version |

---

## `node.event`

Carries a RealityOS event payload. The `payload.data` field is a flat JSON object; `name` identifies the event.

```json
{
  "type": "node.event",
  "level": 1,
  "timestamp": "2026-01-01T00:00:00Z",
  "payload": {
    "node_id":   "depth-node-name",
    "node_type": "depth-camera",
    "data": {
      "name": "realityos.presence",
      "type": "enter",
      "user_id": "u1"
    }
  }
}
```

---

## `node.heartbeat`

```json
{
  "type": "node.heartbeat",
  "level": 3,
  "timestamp": "2026-01-01T00:00:00Z",
  "payload": {
    "node_id":  "depth-node-name",
    "pid":      12345,
    "uptime_s": 120
  }
}
```

Sent every 10 seconds. The hub uses this to monitor node health.

---

## `node.log`

```json
{
  "type": "node.log",
  "level": 2,
  "timestamp": "2026-01-01T00:00:00Z",
  "payload": {
    "node_id": "depth-node-name",
    "message": "device opened successfully"
  }
}
```

---

## `node.action`

Sent by apps to control an actuator node. The hub routes the envelope directly to the node registered under `node_id`.

```json
{
  "type": "node.action",
  "payload": {
    "node_id": "lights-node-name",
    "command": "lights.set",
    "data": {
      "fixture_id": "fixture-name",
      "channels": { "dim": 200, "red": 255, "green": 0, "blue": 0 }
    }
  }
}
```

---

## Reconnection Behavior

Nodes must reconnect automatically after any WebSocket error:

- **WebSocket disconnect**: reconnect after ~3 s
- **Hardware error**: reconnect hardware after ~2 s, then WebSocket if needed
- On reconnect: re-send `node.register` immediately
