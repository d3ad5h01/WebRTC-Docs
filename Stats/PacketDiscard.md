# Packet Discard Stats

Stats dictionary: **RTCInboundRtpStreamStats**

Packets that arrive but are **too late** (or too early) for the jitter buffer to use are discarded. These packets count as `packetsReceived` (not `packetsLost`) but still cause concealment because they miss their playout deadline.

---

## Metrics

### `packetsDiscarded` (unsigned long long)

Cumulative number of RTP packets discarded by the jitter buffer. Reasons for discard:

1. **Late arrival** — The packet arrived after its playout time had passed. The jitter buffer already concealed the gap and moved on.
2. **Too early / duplicate** — The packet's sequence number falls outside the buffer's current window.
3. **Buffer overflow** — The jitter buffer is full and drops the oldest or lowest-priority packet (rare in audio).

---

## Relationship to Other Stats

```
Packets sent by remote
  |
  +--> packetsLost        (never arrived)
  |
  +--> packetsReceived    (arrived at the receiver)
        |
        +--> packetsDiscarded   (arrived but unusable)
        |
        +--> decoded / played out (arrived and used)
```

- `packetsLost` = network lost (never arrived).
- `packetsDiscarded` = arrived but jitter buffer rejected.
- Both cause concealment, but only `packetsLost` is visible in RTCP reports sent back to the sender.

---

## Derived Calculations

| Metric | Formula | Meaning |
|--------|---------|---------|
| Discard rate | `dDiscarded / dReceived` | Fraction of received packets that were useless |
| Effective loss rate | `(dLost + dDiscarded) / (dReceived + dLost)` | True unavailability rate (network + late) |
| Late arrival ratio | `dDiscarded / (dLost + dDiscarded)` | What fraction of "missing" audio was late vs truly lost |

---

## Scenario Behavior

### Normal Conditions

- `packetsDiscarded` stays at 0.
- All received packets arrive within the jitter buffer window.

### High Jitter (Buffer Too Small)

- `packetsDiscarded` increases when delay spikes exceed the current jitter buffer target.
- The jitter buffer adapts by increasing `jitterBufferTargetDelay`, which reduces future discards.
- During the adaptation window, discards spike then settle.
- This is the most common cause of packet discard.

### Burst Packet Loss Followed by Recovery

- Packets arrive in a burst after a gap. Some may have timestamps that have already passed.
- `packetsDiscarded` may increase for the late arrivals in the burst.
- `packetsLost` may initially increase then partially decrease (if late packets are counted as received in the next RTCP interval).

### Network Route Change / Path Switch

- A route change can cause a sudden shift in one-way delay.
- If the new path has higher delay, packets in transit on the old path may arrive "on time" but the next batch arrives late.
- Spike in `packetsDiscarded` until the jitter buffer adapts to the new delay baseline.

### Network Congestion

- Congestion causes both delay increase and loss.
- `packetsDiscarded` grows alongside `packetsLost`.
- The effective loss rate (`lost + discarded`) gives the true picture of unavailable audio.

---

## Code: Monitoring Packet Discards

```javascript
let prev = null;

setInterval(async () => {
  const stats = await pc.getStats();
  stats.forEach(report => {
    if (report.type === 'inbound-rtp' && report.kind === 'audio') {
      if (prev) {
        const dDiscarded = report.packetsDiscarded - prev.packetsDiscarded;
        const dReceived = report.packetsReceived - prev.packetsReceived;
        const dLost = report.packetsLost - prev.packetsLost;

        if (dReceived > 0) {
          const discardRate = ((dDiscarded / dReceived) * 100).toFixed(2);
          const effectiveLoss = (((dLost + dDiscarded) / (dReceived + dLost)) * 100).toFixed(2);
          console.log(`Discarded: ${discardRate}%, Effective loss: ${effectiveLoss}%`);
        }
      }
      prev = report;
    }
  });
}, 1000);
```

---

## Diagnosing Discard vs Loss

| Symptom | Likely Cause |
|---------|--------------|
| High `packetsLost`, low `packetsDiscarded` | Network is dropping packets (congestion, routing) |
| Low `packetsLost`, high `packetsDiscarded` | Packets arrive too late — jitter buffer target is too low, or delay spikes exceed tolerance |
| Both high | Severe network problem — congestion causing both loss and delay |
| Both zero | Healthy network |

> **Actionable:** If `packetsDiscarded` is consistently high, consider setting a minimum jitter buffer delay via `RTCRtpReceiver.jitterBufferTarget` (if supported) to give late packets more time to arrive.
