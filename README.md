# WebRTC Docs

Deepdive into WebRTC internals, audio pipelines, and real-time monitoring.

**[View Documentation](https://d3ad5h01.github.io/WebRTC-Docs/index.html)**

## Topics Covered

### Architecture & Components
- WebRTC Components — ICE, DTLS, Transport, NetEQ, Decoder, PLC
- PCM & Audio Encoding — Opus, PCMU codecs, AudioContext

### Connection & ICE
- WebRTC Establishment — Offer/Answer, ICE candidates, STUN, DTLS handshake

### Audio Pipelines
- Audio Send Pipeline — Microphone → Audio Processing → Encoder → Network
- Audio Receive Pipeline — Packets → Jitter Buffer → Decoder → Speaker

### Jitter & Buffering
- Jitter Buffer — Buffer delay, target delay, network variance absorption
- Jitter Calculation — RFC 3550 formula with examples

### Metrics & Monitoring
- Monitoring Audio Stats — RTT, jitter, bitrate, packet loss, concealment

## Tools

- **WebRTC Internals Analyzer** — Upload a `webrtc-internals` dump to visualize audio metrics with interactive charts
