# Samples Inserted / Removed (Playout Speed Adjustment)

Stats dictionary: **RTCInboundRtpStreamStats** (audio only)

The jitter buffer adjusts playout speed to track changes in network delay. When network delay increases it **slows down** playout (deceleration) by inserting extra samples. When delay decreases it **speeds up** playout (acceleration) by removing samples. These stats measure those adjustments.

---

## Metrics

### `insertedSamplesForDeceleration` (unsigned long long)

Total number of audio samples **inserted** to slow down playout. The jitter buffer stretches the audio to buy time when it senses that network delay is increasing and the buffer is getting thin.

How it works:
- The audio stretching algorithm duplicates or interpolates samples so the listener hears slightly slowed audio.
- The inserted samples are not "real" — they are synthesized to maintain continuity.
- Small amounts of insertion are inaudible. Large amounts produce a noticeable pitch/tempo artifact.

### `removedSamplesForAcceleration` (unsigned long long)

Total number of audio samples **removed** to speed up playout. The jitter buffer compresses the audio to drain excess buffered data when network delay drops.

How it works:
- The algorithm drops or merges samples so playout catches up to the reduced delay.
- Removes the excess buffer that is no longer needed.
- Like insertion, small amounts are inaudible; large amounts are perceptible.

---

## Relationship to Jitter Buffer

```
Network delay increases
  -> jitter buffer target grows
  -> playout must slow down to avoid underrun
  -> insertedSamplesForDeceleration increases

Network delay decreases
  -> jitter buffer target shrinks
  -> playout must speed up to drain excess buffer
  -> removedSamplesForAcceleration increases
```

These adjustments happen **continuously** in small increments. A well-behaved call has a roughly balanced ratio of inserted-to-removed samples over time.

---

## Derived Calculations

| Metric | Formula | Meaning |
|--------|---------|---------|
| Insertion rate | `dInserted / dTotalSamplesReceived` | Fraction of playout that was stretched |
| Removal rate | `dRemoved / dTotalSamplesReceived` | Fraction of playout that was compressed |
| Net adjustment | `dInserted - dRemoved` | Positive = buffer growing, negative = buffer draining |
| Total adjustment rate | `(dInserted + dRemoved) / dTotalSamplesReceived` | Overall playout disturbance |

---

## Scenario Behavior

### Normal Conditions

- Both counters grow slowly and at similar rates.
- The net adjustment is close to zero — small insertions and removals cancel out.
- Total adjustment rate is very low (< 1% of samples).

### High Jitter

- `insertedSamplesForDeceleration` grows faster — the buffer is frequently stretching to absorb delay spikes.
- Periodic bursts of `removedSamplesForAcceleration` as the buffer drains after spikes pass.
- Net adjustment oscillates but trends toward zero over time.

### Sustained Increase in Network Delay

- Steady growth of `insertedSamplesForDeceleration`.
- `removedSamplesForAcceleration` stays relatively flat.
- Net adjustment is consistently positive — the buffer is expanding.

### Sustained Decrease in Network Delay (Recovery)

- `removedSamplesForAcceleration` dominates.
- The buffer is draining back to a smaller target.
- Net adjustment is negative.

### Network Congestion

- Both counters may spike as the jitter buffer adapts aggressively.
- High total adjustment rate indicates the playout system is working hard to maintain continuity.
- Quality degrades when the adjustment rate exceeds ~5% of total samples — artifacts become audible.

---

## Code: Tracking Playout Adjustments

```javascript
let prev = null;

setInterval(async () => {
  const stats = await pc.getStats();
  stats.forEach(report => {
    if (report.type === 'inbound-rtp' && report.kind === 'audio') {
      if (prev) {
        const dInserted = report.insertedSamplesForDeceleration - prev.insertedSamplesForDeceleration;
        const dRemoved = report.removedSamplesForAcceleration - prev.removedSamplesForAcceleration;
        const dTotal = report.totalSamplesReceived - prev.totalSamplesReceived;

        if (dTotal > 0) {
          const insertPct = ((dInserted / dTotal) * 100).toFixed(2);
          const removePct = ((dRemoved / dTotal) * 100).toFixed(2);
          const net = dInserted - dRemoved;
          console.log(`Inserted: ${insertPct}%, Removed: ${removePct}%, Net: ${net} samples`);
        }
      }
      prev = report;
    }
  });
}, 1000);
```

---

## Important Notes

- These are **not** the same as concealment. Concealment replaces lost packets. Insertion/removal adjusts playout speed with real (or interpolated) audio.
- Both counters are monotonically increasing — always compute deltas.
- High insertion + high removal simultaneously means the network is oscillating and the buffer is hunting for a stable target.
