# Configuration Reference

Audio Atlas has two configuration assets: `AudioAtlasSettings` (editor-side) and `AudioAtlasConfig` (runtime-side). Both are `ScriptableObject`-backed and created automatically on first launch.

---

## AudioAtlasSettings

**Location:** `Assets/AudioAtlas/Settings/AudioAtlasSettings.asset`  
**Scope:** Editor only — never included in player builds  
**Access:** `Window → Audio → Audio Atlas → Settings`  
**API:** `AudioAtlasSettings.GetOrCreate()` · `AudioAtlasSettings.GetConfig()`

### Embedded Config (AudioAtlasConfig)

| Field | Default | Type | Description |
|---|---|---|---|
| `MaxTrackedSources` | `128` | int | Ring buffer capacity for `SourceSnapshot[]` |
| `MaxMixerGroups` | `64` | int | Ring buffer capacity for `MixerGroupSnapshot[]` |
| `EventRingBufferSize` | `512` | int | Capacity of `AudioEvent[]` ring buffer |
| `StealRingBufferSize` | `256` | int | Capacity of `VoiceSteal[]` ring buffer |
| `UpdateIntervalFrames` | `3` | int | Source tracker poll rate (frames) |
| `UIRefreshIntervalFrames` | `3` | int | Editor window repaint interval (frames) |
| `SourceCountWarning` | `32` | int | Voice count that triggers yellow status indicator |
| `SourceCountCritical` | `64` | int | Voice count that triggers red status indicator |
| `VoiceUsageWarningRatio` | `0.8` | float | Fraction of voice budget that triggers budget warning |
| `AudioCpuWarningPercent` | `8.0` | float | Audio CPU % threshold for warning state |
| `AudioCpuCriticalPercent` | `15.0` | float | Audio CPU % threshold for critical state |
| `LoopingSourceWarning` | `16` | int | Simultaneous looping sources threshold |
| `EnableHeatmapAutoRefresh` | `false` | bool | Auto-refresh heatmap without user triggering |
| `HeatmapRefreshInterval` | `5.0` | float | Seconds between auto-refresh ticks |
| `HeatmapCellSize` | `2.0` | float | World-unit size of each heatmap grid cell |
| `HeatmapSampleHeight` | `0.0` | float | Y offset for heatmap sampling plane |
| `LatencyProbeBurstCount` | `10` | int | Samples per latency burst measurement |
| `LatencyProbeInterval` | `5.0` | float | Seconds between latency probe bursts |
| `AutoSaveSession` | `false` | bool | Save session JSON automatically on Play Mode exit |
| `AutoSavePath` | `Assets/AudioAtlas/Sessions` | string | Output directory for auto-saved sessions |

### Editor Display Settings

| Field | Default | Description |
|---|---|---|
| `ShowSpatialGizmos` | `true` | Draw distance sphere gizmos on selected 3D sources |
| `ShowSourceLabels` | `true` | Draw world-space labels above tracked sources in Scene View |
| `ShowAttenuation` | `true` | Draw attenuation curve overlay on 3D sources |
| `ShowListenerMarker` | `true` | Draw `AudioListener` position indicator gizmo |
| `EditorUpdateOverride` | `0` | Override `UpdateIntervalFrames` for this session (0 = use config value) |

---

## AudioAtlasConfig (Runtime)

**Location:** `Assets/AudioAtlas/Resources/AudioAtlasConfig.asset`  
**Scope:** Runtime — loaded by `AudioAtlasBootstrap` via `Resources.Load<AudioAtlasConfigAsset>("AudioAtlasConfig")`  
**Create:** `Window → Audio → Audio Atlas → Create Config Asset`

| Field | Default | Description |
|---|---|---|
| `MaxTrackedSources` | `128` | Must match or exceed editor setting for consistent display |
| `MaxMixerGroups` | `64` | Mixer group tracking capacity |
| `EventRingBufferSize` | `512` | Event log ring buffer size |
| `StealRingBufferSize` | `256` | Voice steal ring buffer size |

> **Tip:** If you increase `MaxTrackedSources` in the editor settings, update the runtime config asset to match. A mismatch causes Live Sources to show a truncated list while the runtime tracks more.

---

## Scripting Defines {#scripting-defines}

Add/remove via `Project Settings → Player → Scripting Define Symbols`.

| Define | Scope | Effect |
|---|---|---|
| `AUDIOATLAS_DEBUG` | Runtime + Editor | Activates full monitoring in player builds. **Remove for release.** |
| `AUDIOATLAS_DEV` | Editor only | Enables internal developer warnings. Not for end users. |

### Release Build Checklist

- [ ] `AUDIOATLAS_DEBUG` is **not** in your Scripting Define Symbols
- [ ] Player build log shows no `AudioAtlas.Runtime` in the assembly list
- [ ] `Assets/AudioAtlas/Sessions/` is excluded from build

### Development Build Checklist

- [ ] `AUDIOATLAS_DEBUG` defined
- [ ] `AudioAtlas.Runtime` referenced in your game assembly's asmdef
- [ ] `AudioAtlasConfig.asset` present in `Resources/`

---

## Project Settings Integration

Audio Atlas registers a `SettingsProvider` under `Project Settings → AudioAtlas`. Any modification raises `EventBus.Raise(AudioAtlasEvent.SettingsChanged)` so panels react immediately.

---

## Resetting to Defaults

**Editor settings:**
```
Delete Assets/AudioAtlas/Settings/AudioAtlasSettings.asset
Reopen the window — asset is auto-recreated with defaults
```

**Runtime config:**
```
Delete Assets/AudioAtlas/Resources/AudioAtlasConfig.asset
Use: Window → Audio → Audio Atlas → Create Config Asset
```

---

[← Panels Reference](../panels/panels-reference.md) · [Runtime API →](../api/runtime-api.md)
