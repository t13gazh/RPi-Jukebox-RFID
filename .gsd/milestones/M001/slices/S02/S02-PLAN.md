# S02: Real-Time Layer

**Goal:** The browser receives instant player state updates when anything changes (physical button, RFID card, or web UI action)
**Demo:** WebSocket at `/ws` sends player state on connect; physical button press updates all connected browsers within 500ms

## Must-Haves

- WebSocket endpoint at `/ws` accepts connections and sends player state on connect
- When a physical button or RFID card changes MPD state, all connected browsers update within 500ms
- WebSocket reconnects automatically with exponential backoff after connection loss
- MPD idle bridge runs as a background task with zero CPU usage when player is idle

## Tasks


## Files Likely Touched

