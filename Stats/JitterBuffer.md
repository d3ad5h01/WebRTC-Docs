# Jitter Buffer Stats

Stats dictionary: **RTCInboundRtpStreamStats** (audio only)

The jitter buffer absorbs variable network delay so the audio playout remains smooth. These counters let you measure how large the buffer is, how long samples sit inside it, and whether the target delay is being met.

---

## Metrics

### `jitterBufferDelay` (double, seconds)

Cumulative sum of time each audio sample spent in the jitter buffer, measured from arrival (ingest) to emission (playout). Divide by `jitterBufferEmittedCount` to get the **average buffer delay**:

```
avgBufferDelay = jitterBufferDelay / jitterBufferEmittedCount
```

- Increases monotonically.
- A rising average means the buffer is growing to absorb more jitter.

### `jitterBufferTargetDelay` (double, seconds)

Cumulative sum of the *target* delay at each emission. The target is what the jitter buffer algorithm thinks is optimal given current conditions. Divide by `jitterBufferEmittedCount` to get the **average target delay**:

```
avgTargetDelay = jitterBufferTargetDelay / jitterBufferEmittedCount
```

### `jitterBufferEmittedCount` (unsigned long long)

Total number of audio samples that have exited the jitter buffer. Used as the denominator when computing averages from the cumulative delay counters above.

### `jitterBufferMinimumDelay` (double, seconds)

Cumulative sum of the minimum delay based purely on network characteristics, before any application-level adjustments (e.g., `playout-delay` header extension). Gives a baseline for how much delay the network alone requires.

---

## Derived Calculations

| Metric | Formula |
|--------|---------|
| Average jitter buffer delay | `jitterBufferDelay / jitterBufferEmittedCount` |
| Average target delay | `jitterBufferTargetDelay / jitterBufferEmittedCount` |
| Average minimum delay | `jitterBufferMinimumDelay / jitterBufferEmittedCount` |
| Buffer overshoot | `avgBufferDelay - avgTargetDelay` |

---

## Scenario Behavior

### Normal Conditions (low jitter, no loss)

- `avgBufferDelay` stays small and stable (typically 20-60 ms for audio).
- `avgTargetDelay` converges to a steady value close to `avgBufferDelay`.
- `jitterBufferEmittedCount` grows linearly at the sample rate (e.g., 48000/s for Opus at 48 kHz).

### High Jitter

- `jitterBufferTargetDelay` grows as the algorithm increases the target to absorb variance.
- `jitterBufferDelay` follows the target upward.
- You may see the buffer overshoot (`avgDelay > avgTarget`) during transient spikes before the algorithm catches up.
- More samples are inserted for deceleration (see [SamplesInsertedRemoved](SamplesInsertedRemoved.md)).

### Burst Packet Loss

- The jitter buffer cannot emit real samples during the gap, so `jitterBufferEmittedCount` may stall or emit concealed samples.
- After the burst, the target delay may spike upward as a protective measure.
- Buffer delay temporarily rises if late-arriving packets queue up after recovery.

### Network Congestion

- Sustained increase in both `jitterBufferDelay` and `jitterBufferTargetDelay`.
- If congestion clears, the target gradually decreases (the algorithm decays the estimate).
- Samples removed for acceleration (see [SamplesInsertedRemoved](SamplesInsertedRemoved.md)) during the drain-down phase.

---

## Code: Reading Jitter Buffer Stats

```javascript
let prev = null;

setInterval(async () => {
  const stats = await pc.getStats();
  stats.forEach(report => {
    if (report.type === 'inbound-rtp' && report.kind === 'audio') {
      if (prev) {
        const dDelay = report.jitterBufferDelay - prev.jitterBufferDelay;
        const dCount = report.jitterBufferEmittedCount - prev.jitterBufferEmittedCount;
        if (dCount > 0) {
          const avgDelayMs = (dDelay / dCount) * 1000;
          console.log(`Avg jitter buffer delay (interval): ${avgDelayMs.toFixed(1)} ms`);
        }
      }
      prev = report;
    }
  });
}, 1000);
```

> **Tip:** Always compute deltas between two consecutive `getStats()` snapshots to get interval-level averages rather than session-lifetime averages.
