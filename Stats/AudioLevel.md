# Audio Level & Energy Stats

Stats dictionary: **RTCInboundRtpStreamStats** (audio only)

These metrics track the loudness and energy of the received audio. They are useful for detecting silence, voice activity, and measuring overall audio levels without decoding the audio yourself.

---

## Metrics

### `audioLevel` (double, 0.0 to 1.0)

The audio level of the received track at the time of the stats report. This is an **instantaneous** value (not cumulative).

- Linear scale: `1.0` = 0 dBov (maximum level), `0.0` = silence.
- Derived from RFC 6464 (Client-to-Mixer Audio Level Indication) when the header extension is present, or computed locally from decoded audio otherwise.
- Useful for real-time VU meter display or active speaker detection.

Conversion to dBov:
```
dBov = 20 * Math.log10(audioLevel)
```

| audioLevel | dBov | Rough meaning |
|------------|------|---------------|
| 1.0 | 0 dBov | Maximum / clipping |
| 0.1 | -20 dBov | Normal speech |
| 0.01 | -40 dBov | Quiet speech / background |
| 0.001 | -60 dBov | Near silence |
| 0.0 | -Infinity | Digital silence |

### `totalAudioEnergy` (double)

Cumulative audio energy. Computed as the sum of energy contributions for each audio sample duration:

```
totalAudioEnergy += duration * (rmsLevel)^2
```

where `rmsLevel` is the RMS audio level for that duration (linear, 0 to 1).

- Monotonically increasing.
- Use deltas to compute average energy over an interval.

### `totalSamplesDuration` (double, seconds)

Cumulative duration of all received samples in seconds. Monotonically increasing.

- Grows at real-time rate (1 second per second of audio).
- Used as the denominator when computing average energy.

---

## Derived Calculations

| Metric | Formula | Meaning |
|--------|---------|---------|
| Average RMS level (interval) | `sqrt(dEnergy / dDuration)` | RMS audio level over the measurement interval |
| Average dBov (interval) | `20 * log10(avgRmsLevel)` | Average loudness in dBov |
| Is silent | `audioLevel < 0.001` or `avgRmsLevel < 0.001` | Simple silence detection |

---

## Scenario Behavior

### Normal Speech

- `audioLevel` fluctuates between ~0.02 and ~0.3 during active speech.
- `totalAudioEnergy` grows in proportion to how loud the speaker is.
- `totalSamplesDuration` grows at 1.0 per second.

### Silence / Muted Microphone

- `audioLevel` drops to 0 (or near 0).
- `totalAudioEnergy` growth stalls.
- `totalSamplesDuration` continues growing.
- Average RMS level drops toward 0.

### DTX (Discontinuous Transmission)

- During silence, the sender may stop sending audio packets (or send only SID frames).
- `audioLevel` reflects the comfort noise level (very low but not zero).
- `totalSamplesDuration` still grows from concealed/comfort-noise samples.
- `totalAudioEnergy` grows very slowly.

### Packet Loss During Speech

- `audioLevel` may briefly drop if PLC produces lower-energy output than the original.
- `totalAudioEnergy` slightly underestimates true energy during loss.
- The concealment stats (see [Concealment](Concealment.md)) are more reliable for detecting loss than audio level.

### Multiple Speakers / Mixed Audio

- If the audio track is a mix (e.g., conference server), `audioLevel` reflects the combined level.
- Per-participant levels require SSRC-level stats or the CSRC audio level header extension.

---

## Code: Audio Level Monitoring

```javascript
let prev = null;

setInterval(async () => {
  const stats = await pc.getStats();
  stats.forEach(report => {
    if (report.type === 'inbound-rtp' && report.kind === 'audio') {
      // Instantaneous level
      const dBov = report.audioLevel > 0
        ? (20 * Math.log10(report.audioLevel)).toFixed(1)
        : '-Infinity';
      console.log(`Audio level: ${report.audioLevel.toFixed(4)} (${dBov} dBov)`);

      // Interval average
      if (prev) {
        const dEnergy = report.totalAudioEnergy - prev.totalAudioEnergy;
        const dDuration = report.totalSamplesDuration - prev.totalSamplesDuration;
        if (dDuration > 0) {
          const rms = Math.sqrt(dEnergy / dDuration);
          const avgDBov = rms > 0 ? (20 * Math.log10(rms)).toFixed(1) : '-Infinity';
          console.log(`Avg RMS: ${rms.toFixed(4)} (${avgDBov} dBov)`);
        }
      }
      prev = report;
    }
  });
}, 1000);
```

---

## Use Cases

| Use Case | Approach |
|----------|----------|
| Active speaker detection | Compare `audioLevel` across participants; highest wins |
| VU meter | Display `audioLevel` directly (scale 0-1 to UI range) |
| Silence detection | `audioLevel < threshold` for N consecutive readings |
| Call quality logging | Record `totalAudioEnergy / totalSamplesDuration` per interval |
| Mute detection | `audioLevel === 0` consistently (distinguishes from DTX which has low but non-zero level) |
