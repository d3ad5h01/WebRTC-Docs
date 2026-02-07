# Packet Loss Stats

Stats dictionary: **RTCReceivedRtpStreamStats** / **RTCInboundRtpStreamStats**

Packet loss is the most fundamental measure of network health for a WebRTC audio call. Lost packets directly cause concealment and degrade audio quality.

---

## Metrics

### `packetsReceived` (unsigned long long)

Total number of RTP packets received for this SSRC, including retransmissions. This is the primary "good packets" counter.

### `packetsLost` (long long)

Total number of RTP packets lost for this SSRC, calculated per [RFC 3550 Section 6.4.1](https://datatracker.ietf.org/doc/html/rfc3550#section-6.4.1).

How it is calculated:
- The receiver tracks the highest sequence number received and the total packets received.
- `packetsLost = expectedPackets - packetsReceived`
- `expectedPackets = highestSeqNum - firstSeqNum + 1`

Important properties:
- **Can be negative** if duplicate packets arrive (e.g., from retransmission).
- **Cumulative** — never reset during the session.
- **Includes late packets** that arrived after their playout deadline — they count as received (not lost), even though the jitter buffer may have discarded them.

### `jitter` (double, seconds)

Inter-arrival jitter as defined in RFC 3550. Not a packet loss metric directly, but strongly correlated — high jitter often precedes loss.

---

## Derived Calculations

| Metric | Formula | Meaning |
|--------|---------|---------|
| Total expected packets | `packetsReceived + packetsLost` | How many packets should have arrived |
| Cumulative loss rate | `packetsLost / (packetsReceived + packetsLost)` | Session-lifetime loss percentage |
| Interval loss rate | `dLost / (dReceived + dLost)` | Loss percentage over a measurement interval |
| Packets per second | `dReceived / intervalSeconds` | Packet rate (should match codec ptime) |

Where `dLost` and `dReceived` are deltas between two consecutive `getStats()` calls.

### Expected Packet Rate

For Opus audio at the default 20 ms ptime: **50 packets/second**. A deviation from this rate (after accounting for DTX) indicates loss or reordering.

---

## Scenario Behavior

### Normal Conditions

- `packetsLost` stays at 0 or very close to 0.
- `packetsReceived` grows at the expected rate (e.g., 50/s for 20 ms Opus).
- Cumulative loss rate is < 0.1%.

### Random Packet Loss (e.g., 2% uniform)

- `packetsLost` grows steadily at ~1 packet/s (at 50 pps).
- Interval loss rate hovers around the configured loss rate.
- Concealment events increase proportionally (see [Concealment](Concealment.md)).
- Quality impact depends on FEC: Opus in-band FEC can recover from ~5-10% random loss with minimal quality degradation.

### Burst Packet Loss

- `packetsLost` increases in jumps — flat between bursts, then a step up.
- Interval loss rate spikes during bursts, then drops to zero.
- Burst loss is **harder to conceal** than random loss because PLC has less context to extrapolate from during multi-frame gaps.
- `jitter` may not increase during burst loss (the surviving packets may still arrive on time).

### Network Congestion

- Gradual increase in loss rate as router buffers fill.
- Often accompanied by increasing `jitter`.
- `packetsReceived` rate drops below expected.
- Recovery may produce a burst of late-arriving packets (which count as received, not lost, but may be discarded by the jitter buffer — see [PacketDiscard](PacketDiscard.md)).

### Codec Change / Renegotiation

- Loss counters continue across codec changes — they are per-SSRC, not per-codec.
- If the SSRC changes (some implementations do this on renegotiation), a new stats object is created with counters starting at zero.

---

## Code: Computing Packet Loss Rate

```javascript
let prev = null;

setInterval(async () => {
  const stats = await pc.getStats();
  stats.forEach(report => {
    if (report.type === 'inbound-rtp' && report.kind === 'audio') {
      if (prev) {
        const dReceived = report.packetsReceived - prev.packetsReceived;
        const dLost = report.packetsLost - prev.packetsLost;
        const total = dReceived + dLost;

        if (total > 0) {
          const lossRate = ((dLost / total) * 100).toFixed(2);
          console.log(`Packet loss: ${lossRate}% (${dLost} lost / ${total} expected)`);
        }
      }
      prev = report;
    }
  });
}, 1000);
```

---

## Relationship Between Loss and Concealment

Not all lost packets result in concealment, and not all concealment comes from loss:

| Situation | packetsLost | concealedSamples |
|-----------|-------------|------------------|
| Packet lost during speech | Increases | Increases (PLC) |
| Packet lost during DTX silence | Increases | May increase (comfort noise) |
| Packet arrives too late | No change | Increases (PLC) — packet was received but useless |
| DTX silence (no loss) | No change | May increase (silent concealment) |

> **Key insight:** `packetsLost` measures network-layer loss. `concealedSamples` measures application-layer impact. Use both to get the full picture.
