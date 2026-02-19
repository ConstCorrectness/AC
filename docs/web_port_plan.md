# AssaultCube Full Web Port Plan (No ENet)

This document outlines a **full web-native port** strategy for AssaultCube with:

- Rendering: **WebGL2**
- Networking: **WebSockets** (gameplay + control plane)
- Voice chat: **WebRTC audio**
- Transport: **No ENet dependency** in the browser client

## 1) Target Architecture

### 1.1 Client Runtime (Browser)
- WASM module for core simulation/gameplay logic.
- JavaScript/TypeScript host for platform integrations:
  - WebGL2 rendering backend
  - Input (pointer lock, keyboard, gamepad)
  - Asset loading / persistence
  - WebSocket connection manager
  - WebRTC voice stack

### 1.2 Server Runtime
- Native authoritative game server process.
- Replace ENet-based client transport path with a **WebSocket session layer**.
- Optional gateway process if you want to preserve native server internals while iterating.

### 1.3 Signaling and Presence
- WebSocket control channel handles:
  - login/session
  - lobby/room join
  - voice channel membership
  - player metadata and moderation controls

## 2) Networking Without ENet

### 2.1 WebSocket Protocol Model
- Use binary frames (`ArrayBuffer`) for compact game packets.
- Packet envelope:
  - protocol version
  - message type
  - sequence number
  - ack bitset (for selective reliability)
  - payload

### 2.2 Reliability Classes
Implement ENet-like behavior at app layer:
- **Reliable ordered**: chat, inventory/stateful commands.
- **Unreliable sequenced**: position/aim snapshots.
- **Reliable unordered** (optional): non-order-dependent control events.

### 2.3 Lag Compensation and Tick Model
- Server authoritative fixed tick (e.g., 60 Hz).
- Client sends input commands stamped with local tick.
- Server echoes authoritative state + last processed input tick.
- Client performs prediction + reconciliation.

### 2.4 Match/Server Discovery
- Replace LAN broadcast discovery with:
  - central match service list endpoint, and/or
  - region lobby shards.

## 3) Voice Chat via WebRTC

### 3.1 Topology
For solo development, start with **SFU-based group voice**:
- Easier scaling than full mesh.
- Lower uplink cost for players in larger rooms.

### 3.2 Signaling
Reuse existing WebSocket channel for WebRTC signaling:
- SDP offer/answer
- ICE candidates
- channel membership updates

### 3.3 Audio Pipeline
- Capture mic with `getUserMedia`.
- Apply browser AEC/NS/AGC constraints.
- Optional positional attenuation done client-side in WebAudio gain graph.

### 3.4 Moderation/UX Controls
- Push-to-talk, mute self, mute remote.
- Server-side role checks for room moderation commands.

## 4) Rendering Migration (WebGL2)

### 4.1 Core Changes
- Remove fixed-function/immediate mode assumptions.
- Introduce render abstraction:
  - static world geometry
  - skinned/animated meshes
  - particles
  - HUD/2D overlays

### 4.2 Shader and Material System
- Material definition -> shader variant mapping.
- Texture atlas and sampler state registry.
- Uniform buffer patterns for per-frame/per-object data.

### 4.3 Performance Baseline
- Frustum culling + coarse occlusion strategy.
- Minimize WebGL state changes.
- Batch UI quads and decals.

## 5) Security Model
- Validate all gameplay messages server-side.
- Signed session tokens for websocket auth.
- Rate limiting for chat and command endpoints.
- Voice room auth tied to game session identity.

## 6) Delivery Plan (Solo-Optimized)

### Milestone A (2 weeks)
- Browser boot + input + single map render.
- WebSocket connect/login + ping.
- No combat, no voice.

### Milestone B (3-4 weeks)
- Core movement/combat replicated.
- Prediction + reconciliation stable.
- HUD and weapon rendering parity for a reduced weapon set.

### Milestone C (2-3 weeks)
- Match flow: lobby -> game -> postgame.
- Chat + moderation basics.
- WebRTC voice in party/team channels.

### Milestone D (ongoing)
- Performance tuning, anti-cheat hardening, content parity.
- Expand maps/modes and regression tooling.

## 7) Immediate Task List
1. Define binary packet schema + protocol versioning doc.
2. Implement websocket session manager (client and server).
3. Build server tick/input pipeline with reconciliation hooks.
4. Add prediction for movement + firing.
5. Implement WebRTC signaling over existing websocket channel.
6. Bring up SFU (or managed provider) for room voice.
7. Add end-to-end telemetry: RTT, jitter, packet loss, tick drift.
8. Add deterministic replay snippets for regression tests.

## 8) Non-Goals for Initial Launch
- Full feature parity with every legacy mode/map.
- Perfect visual parity with native renderer on day one.
- Browser P2P authoritative gameplay (server remains authoritative).

