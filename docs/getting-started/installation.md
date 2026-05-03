# Installation

## Requirements

| Requirement | Minimum Value |
|---|---|
| Unity Version | **6000.0** (Unity 6) |
| .NET Compatibility | Standard 2.1 |
| Scripting Backend | Mono or IL2CPP |
| Render Pipeline | Built-in, URP, or HDRP — all supported |
| Editor Platform | Windows, macOS, Linux |

---

## Acquiring the Package

Audio Atlas is a paid commercial asset. A valid purchase is required.

> **Source code repository access is not included with purchase.**
> The asset is delivered exclusively as a Unity package.
> Repository access may be granted separately upon request — contact [tools.studio@zohomail.in](mailto:tools.studio@zohomail.in).

---

## Install Method 1 — Unity Asset Store (Recommended)

1. Purchase Audio Atlas from the Unity Asset Store
2. Open `Window → Package Manager` in Unity
3. Select **My Assets** from the dropdown at the top left
4. Locate **Audio Atlas** in your purchased assets and click **Download**, then **Import**
5. Leave all items checked in the Import dialog and click **Import**

Unity places all content under `Assets/AudioAtlas/`.

---

## Install Method 2 — Manual .unitypackage

For purchases from other authorized marketplaces, a `.unitypackage` file is provided.

1. Open your Unity project
2. Use `Assets → Import Package → Custom Package` and select the downloaded `.unitypackage`, or drag it into the Project window
3. Leave all items checked in the Import dialog and click **Import**

Unity places all content under `Assets/AudioAtlas/`. No repository cloning or source checkout is required or provided.

---

## First Launch

```
Window → Audio → Audio Atlas     Ctrl+Shift+A
```

On first launch, Audio Atlas:
1. Creates `Assets/AudioAtlas/Settings/AudioAtlasSettings.asset` with defaults
2. Opens the main window with **Live Sources** as the active panel

No scene changes. No `GameObject` placement. No code required.

---

## Package Layout

```
Assets/
└── AudioAtlas/
    ├── Documentation/          Local docs (CHANGELOG, README, LICENSE, Docs.md)
    ├── Editor/                 ← editor-only, never in player builds
    │   ├── Analysis/           SessionDiffEngine, VoiceDropSimulator
    │   ├── Automation/         AudioFixer, BatchProcessor, CIBridge, ConfigCreator
    │   ├── Branding/           Package-safe asset loader (AudioAtlasBranding)
    │   ├── Build/              BuildHistory, BuildRecord, HistoryPanel
    │   ├── Config/             AudioAtlasEdition, AudioAtlasLinks, TeamConfig
    │   ├── Features/           AssetsPanel, ClipSearchPanel, MemoryProfilerPanel,
    │   │                       BudgetGuardPanel, SessionNotesPanel, WaveScopePanel,
    │   │                       BookmarksPanel, BlastRadiusPanel
    │   ├── Graph/              MixerGraphView, MixerGroupNode, MixerEdge
    │   ├── Intelligence/       IntelligenceFeedPanel
    │   ├── Knowledge/          ProjectKnowledgeGraph
    │   ├── Panels/             EventLogPanel, MetricsPanel, SessionDiffPanel,
    │   │                       SignalTracePanel, SourceInspectorPanel,
    │   │                       SourceMonitorPanel, VoiceDropSimulatorPanel
    │   ├── Scan/               ScanCoordinator, ArchitectureValidator
    │   ├── SceneOverlay/       SceneOverlay, AudioGizmoRenderer
    │   ├── Serialization/      SessionSerializer
    │   ├── Settings/           AudioAtlasSettings, AudioAtlasSettingsProvider
    │   ├── Utilities/          PanelIcons, PanelStyles, WaveformRenderer, TimeFormatter
    │   ├── Validation/         ArchitectureValidator, BugRuleSet, BugReportPanel
    │   └── Windows/            AudioAtlasWindow, AudioAtlasWindowLayout
    ├── Examples/
    │   └── DemoScene/          Playable demo with annotated audio sources
    ├── Resources/              AudioAtlasConfig.asset (runtime config)
    ├── Runtime/                ← included in builds when referenced + AUDIOATLAS_DEBUG
    │   ├── Analysis/           MixerAnalyzer, SignalPathAnalyzer
    │   ├── Bootstrap/          AudioAtlasBootstrap (auto-init)
    │   ├── Core/               AudioRegistry, EventBus, MetricsCollector
    │   ├── Engine/             AudioAtlasRuntime (pooled playback + fades)
    │   ├── Events/             AudioEventDispatcher, EventInterceptor, ReplayEngine
    │   ├── Intelligence/       AudioIntelligenceEngine, IntelligenceAdvisor
    │   ├── Models/             All data model structs and classes
    │   └── Monitoring/         SourceTracker, VoiceStealTracker, HeatmapSampler,
    │                           ListenerTracker, LatencyProbe
    ├── Settings/               AudioAtlasSettings.asset
    └── Tests/
        └── Editor/             AudioRegistryTests, BugRuleSetTests, EventBusTests,
                                MetricsCollectorTests
```

---

## Assembly Reference Guide

Audio Atlas uses three assembly definitions with strict dependency rules:

| Assembly | In Player Build | Purpose |
|---|---|---|
| `AudioAtlas.Runtime` | **Conditional** (see below) | Registry, EventBus, tracking, engine |
| `AudioAtlas.Editor` | ❌ Never | All editor panels, scanner, graphs |
| `AudioAtlas.Examples` | ❌ Never | Demo scene scripts |

**To use the Runtime API in your scripts:** add `AudioAtlas.Runtime` to your asmdef's **Assembly Definition References**.

**Build inclusion rule:** `AudioAtlas.Runtime` compiles into your player build only if you reference it *and* define `AUDIOATLAS_DEBUG`. Without the define, all tracking code paths compile away. See [Scripting Defines](../configuration/settings-reference.md#scripting-defines).

---

## Uninstalling

1. Delete `Assets/AudioAtlas/`
2. Remove `AudioAtlas.Runtime` from any assembly references in your project
3. Remove `AUDIOATLAS_DEBUG` from `Project Settings → Player → Scripting Define Symbols` if set
4. Delete `Assets/AudioAtlas/Sessions/` if any saved sessions were created

---

[← Back to Docs](../README.md) · [Quick Start →](quick-start.md)
