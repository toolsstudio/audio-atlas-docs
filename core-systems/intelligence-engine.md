# Intelligence Engine

`AudioIntelligenceEngine` runs inside `AudioAtlasBootstrap.Update()` on the main thread. It separates hot-path event callbacks (zero allocation) from its 1-second analysis tick (string operations allowed).

---

## Analysis Algorithms

### Clip Fire-Rate Analysis

**Goal:** Detect clips firing too frequently — a sign of missing cooldown guards or spam-triggering event systems.

**Data structure:** Open-addressing hash table with linear probing. Capacity: 128 slots. Each slot stores a name hash key, cached name string, cumulative fire count, a 12-entry timestamp ring, and ring head.

**Algorithm:** At each 1-second tick, walk the table. For each slot, count fire timestamps within the last 1.0 second. If count > `SpamThreshold` (10), emit a `ClipSpam` insight card.

**Allocation contract:** Zero on the hot-path callback. The string name is cached on first encounter — subsequent lookups use only the hash key.

---

### Co-Occurrence Detection

**Goal:** Identify clip pairs that consistently fire together within 50ms — suggesting a compound clip opportunity or duplicated trigger logic.

**Data structure:** 24-slot circular buffer of recent `(clipNameHash, timestamp, clipName)` entries.

**Algorithm:** On each `AudioEventRecorded`, write the new event to the ring and scan all other entries within a 50ms window. Track co-hit pairs in a 32-slot open-addressing table. At analysis tick, pairs with count ≥ 3 emit a `CoOccurrence` insight card.

---

### Steal Burst Detection

**Goal:** Detect temporal clustering of voice steals — a burst indicates the voice budget is systematically too tight at certain moments.

**Data structure:** 16-slot ring buffer of recent steal timestamps.

**Algorithm:** On each `VoiceStealDetected`, write the timestamp to the ring. At analysis tick, count entries within the last 500ms. If count ≥ 3, declare a burst and emit a `StealBurst` insight card with steal count, time window, and session-total burst count.

---

### Source Design Analysis

Checks run at the 1-second tick:

| Check | Threshold | Insight severity |
|---|---|---|
| Source playing at volume 0 | `volume < 0.01f` | Medium |
| Default priority on high-volume source | `volume > 0.7f && priority == 128` | Medium |
| Source with no mixer group | `outputAudioMixerGroup == null` | Low |
| Extreme pitch | `pitch < 0.1f \|\| pitch > 3f` | Low |
| Looping source with no distance bounds | `loop && maxDistance <= minDistance` | High |

Design insights are deduplicated per source — the same source will not produce the same card twice per session.

---

## Health Score Composition

The health score is a float 0–100, composed from five weighted sub-scores. It is written to `AudioRegistry.HealthScore` and `HealthScoreUpdated` is raised every ~0.5 seconds.

### Sub-Score Formulas

**VoiceScore (0–25)**
```
ratio = activeVoices / max(1, voiceBudget)
if ratio <= 0.8:   score = 25
if ratio <= 1.0:   score = 25 × (1 - ratio) / 0.2
if ratio > 1.0:    score = 0
```

**CpuScore (0–25)**
```
cpu = AudioRegistry.LatencyData.AudioCpuPercent
if cpu <= 8.0:     score = 25
if cpu <= 15.0:    score = 25 × (1 - (cpu - 8) / 7)
if cpu > 15.0:     score = 0
```

**DesignScore (0–25)**
```
ratio = badSources / max(1, totalTrackedSources)
score = 25 × max(0, 1 - ratio × 2)
```

**StealScore (0–15)**
```
stealRate = stealsThisSession / max(1, eventsThisSession)
burstPenalty = min(5, _burstCountThisSession)
score = max(0, 15 × max(0, 1 - stealRate × 5) - burstPenalty)
```

**MixerScore (0–10)**
```
ratio = unroutedSources / max(1, totalTrackedSources)
score = 10 × max(0, 1 - ratio × 2)
```

### Grade Thresholds

| Score | Grade | Status |
|---|---|---|
| ≥ 90 | A | Excellent |
| ≥ 75 | B | Good |
| ≥ 60 | C | Fair |
| ≥ 40 | D | Poor |
| < 40 | F | Critical |

---

## Insight Pool

Pre-allocated array of 32 `InsightCard` structs in `_insightPool`. At each tick, new cards are written into pool slots, reusing memory from previous ticks. After all checks complete, `AudioRegistry.IntelligenceInsights` is updated atomically and `IntelligenceInsightAvailable` is raised.

`IntelligenceAdvisor` subscribes to `IntelligenceInsightAvailable` and raises `CriticalInsightDetected` when any newly Critical-severity card is found.

---

## Subscription Example

```csharp
void Awake()
{
    EventBus.Subscribe(AudioAtlasEvent.HealthScoreUpdated,           OnHealthUpdate);
    EventBus.Subscribe(AudioAtlasEvent.IntelligenceInsightAvailable, OnInsightsReady);
    EventBus.Subscribe(AudioAtlasEvent.CriticalInsightDetected,      OnCritical);
}

void OnDestroy()
{
    EventBus.Unsubscribe(AudioAtlasEvent.HealthScoreUpdated,           OnHealthUpdate);
    EventBus.Unsubscribe(AudioAtlasEvent.IntelligenceInsightAvailable, OnInsightsReady);
    EventBus.Unsubscribe(AudioAtlasEvent.CriticalInsightDetected,      OnCritical);
}

private void OnHealthUpdate(int _)
{
    var score = AudioRegistry.HealthScore;
    if (!score.IsValid) return;
    hudAudioGrade.text = $"{score.Grade} ({score.Score:F0})";
}

private void OnInsightsReady(int count)
{
    var insights = AudioRegistry.IntelligenceInsights;
    for (int i = 0; i < count; i++)
        Debug.Log($"[{insights[i].Severity}] {insights[i].Title}");
}

private void OnCritical(int _)
{
    notificationBanner.Show("Audio warning — open the Insights panel");
}
```

---

[← EventBus Reference](event-bus.md) · [Back to Docs](../README.md)
