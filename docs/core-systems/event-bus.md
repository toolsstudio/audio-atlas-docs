# EventBus Reference

Audio Atlas uses a typed, zero-heap-allocation publish/subscribe system. All 32 events are defined in `AudioAtlasEvent`. Handlers receive an `int` payload whose meaning is event-specific.

---

## API

```csharp
using AudioAtlas.Runtime.Core;

// Subscribe — add handler to the pre-allocated slot array
EventBus.Subscribe(AudioAtlasEvent.SourceListUpdated, OnSourcesChanged);

// Raise — calls all handlers for the event; payload is event-specific
EventBus.Raise(AudioAtlasEvent.SourceListUpdated);

// Unsubscribe — swap-remove from the slot array; O(n), n ≤ 32
EventBus.Unsubscribe(AudioAtlasEvent.SourceListUpdated, OnSourcesChanged);

// Clear — nulls all handler slots across all events; called on session end
EventBus.Clear();
```

**Handler signature:** `Action<int>` — the `int` parameter is the payload.

**Slot limit:** 32 handlers per event. Exceeding this logs a warning and the subscription is dropped.

**Thread safety:** `Subscribe`, `Unsubscribe`, and `Raise` are main-thread-only. They are not synchronized with the audio DSP thread. Ring buffer writes use `Interlocked.Increment` separately in `AudioRegistry`.

---

## Always Unsubscribe in OnDestroy

```csharp
void Awake()
{
    EventBus.Subscribe(AudioAtlasEvent.BugReportGenerated, OnScanComplete);
}

void OnDestroy()
{
    EventBus.Unsubscribe(AudioAtlasEvent.BugReportGenerated, OnScanComplete);
}
```

Failing to unsubscribe leaves a dangling delegate in the slot array. It will be invoked until the slot is recycled by another subscription.

---

## Event Catalogue

### Monitoring Events

| Event | Value | Payload | Fired By | Fired When |
|---|---|---|---|---|
| `SourceListUpdated` | 0 | `ActiveSourceCount` | `SourceTracker` | Source list refreshed (every `UpdateIntervalFrames`) |
| `AudioEventRecorded` | 1 | ring buffer slot index | `AudioEventDispatcher` | A play/stop/steal event is written to the ring buffer |
| `MixerDataUpdated` | 2 | 0 | `MixerAnalyzer` | Mixer snapshot refreshed |
| `SourceSelected` | 3 | source instance ID | `SourceMonitorPanel` | User clicks a row in Live Sources |
| `SessionCleared` | 4 | 0 | `AudioAtlasBootstrap` | Session explicitly cleared or Play Mode exits |
| `VoiceStealDetected` | 5 | steal ring buffer slot | `VoiceStealTracker` | A voice steal is detected and written |
| `HeatmapUpdated` | 6 | 0 | `HeatmapSampler` | Heatmap grid refreshed |
| `SignalPathUpdated` | 7 | 0 | `SignalPathAnalyzer` | Signal path for selected source recomputed |
| `LatencyMeasured` | 8 | 0 | `LatencyProbe` | New latency measurement available |

### Session Events

| Event | Value | Payload | Fired By | Fired When |
|---|---|---|---|---|
| `SessionFinalized` | 9 | 0 | `SessionSerializer` | Session saved to disk |
| `ReplayStarted` | 10 | 0 | `ReplayEngine` | Session replay begins |
| `ReplayStopped` | 11 | 0 | `ReplayEngine` | Session replay ends |
| `ReplayPositionChanged` | 12 | frame index | `ReplayEngine` | Replay scrubs to a new frame |

### UI Coordination Events

| Event | Value | Payload | Fired By | Fired When |
|---|---|---|---|---|
| `MixerGroupFocused` | 13 | group instance ID | `MixerGraphView` | User clicks a node in the Mixer Graph |
| `EventSelected` | 14 | event ring buffer slot | `EventLogPanel` | User selects an event in the Event Log |
| `SettingsChanged` | 19 | 0 | `AudioAtlasSettingsProvider` | Settings ScriptableObject modified |

### Analysis Events

| Event | Value | Payload | Fired By | Fired When |
|---|---|---|---|---|
| `BugReportGenerated` | 15 | total finding count | `ScanCoordinator` | Scan completes — `ScanCoordinator.LastResult` is set |
| `SimulationCompleted` | 16 | 0 | `VoiceDropSimulatorPanel` | Voice drop simulation finishes |
| `DiffCompleted` | 17 | 0 | `SessionDiffPanel` | Session diff computation finishes |
| `DependencyGraphUpdated` | 18 | 0 | `ProjectKnowledgeGraph` | Clip dependency graph rebuilt |
| `ClipAnalysisCompleted` | 26 | clip count | `AudioClipSearchPanel` | Clip inventory scan finishes |
| `ReactionFired` | 27 | 0 | `BlastRadiusPanel` | Audio reaction chain triggered |
| `DashboardRefreshed` | 28 | 0 | `SourceMonitorPanel` | Live Sources auto-update |

### Budget & Performance Events

| Event | Value | Payload | Fired By | Fired When |
|---|---|---|---|---|
| `BudgetViolation` | 20 | violation type flag | `BudgetGuardPanel` | Voice or memory budget threshold crossed |
| `PriorityConflictFound` | 22 | conflict count | `BugRuleSet` | AA-006 priority conflict detected |
| `MemoryProfileUpdated` | 23 | 0 | `MemoryProfilerPanel` | Memory analysis refreshed |
| `WaveformScopeUpdated` | 24 | 0 | `WaveScopePanel` | Waveform buffer available |
| `VoiceLifetimeUpdated` | 25 | 0 | `SourceTracker` | Source lifetime statistics updated |

### Intelligence Events

| Event | Value | Payload | Fired By | Fired When |
|---|---|---|---|---|
| `HealthScoreUpdated` | 29 | score × 100 (int) | `AudioIntelligenceEngine` | Health score tick (~0.5 s) |
| `IntelligenceInsightAvailable` | 30 | insight count | `AudioIntelligenceEngine` | New insight cards written to `AudioRegistry.IntelligenceInsights` |
| `CriticalInsightDetected` | 31 | 0 | `IntelligenceAdvisor` | A Critical-severity insight card is newly available |
| `SessionNoteAdded` | 21 | 0 | `SessionNotesPanel` | User saves a session note |

---

## Usage Patterns

### Reading scan results after `BugReportGenerated`

```csharp
void Awake()
{
    EventBus.Subscribe(AudioAtlasEvent.BugReportGenerated, OnScanComplete);
}

void OnDestroy()
{
    EventBus.Unsubscribe(AudioAtlasEvent.BugReportGenerated, OnScanComplete);
}

private void OnScanComplete(int findingCount)
{
    var result = ScanCoordinator.LastResult;
    if (result == null) return;

    Debug.Log($"Scan complete — {findingCount} findings, health: {result.HealthReport?.HealthGrade}");
}
```

### Reacting to budget violations

```csharp
EventBus.Subscribe(AudioAtlasEvent.BudgetViolation, _ =>
{
    // Throttle non-critical audio in response
    AudioAtlasRuntime.FadeVolume(ambientSource, 0.3f, 1.5f);
});
```

### Watching health score in a custom HUD

```csharp
EventBus.Subscribe(AudioAtlasEvent.HealthScoreUpdated, _ =>
{
    var score = AudioRegistry.HealthScore;
    if (score.IsValid)
        UpdateDebugOverlay(score.Grade, score.Score);
});
```

---

[← Architecture](architecture.md) · [Intelligence Engine →](intelligence-engine.md)
