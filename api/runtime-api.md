# Runtime API Reference

Audio Atlas exposes a code-level API in `AudioAtlas.Runtime.*`. Add `AudioAtlas.Runtime` to your assembly's references to use it.

All APIs are gated behind `#if UNITY_EDITOR || AUDIOATLAS_DEBUG`. In release builds without the define, all tracking code compiles away completely.

---

## AudioRegistry

Static read-only access to live audio state. All properties return pre-allocated values — never null, never allocating.

```csharp
using AudioAtlas.Runtime.Core;
```

### Source Data

```csharp
int count = AudioRegistry.ActiveSourceCount;

SourceSnapshot[] snapshots = AudioRegistry.SourceSnapshots;
for (int i = 0; i < AudioRegistry.ActiveSourceCount; i++)
{
    ref var snap = ref snapshots[i];
    Debug.Log($"{snap.GameObjectName}  vol={snap.Volume:F2}  dist={snap.Distance:F1}");
}

int selectedID = AudioRegistry.SelectedSourceID;  // -1 = no selection
```

### Mixer Data

```csharp
MixerGroupSnapshot[] mixerSnaps = AudioRegistry.MixerSnapshots;
// Each entry: GroupName, Volume, IsActive, ParentName
```

### Event Ring Buffer

```csharp
AudioEvent[] events = AudioRegistry.EventRingBuffer;
int writeIdx = AudioRegistry.EventWriteIndex;

int len = events.Length;
for (int i = 0; i < Mathf.Min(10, writeIdx); i++)
{
    int slot = (writeIdx - 1 - i + len) % len;
    var evt = events[slot];
    Debug.Log($"{evt.EventType}  {evt.ClipName}  t={evt.Timestamp:F2}");
}
```

### Health Score

```csharp
AudioHealthScore score = AudioRegistry.HealthScore;
if (score.IsValid)
{
    // Grade: 'A' 'B' 'C' 'D' 'F'
    // Score: 0-100 float
    // Sub-scores: VoiceScore(0-25), CpuScore(0-25), DesignScore(0-25),
    //             StealScore(0-15), MixerScore(0-10)
    // Delta: change vs ~5s ago
    // TrendArrow: 'up' 'down' 'right'
    Debug.Log($"Health: {score.Grade} ({score.Score:F0})  {score.TrendArrow}  D{score.Delta:+0.0;-0.0}");
}
```

### Intelligence Insights

```csharp
InsightCard[] insights = AudioRegistry.IntelligenceInsights;
int count = AudioRegistry.IntelligenceInsightCount;
for (int i = 0; i < count; i++)
{
    var card = insights[i];
    Debug.Log($"[{card.Severity}] {card.Title}: {card.Summary}");
}
```

### Spatial State

```csharp
Vector3 listenerPos  = AudioRegistry.ListenerPosition;
SignalPath path      = AudioRegistry.CurrentSignalPath;
HeatmapData heatmap  = AudioRegistry.CurrentHeatmap;
LatencyStats latency = AudioRegistry.LatencyData;
```

### Lifecycle

```csharp
bool ready = AudioRegistry.IsInitialized;  // false in Edit Mode and before boot

// Thread-safe ring buffer slot acquisition
int eventSlot = AudioRegistry.ClaimEventSlot();
int stealSlot = AudioRegistry.ClaimStealSlot();
```

---

## EventBus

```csharp
using AudioAtlas.Runtime.Core;
```

See [EventBus Reference](../core-systems/event-bus.md) for the complete event catalogue.

```csharp
EventBus.Subscribe(AudioAtlasEvent.HealthScoreUpdated, OnHealthUpdate);
EventBus.Raise(AudioAtlasEvent.HealthScoreUpdated, (int)(score.Score * 100));
EventBus.Unsubscribe(AudioAtlasEvent.HealthScoreUpdated, OnHealthUpdate);
EventBus.Clear();  // nulls all slots — called on session end
```

---

## AudioAtlasRuntime

High-level playback helpers. 32-slot pooled `AudioSource` array. 32-slot fade registry. Zero heap allocation per call.

```csharp
using AudioAtlas.Runtime.Engine;
```

### Pooled One-Shot Playback

```csharp
// 3D one-shot at world position
AudioSource src = AudioAtlasRuntime.PlayOneShotAuto(
    clip,
    worldPos,
    mixerGroup: null,
    volume: 1f,
    pitch: 1f
);

// 2D one-shot (spatialBlend forced to 0)
AudioSource src2d = AudioAtlasRuntime.PlayOneShotUI(clip, volume: 0.8f, pitch: 1f);
```

Pool capacity: 32 sources. If all slots are occupied, the oldest non-looping source is evicted.

### Volume Fades

```csharp
AudioAtlasRuntime.FadeVolume(source, targetVolume: 0.5f, duration: 1.0f);
AudioAtlasRuntime.FadeOutAndStop(source, duration: 0.8f);
```

Fade registry: 32 simultaneous fades. Overflow is silently dropped.

### Procedural Clip Generation

```csharp
AudioClip click = AudioAtlasRuntime.GetProceduralClick(frequency: 440f);
AudioClip drone = AudioAtlasRuntime.GenerateAmbientTone(freq: 80f, durationSec: 2f);
AudioClip burst = AudioAtlasRuntime.GenerateBurstSFX(freq: 220f, decayRate: 6f);
```

Procedural clips are cached by frequency hash — no re-allocation on repeat calls.

---

## SourceSnapshot Struct

```csharp
public struct SourceSnapshot
{
    public int    InstanceID;
    public string GameObjectName;    // cached — not re-fetched per frame
    public string ClipName;
    public float  Volume;
    public float  Pitch;
    public float  SpatialBlend;      // 0 = 2D, 1 = full 3D
    public float  Distance;          // world units from AudioListener
    public int    Priority;          // 0 = highest, 256 = lowest
    public string MixerGroupName;    // empty if unassigned
    public bool   IsPlaying;
    public bool   IsLooping;
    public bool   IsPaused;
    public float  Time;              // current playback position in seconds
}
```

---

## AudioHealthScore Struct

```csharp
public struct AudioHealthScore
{
    public float  Score;        // 0–100 composite
    public char   Grade;        // 'A' 'B' 'C' 'D' 'F'
    public string StatusLabel;  // "Excellent" "Good" "Fair" "Poor" "Critical"
    public float  VoiceScore;   // 0–25
    public float  CpuScore;     // 0–25
    public float  DesignScore;  // 0–25
    public float  StealScore;   // 0–15
    public float  MixerScore;   // 0–10
    public float  Delta;        // score change vs ~5s ago
    public char   TrendArrow;   // up / down / right
    public float  ComputedAt;   // Time.realtimeSinceStartup
    public bool   IsValid;      // false before first health tick
}
```

---

## Compile Guards

```csharp
#if AUDIOATLAS_DEBUG
    AudioAtlasRuntime.PlayOneShotAuto(clip, transform.position);
#else
    AudioSource.PlayClipAtPoint(clip, transform.position);
#endif
```

`AudioAtlasRuntime` is only available when `UNITY_EDITOR || AUDIOATLAS_DEBUG` is defined — calling it outside that context is a compile error.

---

## ScanCoordinator (Editor)

```csharp
using AudioAtlas.Editor.Scan;

ScanCoordinator.RunAsync();
ScanCoordinator.Cancel();
var result = ScanCoordinator.LastResult;
ScanCoordinator.OnScanComplete += OnDone;
ScanResult result = ScanCoordinator.RunBlocking();  // CI only
```

---

[← Configuration](../configuration/settings-reference.md) · [CI Integration →](../ci/ci-integration.md)
