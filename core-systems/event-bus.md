# EventBus Reference

Audio Atlas uses a typed, zero-heap-allocation publish/subscribe system. All 32 events are defined in `AudioAtlasEvent`. Handlers receive an `int` payload whose meaning is event-specific.

---

## API

```csharp
using AudioAtlas.Runtime.Core;

// Subscribe
EventBus.Subscribe(AudioAtlasEvent.SourceListUpdated, OnSourcesChanged);

// Raise — calls all handlers; payload is event-specific
EventBus.Raise(AudioAtlasEvent.SourceListUpdated);

// Unsubscribe — swap-remove from slot array; O(n), n ≤ 32
EventBus.Unsubscribe(AudioAtlasEvent.SourceListUpdated, OnSourcesChanged);

// Clear — nulls all handler slots across all events; called on session end
EventBus.Clear();
```

**Handler signature:** `Action<int>`  
**Slot limit:** 32 handlers per event. Exceeding this logs a warning and the subscription is dropped.  
**Thread safety:** `Subscribe`, `Unsubscribe`, and `Raise` are main-thread-only.

---

## Always Unsubscribe in OnDestroy

```csharp
void Awake()   => EventBus.Subscribe(AudioAtlasEvent.BugReportGenerated, OnScanComplete);
void OnDestroy() => EventBus.Unsubscribe(AudioAtlasEvent.BugReportGenerated, OnScanComplete);
```

---

## Event Catalogue

### Monitoring Events

| Event | Payload | Fired By | Fired When |
|---|---|---|---|
| `SourceListUpdated` (0) | `ActiveSourceCount` | `SourceTracker` | Source list refreshed |
| `AudioEventRecorded` (1) | ring buffer slot | `AudioEventDispatcher` | Play/stop/steal event written |
| `MixerDataUpdated` (2) | 0 | `MixerAnalyzer` | Mixer snapshot refreshed |
| `SourceSelected` (3) | source instance ID | `SourceMonitorPanel` | User clicks Live Sources row |
| `SessionCleared` (4) | 0 | `AudioAtlasBootstrap` | Session cleared or Play Mode exits |
| `VoiceStealDetected` (5) | steal ring slot | `VoiceStealTracker` | Voice steal detected and written |
| `HeatmapUpdated` (6) | 0 | `HeatmapSampler` | Heatmap grid refreshed |
| `SignalPathUpdated` (7) | 0 | `SignalPathAnalyzer` | Signal path recomputed |
| `LatencyMeasured` (8) | 0 | `LatencyProbe` | New latency measurement available |

### Session Events

| Event | Payload | Fired By | Fired When |
|---|---|---|---|
| `SessionFinalized` (9) | 0 | `SessionSerializer` | Session saved to disk |
| `ReplayStarted` (10) | 0 | `ReplayEngine` | Session replay begins |
| `ReplayStopped` (11) | 0 | `ReplayEngine` | Session replay ends |
| `ReplayPositionChanged` (12) | frame index | `ReplayEngine` | Replay scrubs to new frame |

### UI Coordination Events

| Event | Payload | Fired By | Fired When |
|---|---|---|---|
| `MixerGroupFocused` (13) | group instance ID | `MixerGraphView` | User clicks a graph node |
| `EventSelected` (14) | event ring slot | `EventLogPanel` | User selects Event Log row |
| `SettingsChanged` (19) | 0 | `AudioAtlasSettingsProvider` | Settings asset modified |

### Analysis Events

| Event | Payload | Fired By | Fired When |
|---|---|---|---|
| `BugReportGenerated` (15) | total finding count | `ScanCoordinator` | Scan completes |
| `SimulationCompleted` (16) | 0 | `VoiceDropSimulatorPanel` | Voice drop simulation finishes |
| `DiffCompleted` (17) | 0 | `SessionDiffPanel` | Session diff computation finishes |
| `DependencyGraphUpdated` (18) | 0 | `ProjectKnowledgeGraph` | Clip dependency graph rebuilt |
| `ClipAnalysisCompleted` (26) | clip count | `AudioClipSearchPanel` | Clip inventory scan finishes |
| `DashboardRefreshed` (28) | 0 | `SourceMonitorPanel` | Live Sources auto-update |

### Budget & Performance Events

| Event | Payload | Fired By | Fired When |
|---|---|---|---|
| `BudgetViolation` (20) | violation type flag | `BudgetGuardPanel` | Budget threshold crossed |
| `PriorityConflictFound` (22) | conflict count | `BugRuleSet` | AA-006 conflict detected |
| `MemoryProfileUpdated` (23) | 0 | `MemoryProfilerPanel` | Memory analysis refreshed |
| `WaveformScopeUpdated` (24) | 0 | `WaveScopePanel` | Waveform buffer available |
| `VoiceLifetimeUpdated` (25) | 0 | `SourceTracker` | Source lifetime statistics updated |

### Intelligence Events

| Event | Payload | Fired By | Fired When |
|---|---|---|---|
| `HealthScoreUpdated` (29) | score × 100 | `AudioIntelligenceEngine` | Health score tick (~0.5 s) |
| `IntelligenceInsightAvailable` (30) | insight count | `AudioIntelligenceEngine` | New insight cards written |
| `CriticalInsightDetected` (31) | 0 | `IntelligenceAdvisor` | Critical-severity insight newly available |
| `SessionNoteAdded` (21) | 0 | `SessionNotesPanel` | User saves a note |

---

## Usage Patterns

### Reading scan results

```csharp
void Awake()     => EventBus.Subscribe(AudioAtlasEvent.BugReportGenerated, OnScanComplete);
void OnDestroy() => EventBus.Unsubscribe(AudioAtlasEvent.BugReportGenerated, OnScanComplete);

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
