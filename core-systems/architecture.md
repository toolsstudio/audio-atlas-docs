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
│  VoiceDropSimulator · MixerGraphView · SceneOverlay        │
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

**The Runtime assembly has zero dependency on the Editor assembly.** This boundary is enforced by `.asmdef` configuration and ensures the Runtime can ship without any editor code leaking into the player.

---

## Bootstrap Lifecycle

```
SubsystemRegistration  → ResetState()
                          clears static _root field
                          (safe with Disable Domain Reload)

BeforeSceneLoad        → Initialize()
                          if UNITY_EDITOR || AUDIOATLAS_DEBUG:
                            LoadOrCreateConfig()
                            AudioRegistry.Initialize(config)
                            Instantiate [AudioAtlas_Root] GameObject
                            DontDestroyOnLoad
                            root.Boot(config)
                            AudioAtlasRuntime.BootRuntime()
                              → [AudioAtlas_Engine] GameObject
```

The `[AudioAtlas_Root]` object hosts `MetricsCollector`, `MixerAnalyzer`, `SignalPathAnalyzer`, `AudioIntelligenceEngine`, `IntelligenceAdvisor`, `ListenerTracker`, `SourceTracker`, `VoiceStealTracker`, `HeatmapSampler`, and `LatencyProbe`.

The `[AudioAtlas_Engine]` object hosts `AudioAtlasRuntime` — the 32-slot `AudioSource` pool and 32-slot fade registry.

Both objects use `HideFlags.HideAndDontSave` — they do not appear in the Hierarchy and are not serialized.

---

## AudioRegistry

`AudioRegistry` is a static class. All mutable state for the runtime monitoring system lives here. Editor panels read from it on every repaint.

```csharp
// Core arrays — pre-allocated to configured capacity
SourceSnapshot[]     SourceSnapshots   // [MaxTrackedSources]
MixerGroupSnapshot[] MixerSnapshots    // [MaxMixerGroups]
AudioEvent[]         EventRingBuffer   // [EventRingBufferSize]  — ring
VoiceSteal[]         StealRingBuffer   // [StealRingBufferSize]  — ring

// Atomic write indices (Interlocked.Increment)
int EventWriteIndex
int StealWriteIndex

// Scalar state
int   ActiveSourceCount
int   SelectedSourceID     // -1 = no selection
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

All array properties return non-null empty arrays before `Initialize()` is called — safe to read from editor panels in Edit Mode without null checks.

Ring buffer writes use `Interlocked.Increment` for thread safety between the DSP thread and the main thread.

---

## EventBus

Typed, zero-heap-allocation publish/subscribe. 32 events. 32 handler slots per event. Pre-allocated at static construction.

```csharp
// Internal representation
Action<int>[][] _handlers      // [eventCount][MaxHandlersPerEvent]
int[]           _handlerCounts // [eventCount]
```

`Raise(evt, payload)` iterates the pre-allocated slot array — no `delegate.GetInvocationList()`, no LINQ, no array allocation. Each handler is wrapped in a per-slot try/catch to isolate failures.

`Clear()` is called on session end — it nulls all slots and resets counts without reallocating the arrays.

See [EventBus Reference](event-bus.md) for the complete event catalogue.

---

## Intelligence Engine

`AudioIntelligenceEngine` runs on the main thread inside `AudioAtlasBootstrap.Update()`. It uses a 1-second analysis throttle to separate hot-path callbacks (zero-alloc) from analysis passes (string work allowed).

### Health Score Composition

| Sub-score | Weight | Measures |
|---|---|---|
| `VoiceScore` | 25 | Voice efficiency — ratio of active voices to budget |
| `CpuScore` | 25 | Audio CPU vs `AudioCpuCriticalPercent` threshold |
| `DesignScore` | 25 | Priority assignment quality, routing coverage, spatial accuracy |
| `StealScore` | 15 | Steal rate + burst penalty |
| `MixerScore` | 10 | Mixer group routing coverage |

Grade thresholds: **A** ≥90 · **B** ≥75 · **C** ≥60 · **D** ≥40 · **F** <40

See [Intelligence Engine](intelligence-engine.md) for the full formula breakdown.

---

## ScanCoordinator

Single ownership model for all project scans. Every scan entry point calls `ScanCoordinator.RunAsync()` or `ScanCoordinator.RunBlocking()`. There are no private result copies elsewhere.

```csharp
ScanCoordinator.LastResult       // null before first scan
ScanCoordinator.IsRunning        // delegates to ArchitectureValidator.IsRunning
ScanCoordinator.OnScanComplete   // event — receives ScanResult when scan ends
ScanCoordinator.RunAsync()       // starts async scan; no-op if already running
ScanCoordinator.RunBlocking()    // synchronous; CI only — pumps editor loop
ScanCoordinator.Cancel()         // safe to call when nothing is running
```

`ArchitectureValidator` iterates every `AudioSource` in every open scene plus every `AudioClip` via `AssetDatabase`. Each source is wrapped in a `ScanContext` struct and passed to every enabled rule in `BugRuleSet.BuildRuleTable()`.

---

## VoiceDropSimulator

Deterministic voice drop prediction — no randomisation. For each `SimulationScenario`, it samples each source's expected audibility across 32 discrete distance points. Sources ranked beyond `VoiceLimit` are counted as dropped. Working buffers are pre-allocated at 256 slots — zero heap allocation per simulation run.

---

## SessionDiffEngine

Computes structured diffs between two `AudioSession` objects across six behavioral dimensions. Internal `Dictionary<string, int>` working sets are cleared and reused per call — no reallocation.

Thresholds: volume delta **3 dB** · trigger count change **20%** · latency delta **5 ms**

---

[Back to Docs](../README.md) · [EventBus Reference →](event-bus.md)
