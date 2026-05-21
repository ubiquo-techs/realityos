# RealityOS

**RealityOS** is an open standard for the phygital layer — the boundary where physical bodies, spaces, and objects interact with digital experiences.

It defines a minimal, hardware-agnostic protocol for:
- Detecting physical presence, movement, and intent (sensors)
- Controlling physical hardware from digital experiences (actuators)
- Coordinating real-time state between physical players and cloud backends

Ubox is the reference implementation. Any node that speaks this protocol is a first-class RealityOS node.

---

## Four Namespaces

| Namespace | Direction | Description |
|-----------|-----------|-------------|
| `realityos.*` | Node → App → Xspace | Standard phygital events — presence, navigation, selection, body, face, cursor, lights, MIDI |
| `realityos.op.*` | Any ↔ Any | Operational coordination (stream level control) |
| `realityos.custom.*` | App ↔ Xspace | Developer-defined free-form events |
| `realityos.lights.*` | App → Node (actions) / Node → App (state) | DMX/Art-Net lighting control |
| `realityos.midi.*` | Node → App | MIDI hardware events |

---

## Platform Peers

```
┌─────────────────────────────────────────────────────┐
│                    Ubox Core (cloud)                 │
│              Socket.IO  /xpace  namespace            │
└────────────────────────┬────────────────────────────┘
                         │ wss://core.ubox.world
                         │ events: "standard", "custom"
┌────────────────────────▼────────────────────────────┐
│              Physical Player (on-site)               │
│         ws://127.0.0.1:8181/KinectHtml5              │
│              WebSocket Hub + HTTP server             │
└──────┬──────────────────────────────────┬───────────┘
       │ node.event / node.action         │ node.event / node.data
┌──────▼──────┐                    ┌──────▼──────┐
│  Detectors  │                    │  Actuators  │
│ Kinect LiDAR│                    │  DMX Lights │
│ MIDI Button │                    │             │
└─────────────┘                    └─────────────┘
                         │
              ┌──────────▼──────────┐
              │  Browser App        │
              │  uboxclient.js API  │
              └─────────────────────┘
```

---

## JSON Schema

The canonical schema is published at:

```
https://raw.githubusercontent.com/ubiquo-techs/realityos/main/schema/realityos-schema.json
```

It validates all standard `realityos.*` events using JSON Schema 2020-12. The Ubox Physical Player downloads this schema at startup and uses it for soft validation — failures are logged but messages are never dropped.

Current version: **1.1.0** (see [CHANGELOG.md](CHANGELOG.md))

---

## Documentation

| Document | Description |
|----------|-------------|
| [docs/protocol.md](docs/protocol.md) | Node wire protocol — how nodes connect, register, and communicate with the hub |
| [docs/events.md](docs/events.md) | All events in all namespaces with wire JSON examples |
| [docs/app-developer.md](docs/app-developer.md) | Browser app developer guide — `uboxclient.js` API with JS examples |

---

## Node Types

| Type | Role |
|------|------|
| **Detector** | Reads physical state, emits events to apps (Kinect, LiDAR, MIDI, button) |
| **Actuator** | Receives actions from apps, drives physical hardware (DMX lights) |
| **Bidirectional** | Both detects and actuates |

Every node declares its type in `node.register` via the `mode` field: `"detector" | "actuator" | "bidirectional"`.

---

## License

MIT — Ubiquo Technologies
