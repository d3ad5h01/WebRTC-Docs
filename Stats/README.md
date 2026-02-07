# WebRTC Audio Statistics

This folder contains documentation for **audio-only** WebRTC statistics exposed via `RTCPeerConnection.getStats()`. All stats documented here apply to audio `RTCRtpReceiver` / `RTCRtpSender` streams and are defined in the [W3C WebRTC Stats spec](https://www.w3.org/TR/webrtc-stats/).

## Stats Covered

| Document | Key Metrics | Scenario Focus |
|----------|-------------|----------------|
| [JitterBuffer](JitterBuffer.md) | `jitterBufferDelay`, `jitterBufferTargetDelay`, `jitterBufferEmittedCount`, `jitterBufferMinimumDelay` | Buffering behavior under varying network jitter |
| [Concealment](Concealment.md) | `concealedSamples`, `silentConcealedSamples`, `concealmentEvents`, `totalSamplesReceived` | How the decoder hides missing audio |
| [SamplesInsertedRemoved](SamplesInsertedRemoved.md) | `insertedSamplesForDeceleration`, `removedSamplesForAcceleration` | Playout speed adjustments by the jitter buffer |
| [PacketLoss](PacketLoss.md) | `packetsLost`, `packetsReceived` | Network-level packet loss detection |
| [PacketDiscard](PacketDiscard.md) | `packetsDiscarded` | Jitter buffer discard behavior |
| [AudioLevel](AudioLevel.md) | `audioLevel`, `totalAudioEnergy`, `totalSamplesDuration` | Audio energy and silence detection |

## How to Collect These Stats

```javascript
const pc = new RTCPeerConnection();
// ... add tracks, connect ...

setInterval(async () => {
  const stats = await pc.getStats();
  stats.forEach(report => {
    if (report.type === 'inbound-rtp' && report.kind === 'audio') {
      console.log(report);
    }
  });
}, 1000);
```

All inbound audio metrics live on the `RTCInboundRtpStreamStats` dictionary (`report.type === 'inbound-rtp'` and `report.kind === 'audio'`).

## Scenarios

Each document covers the stat behavior under these scenarios:

1. **Normal conditions** -- low jitter, no loss
2. **High jitter** -- variable network delay
3. **Burst packet loss** -- consecutive packets dropped
4. **Random packet loss** -- sporadic drops
5. **Network congestion** -- sustained throughput pressure
6. **Codec change / renegotiation** -- mid-call parameter changes
