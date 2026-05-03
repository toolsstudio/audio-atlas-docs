# Changelog

All notable changes are documented here.  
Format: Keep a Changelog (keepachangelog.com)  
Versioning: Semantic Versioning (semver.org)

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
- `AudioAtlasRuntime` — 32-slot `AudioSource` pool + 32-slot fade registry. `PlayOneShotAuto()`, `PlayOneShotUI()`, `FadeVolume()`, `FadeOutAndStop()`. Procedural clip generation: `GetProceduralClick()`, `GenerateAmbientTone()`, `GenerateBurstSFX()`. Procedural clip cache keyed by frequency hash.

**Monitoring**
- `SourceTracker` — per-frame source scan, `SourceSnapshot` population, `SourceListUpdated` event
- `VoiceStealTracker` — frame-delta steal detection via priority/playstate comparison
- `HeatmapSampler` — configurable-interval 3D spatial sampling → `HeatmapData`
- `ListenerTracker` — `AudioListener` world position capture → `AudioRegistry.ListenerPosition`
- `LatencyProbe` — DSP burst timing → `LatencyStats`

**Intelligence**
- `AudioIntelligenceEngine` — 1-second analysis tick. Clip fire-rate (open-addressing hash table, cap 128). Co-occurrence detection (24-slot ring, 50ms window). Steal burst detection (16-slot ring, 500ms window, 3-steal threshold). Source design analysis. Five-component health score (Voice 25 · CPU 25 · Design 25 · Steal 15 · Mixer 10).
- `IntelligenceAdvisor` — `CriticalInsightDetected` promotion from insight pool.

**Analysis**
- `MixerAnalyzer` — `AudioMixer` group snapshot via reflection
- `SignalPathAnalyzer` — mixer group parent chain walking → `SignalPath`

**Events**
- `AudioEventDispatcher` — play/stop event writing to `EventRingBuffer`
- `EventInterceptor` — DSP-thread event capture
- `ReplayEngine` — session frame replay

**Models (all structs/classes)**
- `AudioAtlasConfig`, `AudioAtlasConfigAsset`, `AudioEvent`, `AudioHealthReport`, `AudioHealthScore`, `AudioSession`, `BugRecord`, `DependencyGraph`, `HeatmapData`, `InsightCard`, `IntelligenceContext`, `LatencyStats`, `MetricsFrame`, `MixerGroupSnapshot`, `PlatformBudgetProfile`, `SessionDiff`, `SessionMetadata`, `SignalPath`, `SimulationScenario`, `SourceSnapshot`, `VoiceSteal`

---

### Editor Assembly (`AudioAtlas.Editor`)

**Window**
- `AudioAtlasWindow` — 20-panel docked `EditorWindow`. Resizable sidebar (icon-only collapse at <900px). Draggable resize handle with `EditorPrefs` persistence. Toolbar: Scan, Report, Pause, Clear, Settings, Docs, Overflow. Status bar: PLAY/EDIT state, voice count, event count, steal count, health grade.
- `AudioAtlasWindowLayout` — layout constants (ToolbarZones, SidebarLayout, WindowMetrics). Dynamic `bannerX` computation prevents LIVE indicator overlap.

**Panels — Monitor**
- `SourceMonitorPanel` — Live Sources table with Playing/All/Looping filter, search, column sort, click-to-select
- `MixerGraphView` — node graph with `MixerGroupNode`, `MixerEdge`, click-to-focus
- `EventLogPanel` — ring buffer table, CSV export, click-to-select

**Panels — Analyze**
- `BugReportPanel` — scan results table, severity grouping, Fix buttons, Simulate mode
- `SignalTracePanel` — signal chain render for selected source
- `SourceInspectorPanel` — serialized vs. runtime comparison for selected source
- `SceneOverlay` + `AudioGizmoRenderer` — Scene View heatmap + spatial gizmos

**Panels — Insight**
- `MetricsPanel` — health score display with sub-score bars and KPI cards
- `IntelligenceFeedPanel` — ranked insight cards from `IntelligenceInsights`

**Panels — Assets**
- `AudioClipSearchPanel` — full-project clip inventory with filter/sort/export/delete
- `MemoryProfilerPanel` — per-clip memory analysis by load type
- `AssetsPanel` — asset references cross-map (clip → AudioSource consumers)
- `BudgetGuardPanel` — per-platform budget enforcement with `BudgetViolation` events
- `BlastRadiusPanel` — audio asset dependency and blast radius analysis

**Panels — Tools**
- `SessionNotesPanel` — freeform notes editor persisted in `AudioSession`
- `BookmarksPanel` — `EditorPrefs`-persisted source bookmarks
- `WaveScopePanel` — real-time DSP waveform (zero alloc, `Volatile`/`MemoryBarrier` thread safety)
- `SessionDiffPanel` — six-dimension session comparison via `SessionDiffEngine`
- `VoiceDropSimulatorPanel` — Monte Carlo-free deterministic voice drop probability

**Panels — Build**
- `HistoryPanel` — build record log with sparkline health trend

**Scan**
- `ScanCoordinator` — single ownership scan entry point. `RunAsync()`, `RunBlocking()` (CI), `Cancel()`, `LastResult`, `OnScanComplete` event.
- `ArchitectureValidator` — async project scan with timeout

**Validation**
- `BugRuleSet` — 12 rules (AA-001 through AA-012), self-contained static methods, `ScanContext` struct
- `AudioFixer` — 5 auto-fixes (AA-001, 003, 007, 008, 009) with named undo groups
- `BugReportPanel` — interactive results display

**Automation**
- `CIBridge` — `-executeMethod` entry point, JSON report writer, exit codes 0/1/2
- `BatchProcessor` — multi-project scan infrastructure
- `AutoTagSystem` — automatic audio asset tagging
- `AudioAtlasConfigCreator` — `Create Config Asset` menu item handler

**Analysis**
- `SessionDiffEngine` — six-dimension diff: new events, missing events, volume changes (3dB threshold), trigger count changes (20% threshold), steal changes, latency changes (5ms threshold). Pre-allocated dictionary reuse, zero reallocation per `Compute()` call.
- `VoiceDropSimulator` — deterministic voice drop probability. 32 distance samples per source. Pre-allocated working buffers (256 slots). `SimulationResult` with per-source drop probability.

**Settings**
- `AudioAtlasSettings` — `ScriptableObject` editor config with embedded `AudioAtlasConfig`
- `AudioAtlasSettingsProvider` — `Project Settings → AudioAtlas` page

**Utilities**
- `PanelStyles` — IMGUI style cache, colour palette, `GetWhiteIcon()` blit utility
- `PanelIcons` — `GUIContent` icon cache for sidebar navigation
- `WaveformRenderer` — DSP waveform render utility
- `TimeFormatter` — consistent timestamp formatting
- `RenderPipelineShaderResolver` — pipeline-safe material creation for gizmo overlay

**Branding**
- `AudioAtlasBranding` — `[InitializeOnLoad]` package-safe texture loader via `AssetDatabase` self-discovery (not `EditorGUIUtility.Load`)
- `AudioAtlasBrandingImporter` — `AssetPostprocessor` that enforces `TextureImporterType.GUI` on first import of branding PNGs

**Build**
- `BuildHistory` — `EditorPrefs`-persisted build record list
- `BuildRecord` — scan result snapshot per build

**Serialization**
- `SessionSerializer` — JSON session save/load to `AutoSavePath`

**Knowledge**
- `ProjectKnowledgeGraph` — clip dependency graph builder

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

[tools.studio@zohomail.in](mailto:tools.studio@zohomail.in)
