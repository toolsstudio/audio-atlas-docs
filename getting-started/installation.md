# Installation

## Requirements

| Requirement | Minimum |
|---|---|
| Unity Version | **6000.0** (Unity 6) |
| .NET Compatibility | Standard 2.1 |
| Scripting Backend | Mono or IL2CPP |
| Render Pipeline | Built-in, URP, or HDRP |
| Editor Platform | Windows, macOS, Linux |

---

## Acquiring the Package

Audio Atlas is a paid commercial asset. A valid purchase is required before use.

> **Source code repository access is not included with purchase.**  
> The asset is delivered exclusively as a Unity package. The development repository is private.  
> Repository access may be granted separately upon request — [Contact Support](mailto:tools.studio@zohomail.in).

---

## Install Method 1 — Unity Asset Store (Recommended)

1. Purchase Audio Atlas from the Unity Asset Store
2. Open `Window → Package Manager` in Unity
3. Select **My Assets** from the dropdown
4. Locate **Audio Atlas** and click **Download**, then **Import**
5. Leave all items checked in the Import dialog and click **Import**

Unity places all content under `Assets/AudioAtlas/`.

---

## Install Method 2 — Manual .unitypackage

1. Open your Unity project
2. Use `Assets → Import Package → Custom Package` and select the `.unitypackage`, or drag it into the Project window
3. Leave all items checked and click **Import**

No repository cloning or source checkout is required or provided.

---

## First Launch

```
Window → Audio → Audio Atlas     Ctrl+Shift+A
```

On first launch, Audio Atlas creates `Assets/AudioAtlas/Settings/AudioAtlasSettings.asset` with defaults and opens the main window with **Live Sources** as the active panel. No scene changes. No component placement. No code required.

---

## Package Layout

```
Assets/
└── AudioAtlas/
    ├── Documentation/          Local docs (README, CHANGELOG, LICENSE)
    ├── Editor/                 Editor-only — never in player builds
    │   ├── Analysis/           SessionDiffEngine, VoiceDropSimulator
    │   ├── Automation/         AudioFixer, CIBridge, BatchProcessor, ConfigCreator
    │   ├── Branding/           Package-safe asset loader
    │   ├── Build/              BuildHistory, BuildRecord, HistoryPanel
    │   ├── Features/           Asset panels, Session tools, Waveform, Bookmarks
    │   ├── Graph/              MixerGraphView, MixerGroupNode, MixerEdge
    │   ├── Intelligence/       IntelligenceFeedPanel
    │   ├── Knowledge/          ProjectKnowledgeGraph
    │   ├── Panels/             Core panel implementations
    │   ├── Scan/               ScanCoordinator, ArchitectureValidator
    │   ├── SceneOverlay/       SceneOverlay, AudioGizmoRenderer
    │   ├── Serialization/      SessionSerializer
    │   ├── Settings/           AudioAtlasSettings, SettingsProvider
    │   ├── Utilities/          PanelIcons, PanelStyles, WaveformRenderer
    │   ├── Validation/         BugRuleSet, BugReportPanel
    │   └── Windows/            AudioAtlasWindow, AudioAtlasWindowLayout
    ├── Examples/
    │   └── DemoScene/          Playable demo with annotated audio sources
    ├── Resources/              AudioAtlasConfig.asset (runtime config)
    ├── Runtime/                Conditional — active when AUDIOATLAS_DEBUG is defined
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

| Assembly | In Player Build | Purpose |
|---|---|---|
| `AudioAtlas.Runtime` | **Conditional** (see below) | Registry, EventBus, tracking, engine |
| `AudioAtlas.Editor` | ❌ Never | All editor panels, scanner, graphs |
| `AudioAtlas.Examples` | ❌ Never | Demo scene scripts |

To use the Runtime API in your scripts, add `AudioAtlas.Runtime` to your asmdef's **Assembly Definition References**.

`AudioAtlas.Runtime` compiles into your player build only if you reference it *and* define `AUDIOATLAS_DEBUG`. Without the define, all tracking code paths compile away. See [Scripting Defines](../configuration/settings-reference.md#scripting-defines).

---

## Uninstalling

1. Delete `Assets/AudioAtlas/`
2. Remove `AudioAtlas.Runtime` from any assembly references
3. Remove `AUDIOATLAS_DEBUG` from `Project Settings → Player → Scripting Define Symbols` if set
4. Delete `Assets/AudioAtlas/Sessions/` if any saved sessions were created

---

[Back to Docs](../README.md) · [Quick Start →](quick-start.md)
