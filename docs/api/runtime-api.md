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
// Count of sources currently in the tracked list
int count = AudioRegistry.ActiveSourceCount;

// Pre-allocated snapshot array — read up to ActiveSourceCount entries
SourceSnapshot[] snapshots = AudioRegistry.SourceSnapshots;
for (int i = 0; i < AudioRegistry.ActiveSourceCount; i++)
{
    ref var snap = ref snapshots[i];
    Debug.Log($"{snap.GameObjectName}  vol={snap.Volume:F2}  dist={snap.Distance:F1}");
}

// Currently selected source (set by clicking in Live Sources)
// -1 = no selection
int selectedID = AudioRegistry.SelectedSourceID;
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

// Read the last N events (ring — wrap at length)
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
    // SubScores: VoiceScore(0-25), CpuScore(0-25), DesignScore(0-25),
    //            StealScore(0-15), MixerScore(0-10)
    // Delta: score change vs ~5s ago (positive = improving)
    // TrendArrow: '↑' '↓' '→'
    Debug.Log($"Health: {score.Grade} ({score.Score:F0})  {score.TrendArrow}  Δ{score.Delta:+0.0;-0.0}");
}
```

### Intelligence Insights

```csharp
InsightCard[] insights = AudioRegistry.IntelligenceInsights;
int count = AudioRegistry.IntelligenceInsightCount;
for (int i = 0; i < count; i++)
{
    var card = insights[i];
    // card.Title, card.Summary, card.Severity, card.Category
    Debug.Log($"[{card.Severity}] {card.Title}: {card.Summary}");
}
```

### Spatial State

```csharp
Vector3 listenerPos = AudioRegistry.ListenerPosition;
SignalPath path = AudioRegistry.CurrentSignalPath;
HeatmapData heatmap = AudioRegistry.CurrentHeatmap;
LatencyStats latency = AudioRegistry.LatencyData;
```

### Lifecycle

```csharp
bool ready = AudioRegistry.IsInitialized;  // false in Edit Mode and before boot

// Called automatically by Bootstrap — do not call manually
// AudioRegistry.Initialize(config);   // pre-allocates all arrays
// AudioRegistry.ClearSession();       // clears arrays, resets indices
// AudioRegistry.Shutdown();           // ClearSession + IsInitialized = false

// Thread-safe ring buffer slot acquisition (Interlocked.Increment)
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
// Handler signature: Action<int> — payload meaning is event-specific
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
// 3D one-shot at world position — pulls from pool, returns when done
AudioSource src = AudioAtlasRuntime.PlayOneShotAuto(
    clip,
    worldPos,
    mixerGroup: null,    // optional; null = no group
    volume: 1f,
    pitch: 1f
);

// 2D one-shot (spatialBlend forced to 0)
AudioSource src2d = AudioAtlasRuntime.PlayOneShotUI(
    clip,
    volume: 0.8f,
    pitch: 1f
);
```

Pool capacity is 32 sources. If all slots are occupied, the oldest non-looping source is evicted.

### Volume Fades

```csharp
// Smooth volume transition — updates in Update() each frame, zero alloc
AudioAtlasRuntime.FadeVolume(source, targetVolume: 0.5f, duration: 1.0f);

// Fade to zero, then call source.Stop() automatically
AudioAtlasRuntime.FadeOutAndStop(source, duration: 0.8f);
```

Fade registry holds 32 simultaneous fades. If the registry is full, the fade request is silently dropped.

### Procedural Clip Generation

```csharp
// Sine wave click — cached by frequency hash (no re-allocation on repeat calls)
AudioClip click = AudioAtlasRuntime.GetProceduralClick(frequency: 440f);

// Loopable ambient drone
AudioClip drone = AudioAtlasRuntime.GenerateAmbientTone(freq: 80f, durationSec: 2f);

// Percussive SFX burst — useful for voice stress testing
AudioClip burst = AudioAtlasRuntime.GenerateBurstSFX(freq: 220f, decayRate: 6f);
```

Procedural clips are cached in a static `Dictionary<int, AudioClip>` keyed by frequency hash. They are not cleared between sessions — only on domain reload.

---

## SourceSnapshot Struct

Each entry in `AudioRegistry.SourceSnapshots` is a `SourceSnapshot`:

```csharp
public struct SourceSnapshot
{
    public int    InstanceID;         // AudioSource.GetInstanceID()
    public string GameObjectName;     // cached — not re-fetched per frame
    public string ClipName;
    public float  Volume;
    public float  Pitch;
    public float  SpatialBlend;       // 0 = 2D, 1 = full 3D
    public float  Distance;           // world units from AudioListener
    public int    Priority;           // 0 = highest, 256 = lowest
    public string MixerGroupName;     // empty if unassigned
    public bool   IsPlaying;
    public bool   IsLooping;
    public bool   IsPaused;
    public float  Time;               // current playback position in seconds
}
```

---

## AudioHealthScore Struct

```csharp
public struct AudioHealthScore
{
    public float Score;        // 0–100 composite
    public char  Grade;        // 'A' 'B' 'C' 'D' 'F'
    public string StatusLabel; // "Excellent" "Good" "Fair" "Poor" "Critical"

    // Sub-scores (see Intelligence Engine for weight breakdown)
    public float VoiceScore;   // 0–25
    public float CpuScore;     // 0–25
    public float DesignScore;  // 0–25
    public float StealScore;   // 0–15
    public float MixerScore;   // 0–10

    public float Delta;        // score change vs ~5s ago
    public char  TrendArrow;   // '↑' '↓' '→'
    public float ComputedAt;   // Time.realtimeSinceStartup at computation
    public bool  IsValid;      // false before first health tick
}
```

---

## Compile Guards

All tracked playback and registry reads are safe to leave in release builds. The compiler eliminates them when `AUDIOATLAS_DEBUG` is not defined:

```csharp
// Explicit guard — maximum clarity
#if AUDIOATLAS_DEBUG
    AudioAtlasRuntime.PlayOneShotAuto(clip, transform.position);
#else
    AudioSource.PlayClipAtPoint(clip, transform.position);
#endif

// Or: rely on the runtime engine's own internal guard
// AudioAtlasRuntime is only available when UNITY_EDITOR || AUDIOATLAS_DEBUG
// is defined in its source file — calling it outside that context is a compile error
```

---

## ScanCoordinator (Editor)

```csharp
using AudioAtlas.Editor.Scan;
```

```csharp
// Start an async scan — no-op if already running
ScanCoordinator.RunAsync();

// Cancel a running scan
ScanCoordinator.Cancel();

// Read results (null before first scan)
var result = ScanCoordinator.LastResult;

// Subscribe to completion
ScanCoordinator.OnScanComplete += OnScanDone;

// CI-only: blocking scan that pumps the editor loop
ScanResult result = ScanCoordinator.RunBlocking();
```

---

[← Configuration](../configuration/settings-reference.md) · [CI Integration →](../ci/ci-integration.md)
