# Intelligence Engine

`AudioIntelligenceEngine` runs inside `AudioAtlasBootstrap.Update()` on the main thread. It separates hot-path event callbacks (zero allocation) from its 1-second analysis tick (string operations allowed).

---

## Analysis Algorithms

### Clip Fire-Rate Analysis

**Goal:** Detect clips that fire too frequently — a sign of missing cooldown guards or spam-triggering event systems.

**Data structure:** Open-addressing hash table with linear probing. Capacity: 128 slots (must be power-of-2). Each slot stores:
- `_clipSlotKey[slot]` — name hash (0 = empty slot)
- `_clipSlotName[slot]` — name string, cached at first write
- `_clipFireTotal[slot]` — cumulative fire count this session
- `_clipFireRing[slot * FireWindowSize + head]` — ring of 12 recent fire timestamps
- `_clipFireHead[slot]` — ring write head
- `_clipFireFilled[slot]` — entries written (0–12)

**Algorithm (per tick):**
1. Walk the hash table (O(128))
2. For each populated slot, count fire timestamps within the last 1.0 second using the ring buffer
3. If count > `SpamThreshold` (10): emit a `ClipSpam` insight card

**Allocation contract:** Zero on the hot-path callback. The string name is cached on first encounter — subsequent lookups use the hash key only.

---

### Co-Occurrence Detection

**Goal:** Identify clip pairs that consistently fire together within 50ms — suggesting they should be a single compound clip, or that triggering logic is duplicated.

**Data structure:** 24-slot circular buffer of recent `(clipNameHash, timestamp, clipName)` entries.

**Algorithm (on each `AudioEventRecorded`):**
1. Write new event to the co-occurrence ring
2. Scan all other entries with `|timestamp - newTimestamp| < 50ms`
3. For each pair found, look up or insert into the co-hit table (32 slots, open addressing)
4. Increment co-hit count for this pair
5. At analysis tick: pairs with count ≥ 3 emit a `CoOccurrence` insight card

**Threshold:** 50ms window. Pairs co-firing 3 or more times in the session are surfaced.

---

### Steal Burst Detection

**Goal:** Detect temporal clustering of voice steals — a burst indicates the voice budget is systematically too tight for the game's audio density at certain moments.

**Data structure:** 16-slot ring buffer of recent steal timestamps.

**Algorithm (on each `VoiceStealDetected`):**
1. Write steal timestamp to the ring
2. At analysis tick: count entries within the last 500ms
3. If count ≥ 3: declare a burst, increment `_burstCountThisSession`, emit a `StealBurst` insight card

The burst insight includes: steal count, time window, and session-total burst count.

---

### Source Design Analysis

**Goal:** Surface `AudioSource` configuration problems that are not caught by the rule scanner but emerge from live runtime observation.

**Checks run at the 1-second analysis tick:**

| Check | Threshold | Insight severity |
|---|---|---|
| Source playing at volume 0 | `volume < 0.01f` | Medium — wastes voice slot |
| Source with default priority 128 and high volume | `volume > 0.7f && priority == 128` | Medium — steal risk |
| Source with no mixer group | `outputAudioMixerGroup == null` | Low — uncontrolled routing |
| Source with extreme pitch | `pitch < 0.1f \|\| pitch > 3f` | Low — artefact risk |
| Looping source with no distance bounds | `loop && maxDistance <= minDistance` | High — will never attenuate |

Design insights are deduplicated per source — the same source will not produce the same insight card twice per session.

---

## Health Score Composition

The health score is a float 0–100, composed from five weighted sub-scores. It is written to `AudioRegistry.HealthScore` and the event `HealthScoreUpdated` is raised every ~0.5 seconds.

### Sub-Score Formulas

**VoiceScore (0–25)**
```
ratio = activeVoices / max(1, voiceBudget)   // voiceBudget from config
if ratio <= VoiceUsageWarningRatio (0.8):     score = 25
if ratio <= 1.0:                              score = 25 × (1 - ratio) / 0.2
if ratio > 1.0:                               score = 0  (over budget)
```

**CpuScore (0–25)**
```
cpu = AudioRegistry.LatencyData.AudioCpuPercent
if cpu <= AudioCpuWarningPercent (8.0):       score = 25
if cpu <= AudioCpuCriticalPercent (15.0):     score = 25 × (1 - (cpu - 8) / 7)
if cpu > 15.0:                                score = 0
```

**DesignScore (0–25)**
```
// Count sources with design issues (no mixer group, bad spatial, default priority on loud)
badSources = sources with ≥1 design issue
ratio = badSources / max(1, totalTrackedSources)
score = 25 × max(0, 1 - ratio × 2)   // 50% bad sources = 0 score
```

**StealScore (0–15)**
```
stealRate = stealsThisSession / max(1, eventsThisSession)
burstPenalty = min(5, _burstCountThisSession)   // 1 point per burst, max 5
baseScore = 15 × max(0, 1 - stealRate × 5)
score = max(0, baseScore - burstPenalty)
```

**MixerScore (0–10)**
```
unroutedSources = sources with outputAudioMixerGroup == null
ratio = unroutedSources / max(1, totalTrackedSources)
score = 10 × max(0, 1 - ratio × 2)
```

### Grade Thresholds

| Score | Grade | StatusLabel |
|---|---|---|
| ≥ 90 | A | Excellent |
| ≥ 75 | B | Good |
| ≥ 60 | C | Fair |
| ≥ 40 | D | Poor |
| < 40 | F | Critical |

### Trend Calculation

A circular buffer of the last 10 scores (sampled every tick) is maintained. The delta is computed as:
```
delta = currentScore - scoreFrom5SecondsAgo
TrendArrow: '↑' if delta > 1.0, '↓' if delta < -1.0, '→' otherwise
```

---

## Insight Pool

Pre-allocated array of 32 `InsightCard` structs in `_insightPool`. At analysis tick, new cards are written into the pool slots (reusing memory from previous ticks). After all checks complete, `AudioRegistry.IntelligenceInsights` is updated atomically and `IntelligenceInsightAvailable` is raised.

`IntelligenceAdvisor` subscribes to `IntelligenceInsightAvailable` and scans the pool for any newly Critical-severity card. If found, it raises `CriticalInsightDetected` — a separate high-priority event for custom subscriber reactions.

---

## Subscription Example

```csharp
using AudioAtlas.Runtime.Core;
using AudioAtlas.Runtime.Models;

void Awake()
{
    EventBus.Subscribe(AudioAtlasEvent.HealthScoreUpdated,          OnHealthUpdate);
    EventBus.Subscribe(AudioAtlasEvent.IntelligenceInsightAvailable, OnInsightsReady);
    EventBus.Subscribe(AudioAtlasEvent.CriticalInsightDetected,     OnCritical);
}

void OnDestroy()
{
    EventBus.Unsubscribe(AudioAtlasEvent.HealthScoreUpdated,          OnHealthUpdate);
    EventBus.Unsubscribe(AudioAtlasEvent.IntelligenceInsightAvailable, OnInsightsReady);
    EventBus.Unsubscribe(AudioAtlasEvent.CriticalInsightDetected,     OnCritical);
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
    // A new Critical insight appeared — surface it prominently
    notificationBanner.Show("Audio warning — open Audio Atlas Insights panel");
}
```

---

[← EventBus Reference](event-bus.md) · [Back to Docs](../../README.md)
