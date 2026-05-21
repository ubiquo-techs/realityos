# RealityOS — Node Wire Protocol

This document describes how sensor and actuator nodes connect to the Ubox Physical Player hub and exchange messages.

---

## Transport

Nodes connect via WebSocket to:

```
ws://127.0.0.1:8181/KinectHtml5
```

All messages are JSON text frames.

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
| `node.register` | Node → Hub | On connect | Declares node identity, mode, and optional test page |
| `node.event` | Node → Hub | On phygital event | RealityOS event payload |
| `node.data` | Node → Hub | Continuous data | Legacy Arduino-wrapped data (LDR format) |
| `node.heartbeat` | Node → Hub | Every 10 s | Keepalive with uptime |
| `node.log` | Node → Hub | Operational | Human-readable log message |
| `node.action` | App/REST → Hub → Node | To control actuators | Command routed to a named node |

---

## `node.register`

Sent immediately after the WebSocket connection is established.

```json
{
  "type": "node.register",
  "level": 0,
  "timestamp": "2026-01-01T00:00:00Z",
  "payload": {
    "node_id":   "lidar-01",
    "node_type": "lidar",
    "mode":      "detector",
    "version":   "1.2.0",
    "test_page": "<html>...</html>"
  }
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `node_id` | Yes | Unique identifier for this node instance |
| `node_type` | Yes | Hardware type string (e.g. `"kinect2"`, `"lidar"`, `"dmx"`, `"midi"`) |
| `mode` | Yes | `"detector"` \| `"actuator"` \| `"bidirectional"` |
| `version` | Yes | Node binary version |
| `test_page` | No | HTML string for the node test UI (served at `GET /node-test/{node_id}`) |

---

## `node.event`

Carries a RealityOS event payload. The `payload` is a flat JSON object; `name` identifies the event.

```json
{
  "type": "node.event",
  "level": 1,
  "timestamp": "2026-01-01T00:00:00Z",
  "payload": {
    "node_id":   "kinect2-01",
    "node_type": "kinect2",
    "data": {
      "name": "realityos.presence",
      "type": "enter",
      "user_id": "u1"
    }
  }
}
```

The hub extracts `payload.data`, validates it against the RealityOS schema, and forwards it to browser apps.

---

## `node.data`

Legacy format — used by the LiDAR node for backward compatibility with apps that read `msgFromArduino()`.

```json
{
  "type": "node.data",
  "level": 1,
  "timestamp": "2026-01-01T00:00:00Z",
  "payload": {
    "node_id": "lidar-01",
    "data": {
      "name":  "LDR",
      "value": "LDR X: 312.1, Y: 205.5"
    }
  }
}
```

The hub Arduino-wraps `payload.data` before forwarding to KindApp clients.

New nodes should use `node.event` instead.

---

## `node.heartbeat`

```json
{
  "type": "node.heartbeat",
  "level": 3,
  "timestamp": "2026-01-01T00:00:00Z",
  "payload": {
    "node_id":  "lidar-01",
    "pid":      12345,
    "uptime_s": 120
  }
}
```

Sent every 10 seconds. Dashboard uses this to show node uptime.

---

## `node.log`

```json
{
  "type": "node.log",
  "level": 2,
  "timestamp": "2026-01-01T00:00:00Z",
  "payload": {
    "node_id": "lidar-01",
    "message": "serial port COM5 opened successfully"
  }
}
```

Forwarded to the dashboard only — not to browser apps.

---

## `node.action`

Sent by browser apps (or REST) to control an actuator node. The hub routes the envelope directly to the WebSocket connection registered under `node_id`.

```json
{
  "type": "node.action",
  "payload": {
    "node_id": "dmx-01",
    "command": "lights.set",
    "data": {
      "fixture_id": "par-01",
      "channels": { "dim": 200, "red": 255, "green": 0, "blue": 0 }
    }
  }
}
```

**REST alternative** — `POST http://127.0.0.1:8080/api/node/action` with `payload` as the request body:

```json
{ "node_id": "dmx-01", "command": "lights.blackout", "data": {} }
```

Response: `200 {"ok":true}` or `503 {"error":"node not connected"}`.

---

## Hub Routing Table

| Message type | KindApp clients (`/KinectHtml5`) | KindDashboard (`/ws/dashboard`) | Target node |
|---|---|---|---|
| `node.event` | ✓ payload forwarded (translated to RealityOS if needed) | ✓ full envelope | — |
| `node.data` | ✓ Arduino-wrapped | ✓ full envelope | — |
| `node.action` | — | ✓ full envelope | ✓ routed by `node_id` |
| `node.heartbeat` | — | ✓ full envelope | — |
| `node.log` | — | ✓ full envelope | — |
| `node.register` | — | ✓ + registry update | — |

---

## Reconnection Behavior

Nodes must reconnect automatically after any WebSocket error:

- **WebSocket disconnect**: reconnect after 3 s
- **Serial/hardware error**: reconnect after 2 s
- On reconnect: re-send `node.register` immediately

---

## Legacy Name Translation

The hub translates Kinect v2 event names to canonical RealityOS names automatically. Both the original name and the translated name are forwarded so legacy apps keep working.

See [events.md](events.md) for the full translation table.
