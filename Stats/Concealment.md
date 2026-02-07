# Concealment Stats

Stats dictionary: **RTCInboundRtpStreamStats** (audio only)

When audio packets are lost or arrive too late for playout, the decoder must **conceal** the gap by synthesizing replacement samples. These stats track how much concealment is happening and what kind.

---

## Metrics

### `totalSamplesReceived` (unsigned long long)

Total number of audio samples received on this RTP stream, including both normal and concealed samples. This is the baseline denominator for concealment ratios.

### `concealedSamples` (unsigned long long)

Total number of samples that were **synthesized** (concealed) rather than decoded from received packets. Concealment methods include:

- **Packet Loss Concealment (PLC):** The decoder extrapolates from previous audio to fill the gap. Produces non-silent audio that approximates what was lost.
- **Comfort Noise (CN/CNG):** Generates low-level background noise when the sender uses DTX (Discontinuous Transmission) and sends SID (Silence Insertion Descriptor) frames.
- **Silence insertion:** Zero samples inserted as a last resort.

### `silentConcealedSamples` (unsigned long long)

Subset of `concealedSamples` that were silent (zero-value) or comfort noise. This helps distinguish between:

- **Active PLC** (`concealedSamples - silentConcealedSamples`): The decoder generated audible filler — indicates real packet loss during speech.
- **Silent concealment** (`silentConcealedSamples`): Silence or comfort noise — may just be DTX behavior, not necessarily a quality problem.

### `concealmentEvents` (unsigned long long)

Incremented each time a concealed sample **follows a non-concealed sample**. Counts the number of distinct concealment episodes, not the total concealed duration.

- A single burst of loss = 1 concealment event regardless of how many samples are concealed.
- Frequent short events are perceptually worse than rare long ones (each transition is audible).

---

## Derived Calculations

| Metric | Formula | Meaning |
|--------|---------|---------|
| Concealment ratio | `concealedSamples / totalSamplesReceived` | Fraction of audio that was synthesized |
| Active PLC ratio | `(concealedSamples - silentConcealedSamples) / totalSamplesReceived` | Fraction of audio where real speech was lost |
| Silent concealment ratio | `silentConcealedSamples / totalSamplesReceived` | Fraction that was comfort noise or silence |
| Avg concealment duration | `concealedSamples / concealmentEvents` (in samples) | Average length of each concealment episode |
| Concealment event rate | `concealmentEvents / totalSamplesDuration` | Events per second of audio |

---

## Scenario Behavior

### Normal Conditions

- `concealedSamples` stays near zero (or only reflects DTX comfort noise).
- `silentConcealedSamples` may grow if the remote side uses DTX/VAD — this is expected and not a quality issue.
- `concealmentEvents` is very low.

### Burst Packet Loss

- `concealedSamples` jumps by a large amount per burst (e.g., 960 samples for a 20 ms Opus frame at 48 kHz).
- `concealmentEvents` increments by 1 per burst regardless of burst length.
- `silentConcealedSamples` stays low if the loss occurs during active speech — PLC produces non-silent output.
- Average concealment duration is high.

### Random Packet Loss (e.g., 5% uniform)

- `concealedSamples` grows steadily.
- `concealmentEvents` grows proportionally — each isolated loss is a separate event.
- Perceptual quality may be worse than burst loss at the same overall rate because every event is a separate glitch.
- Average concealment duration is short (typically one frame per event).

### DTX / Silence Suppression

- `silentConcealedSamples` grows during silence periods.
- `concealedSamples` grows at roughly the same rate.
- `concealmentEvents` may show periodic increments as VAD toggles.
- This is **normal behavior**, not an indication of poor quality.

### Codec Change / Renegotiation

- Concealment stats continue accumulating across codec changes — they are not reset.
- A brief concealment episode may occur at the transition point while the new codec initializes.

---

## Code: Monitoring Concealment

```javascript
let prev = null;

setInterval(async () => {
  const stats = await pc.getStats();
  stats.forEach(report => {
    if (report.type === 'inbound-rtp' && report.kind === 'audio') {
      if (prev) {
        const dTotal = report.totalSamplesReceived - prev.totalSamplesReceived;
        const dConcealed = report.concealedSamples - prev.concealedSamples;
        const dSilent = report.silentConcealedSamples - prev.silentConcealedSamples;
        const dEvents = report.concealmentEvents - prev.concealmentEvents;

        if (dTotal > 0) {
          const concealPct = ((dConcealed / dTotal) * 100).toFixed(2);
          const plcPct = (((dConcealed - dSilent) / dTotal) * 100).toFixed(2);
          console.log(`Concealment: ${concealPct}% (active PLC: ${plcPct}%), events: ${dEvents}`);
        }
      }
      prev = report;
    }
  });
}, 1000);
```

---

## Quality Thresholds (Rules of Thumb)

| Concealment Ratio | Quality Impact |
|-------------------|----------------|
| < 1% | Imperceptible |
| 1% - 3% | Occasionally audible, acceptable |
| 3% - 8% | Noticeable degradation |
| > 8% | Severe — investigate network or endpoint |

> These thresholds apply to **active PLC** (`concealedSamples - silentConcealedSamples`). Silent concealment from DTX should be excluded when evaluating quality.
