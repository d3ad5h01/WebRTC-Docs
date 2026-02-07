# WebRTC-Docs

Documentation and study notes for WebRTC internals.

## Contents

### [Stats/](Stats/)

Audio-only WebRTC statistics from `RTCPeerConnection.getStats()`:

- **[JitterBuffer](Stats/JitterBuffer.md)** — Buffer delay, target delay, emitted count, minimum delay
- **[Concealment](Stats/Concealment.md)** — Concealed samples, silent concealment, concealment events
- **[SamplesInsertedRemoved](Stats/SamplesInsertedRemoved.md)** — Playout speed adjustments (deceleration / acceleration)
- **[PacketLoss](Stats/PacketLoss.md)** — Packets lost, packets received, loss rate calculation
- **[PacketDiscard](Stats/PacketDiscard.md)** — Packets discarded by the jitter buffer (late arrivals)
- **[AudioLevel](Stats/AudioLevel.md)** — Audio level, energy, duration, silence detection

Each document covers the stat definitions, derived calculations, scenario behavior (normal, high jitter, burst loss, congestion, etc.), and code examples.

## Reference

- [W3C WebRTC Statistics API](https://www.w3.org/TR/webrtc-stats/)
- [RFC 3550 — RTP](https://datatracker.ietf.org/doc/html/rfc3550)
- [RFC 6464 — Client-to-Mixer Audio Level](https://datatracker.ietf.org/doc/html/rfc6464)
