# Changelog

All notable changes are documented here.  
Format: [Keep a Changelog](https://keepachangelog.com)  
Versioning: [Semantic Versioning](https://semver.org)

---

## [1.0.0] — 2025

Initial commercial release.

### Runtime Assembly (`AudioAtlas.Runtime`)

**Core**
- `AudioRegistry` — static data store with pre-allocated `SourceSnapshot[]`, `MixerGroupSnapshot[]`, `AudioEvent[]`, `VoiceSteal[]` ring buffers. All properties non-null before initialization. Thread-safe ring write via `Interlocked.Increment`.
- `EventBus` — typed publish/subscribe with 32 events and 32 handler slots per event. Zero heap allocation on `Raise`. Handler isolation via per-slot try/catch.
- `MetricsCollector` — per-frame aggregation of voice count, CPU usage, and source state into `AudioRegistry`.

**Bootstrap**
- `AudioAtlasBootstrap` — `[RuntimeInitializeOnLoadMethod(BeforeSceneLoad)]` activation. Creates `[AudioAtlas_Root]` and `[AudioAtlas_Engine]` GameObjects with `DontDestroyOnLoad` and `HideFlags.HideAndDontSave`. `SubsystemRegistration` reset for Disable Domain Reload compatibility.

**Engine**
- `AudioAtlasRuntime` — 32-slot `AudioSource` pool + 32-slot fade registry. `PlayOneShotAuto()`, `PlayOneShotUI()`, `FadeVolume()`, `FadeOutAndStop()`. Procedural clip generation with frequency-hash cache.

**Monitoring**
- `SourceTracker` — per-frame source scan, `SourceSnapshot` population, `SourceListUpdated` event
- `VoiceStealTracker` — frame-delta steal detection via priority/playstate comparison
- `HeatmapSampler` — configurable-interval 3D spatial sampling → `HeatmapData`
- `ListenerTracker` — `AudioListener` world position capture
- `LatencyProbe` — DSP burst timing → `LatencyStats`

**Intelligence**
- `AudioIntelligenceEngine` — 1-second analysis tick. Clip fire-rate detection (open-addressing hash, cap 128). Co-occurrence detection (24-slot ring, 50ms window). Steal burst detection (16-slot ring, 500ms window, 3-steal threshold). Five-component health score (Voice 25 · CPU 25 · Design 25 · Steal 15 · Mixer 10).
- `IntelligenceAdvisor` — `CriticalInsightDetected` promotion from insight pool.

**Analysis**
- `MixerAnalyzer` — `AudioMixer` group snapshot via reflection
- `SignalPathAnalyzer` — mixer group parent chain walking → `SignalPath`

**Models**
- `AudioAtlasConfig`, `AudioAtlasConfigAsset`, `AudioEvent`, `AudioHealthReport`, `AudioHealthScore`, `AudioSession`, `BugRecord`, `DependencyGraph`, `HeatmapData`, `InsightCard`, `IntelligenceContext`, `LatencyStats`, `MetricsFrame`, `MixerGroupSnapshot`, `PlatformBudgetProfile`, `SessionDiff`, `SessionMetadata`, `SignalPath`, `SimulationScenario`, `SourceSnapshot`, `VoiceSteal`

---

### Editor Assembly (`AudioAtlas.Editor`)

**Window**
- `AudioAtlasWindow` — 20-panel docked `EditorWindow`. Resizable sidebar (icon-only collapse at <900px). Draggable resize handle with `EditorPrefs` persistence. Toolbar: Scan, Report, Pause, Clear, Settings, Docs, Overflow. Status bar: PLAY/EDIT state, voice count, event count, steal count, health grade.

**Panels — Monitor**
- `SourceMonitorPanel`, `MixerGraphView`, `EventLogPanel`

**Panels — Analyze**
- `BugReportPanel`, `SignalTracePanel`, `SourceInspectorPanel`, `SceneOverlay`, `AudioGizmoRenderer`

**Panels — Insight**
- `MetricsPanel`, `IntelligenceFeedPanel`

**Panels — Assets**
- `AudioClipSearchPanel`, `MemoryProfilerPanel`, `AssetsPanel`, `BudgetGuardPanel`, `BlastRadiusPanel`

**Panels — Tools**
- `SessionNotesPanel`, `BookmarksPanel`, `WaveScopePanel`, `SessionDiffPanel`, `VoiceDropSimulatorPanel`

**Panels — Build**
- `HistoryPanel`

**Scan & Validation**
- `ScanCoordinator`, `ArchitectureValidator`, `BugRuleSet`, `AudioFixer`

**Automation**
- `CIBridge` — `-executeMethod` entry point, JSON report writer, exit codes 0/1/2
- `BatchProcessor`, `AutoTagSystem`, `AudioAtlasConfigCreator`

**Analysis**
- `SessionDiffEngine` — six-dimension diff, pre-allocated dictionary reuse, zero reallocation per `Compute()` call
- `VoiceDropSimulator` — deterministic probability, 32 distance samples, pre-allocated 256-slot buffers

**Settings**
- `AudioAtlasSettings`, `AudioAtlasSettingsProvider`

**Tests**
- `AudioRegistryTests`, `BugRuleSetTests`, `EventBusTests`, `MetricsCollectorTests`

---

## Versioning Policy

| Change type | Version bump |
|---|---|
| Breaking API change (removed/renamed public member) | Major (1.x → 2.x) |
| New panel, new event, new auto-fix rule | Minor (1.0 → 1.1) |
| Bug fix, performance improvement, log cleanup | Patch (1.0.0 → 1.0.1) |

---

[Back to Docs](README.md) · [Contact Support](mailto:tools.studio@zohomail.in)
