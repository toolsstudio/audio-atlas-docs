# Architecture

Audio Atlas is structured around three independently compilable assemblies, a central static registry, a lock-free event bus, and an intelligence engine that runs on the main thread with a throttled analysis cadence.

---

## Assembly Dependency Graph

```
┌────────────────────────────────────────────────────────────┐
│  AudioAtlas.Editor  (editor-only, stripped from builds)    │
│                                                            │
│  AudioAtlasWindow · 20 panels · ScanCoordinator            │
│  BugRuleSet · AudioFixer · CIBridge · SessionDiffEngine    │
│  VoiceDropSimulator · AudioIntelligenceEngine panel        │
│  MixerGraphView · SceneOverlay · SessionSerializer         │
└────────────────────┬───────────────────────────────────────┘
                     │  depends on
┌────────────────────▼───────────────────────────────────────┐
│  AudioAtlas.Runtime  (conditional — see Scripting Defines) │
│                                                            │
│  AudioRegistry · EventBus · MetricsCollector               │
│  AudioAtlasBootstrap · AudioAtlasRuntime engine            │
│  SourceTracker · VoiceStealTracker · HeatmapSampler        │
│  ListenerTracker · LatencyProbe · MixerAnalyzer            │
│  SignalPathAnalyzer · AudioIntelligenceEngine              │
│  IntelligenceAdvisor · AudioEventDispatcher                │
└────────────────────────────────────────────────────────────┘

AudioAtlas.Examples → AudioAtlas.Runtime (one-way, editor only)
```

**The Runtime assembly has zero dependency on the Editor assembly.** This boundary is enforced by the `.asmdef` configuration and ensures the Runtime can ship without any editor code leaking into your player.

---

## Bootstrap Lifecycle

Audio Atlas uses Unity's `RuntimeInitializeOnLoadMethod` infrastructure so no scene setup is required.

```
SubsystemRegistration  → ResetState()        clears static _root field
                                              (safe with Disable Domain Reload)

BeforeSceneLoad        → Initialize()        if UNITY_EDITOR || AUDIOATLAS_DEBUG:
                                               LoadOrCreateConfig()
                                               AudioRegistry.Initialize(config)
                                               Instantiate [AudioAtlas_Root] GameObject
                                               DontDestroyOnLoad
                                               root.Boot(config)
                                               AudioAtlasRuntime.BootRuntime()
                                                 → [AudioAtlas_Engine] GameObject
```

The `[AudioAtlas_Root]` object hosts:
- `MetricsCollector` — aggregates `SourceSnapshot[]` and mixer data
- `MixerAnalyzer` — polls `AudioMixer` via reflection introspection
- `SignalPathAnalyzer` — walks mixer group parent chains
- `AudioIntelligenceEngine` — tick-throttled analysis + health scoring
- `IntelligenceAdvisor` — promotes top-ranked insights to critical status
- `ListenerTracker`, `SourceTracker`, `VoiceStealTracker`, `HeatmapSampler`, `LatencyProbe`

The `[AudioAtlas_Engine]` object hosts:
- `AudioAtlasRuntime` — 32-slot `AudioSource` pool + 32-slot fade registry

Both objects use `HideFlags.HideAndDontSave` — they do not appear in the Hierarchy and are not serialized.

---

## AudioRegistry

`AudioRegistry` is a static class. All mutable state for the runtime monitoring system lives here. Editor panels read from it on every repaint.

```csharp
// Core arrays — pre-allocated to configured capacity
SourceSnapshot[]     SourceSnapshots   // [MaxTrackedSources]
MixerGroupSnapshot[] MixerSnapshots    // [MaxMixerGroups]
AudioEvent[]         EventRingBuffer   // [EventRingBufferSize]  — ring, not queue
VoiceSteal[]         StealRingBuffer   // [StealRingBufferSize]  — ring, not queue

// Atomic write indices (Interlocked.Increment)
int EventWriteIndex
int StealWriteIndex

// Scalar state
int   ActiveSourceCount
int   SelectedSourceID     // -1 = no selection; set by SourceMonitorPanel
bool  IsInitialized

// Spatial + session
Vector3        ListenerPosition
SignalPath     CurrentSignalPath
HeatmapData    CurrentHeatmap
LatencyStats   LatencyData
AudioSession   ActiveSession

// Intelligence output
InsightCard[]      IntelligenceInsights
int                IntelligenceInsightCount
AudioHealthScore   HealthScore
IntelligenceContext ActiveContext
```

All array properties return non-null empty arrays before `Initialize()` is called, making them safe to read from editor panels in Edit Mode without null checks.

Ring buffer writes use `Interlocked.Increment` on the write index for thread safety between the audio thread (DSP callbacks) and the main thread (panel reads).

---

## EventBus

A typed, zero-heap-allocation publish/subscribe system. 32 events. 32 handler slots per event. Pre-allocated at static construction.

```csharp
// Internal representation
Action<int>[][] _handlers      // [eventCount][MaxHandlersPerEvent]
int[]           _handlerCounts // [eventCount]
```

`Raise(evt, payload)` iterates the pre-allocated slot array — no `delegate.GetInvocationList()`, no LINQ, no array allocation. Each handler is wrapped in a try/catch to isolate failures.

`Subscribe`/`Unsubscribe` are O(n) over handler count (≤32), acceptable for an editor tool.

`Clear()` is called on session end — it nulls all slots and resets counts without reallocating the arrays.

See [EventBus Reference](event-bus.md) for the complete event catalogue and usage.

---

## Intelligence Engine

`AudioIntelligenceEngine` runs on the main thread inside `AudioAtlasBootstrap.Update()`. It uses a 1-second analysis throttle to separate hot-path callbacks (zero-alloc) from analysis passes (string work allowed).

### Analysis Algorithms

**Clip fire-rate analysis**
Maintains a hash table (open addressing, linear probe, capacity 128) mapping clip name hash → ring buffer of up to 12 recent fire timestamps. On each 1-second tick, clips that fire >10× within a 1-second window are flagged as spam candidates.

**Co-occurrence detection**
A 24-slot circular buffer records recent `(clipNameHash, timestamp)` pairs. When a new event arrives, the buffer is scanned for entries within a 50ms window. Clip pairs that co-fire 3+ times are marked as implicitly coupled.

**Steal burst detection**
A 16-slot ring of recent steal timestamps. Bursts are detected when 3+ steals occur within a 500ms window. `_burstCountThisSession` accumulates across the session.

**Health score composition**
Five sub-scores summing to 100:

| Sub-score | Weight | Measures |
|---|---|---|
| `VoiceScore` | 25 | Voice efficiency — ratio of active voices to budget |
| `CpuScore` | 25 | Audio CPU vs `AudioCpuCriticalPercent` threshold |
| `DesignScore` | 25 | Priority assignment quality, routing coverage, spatial accuracy |
| `StealScore` | 15 | Steal rate + burst penalty |
| `MixerScore` | 10 | Mixer group routing coverage (sources without groups) |

Grade thresholds: **A** ≥90 · **B** ≥75 · **C** ≥60 · **D** ≥40 · **F** <40

Updated at `EventBus.Raise(HealthScoreUpdated)`. The delta is computed against the score 5 seconds prior; the trend arrow character (`↑ ↓ →`) reflects the direction.

---

## ScanCoordinator

Single ownership model for all project scans. Every scan entry point — toolbar button, BugReportPanel, CIBridge — calls `ScanCoordinator.RunAsync()` or `ScanCoordinator.RunBlocking()`. There are no private result copies anywhere else.

```csharp
ScanCoordinator.LastResult     // null before first scan; set atomically before event fires
ScanCoordinator.IsRunning      // delegates to ArchitectureValidator.IsRunning
ScanCoordinator.OnScanComplete // event — receives ScanResult when scan ends
ScanCoordinator.RunAsync()     // starts async scan; no-op if already running
ScanCoordinator.RunBlocking()  // synchronous; CI only — pumps editor loop, no Thread.Sleep
ScanCoordinator.Cancel()       // safe to call when nothing is running
```

`ArchitectureValidator` iterates every `AudioSource` in every open scene plus every `AudioClip` in the project via `AssetDatabase`. Each source is wrapped in a `ScanContext` struct (pre-populated with the serialized object, clip, sibling buffer) and passed to every enabled rule in `BugRuleSet.BuildRuleTable()`.

---

## VoiceDropSimulator

Deterministic voice drop prediction. **No randomisation.** For each `SimulationScenario`, it samples each source's expected audibility across `DistanceSamples` (32) discrete distance points between its min and max distance.

For each sample point, all sources are ranked by audibility. Sources ranked beyond `VoiceLimit` are counted as dropped. The drop probability for each source is `dropCount / totalSamples`.

Working buffers (`_audibilityBuffer`, `_rankBuffer`, `_dropCountBuffer`, etc.) are pre-allocated at 256 and grown if the scenario is larger. Zero heap allocation per simulation run.

---

## SessionDiffEngine

Computes structured diffs between two `AudioSession` objects across six behavioral dimensions. Internal `Dictionary<string, int>` working sets are cleared and reused per call — no reallocation.

Thresholds:
- Volume delta: **3 dB** (below this, the change is perceptually insignificant)
- Trigger count change: **20%** (below this, variation is noise)
- Latency delta: **5 ms** (below this, not worth surfacing)

---

[← Back to Docs](../../README.md) · [EventBus Reference →](event-bus.md)
