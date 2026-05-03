# Panels Reference

Audio Atlas provides 20 panels in a docked editor window. The sidebar collapses to icon-only mode when the window is narrower than 900 px — drag the resize handle to adjust its width (160–220 px, persisted via `EditorPrefs`).

---

## MONITOR

### Live Sources

Tracks every `AudioSource` managed by `SourceTracker`. Updates every `UpdateIntervalFrames` (default: 3 frames).

| Column | Source field | Notes |
|---|---|---|
| State | `isPlaying`, `loop` | PLAY · LOOP · STOP |
| GameObject | `gameObject.name` (cached) | Click to ping in Hierarchy |
| Clip | `clip.name` | Empty if no clip assigned |
| Vol | `volume` | 0.0–1.0 |
| Pitch | `pitch` | Multiplier |
| Blend | `spatialBlend` | 0 = 2D, 1 = full 3D |
| Dist | world distance to `AudioListener` | Metres |
| Pri | `priority` | 0 = highest priority |
| Group | `outputAudioMixerGroup.name` | Empty if unassigned |

**Filters:** Playing · All · Looping. **Search:** by GameObject name or clip name (case-insensitive, contains). **Selection:** Click any row to set `AudioRegistry.SelectedSourceID` and fire `SourceSelected` — consumed by Signal Chain and Source Inspector.

**Constraint:** Maximum visible sources = `MaxTrackedSources`.

---

### Mixer Graph

Visual node graph of the full `AudioMixer` hierarchy. Populated by `MixerAnalyzer` via reflection. Each node shows group name, approximate volume level, and child count.

**Interaction:** Click a node to fire `MixerGroupFocused` and select the group in the Inspector.

**Constraint:** Requires at least one `AudioMixer` asset with at least one source routed to it.

---

### Event Log

Ring buffer of play, stop, and voice steal events. Capacity: `EventRingBufferSize` (default 512).

**Columns:** Timestamp · Type (PLAY / STOP / STEAL) · GameObject · Clip · Extra

**Export CSV:** Writes to `Assets/AudioAtlas/Exports/event_log_YYYYMMDD_HHMMSS.csv`.

---

## ANALYZE

### Project Scanner

Runs `ScanCoordinator.RunAsync()` → `ArchitectureValidator` → 12 rules from `BugRuleSet`. Results grouped by severity (Error → Warning → Info).

**Fix behaviour:** `AudioFixer.ApplyFix(record)` — all fixes register a named undo group. `Ctrl+Z` shows `"AudioAtlas Fix — AA-XXX"` in Undo history.

See [Scanner Rules Reference](scanner-rules.md) for the complete rule catalogue.

---

### Signal Chain

Traces the audio signal from the currently selected `AudioSource` through the full mixer hierarchy to the output bus.

```
[AudioSource name] — vol: 0.8 pitch: 1.0
  └─ [MixerGroup: SFX] — vol: 0.9
       └─ [MixerGroup: Master] — vol: 1.0
            └─ [Output Bus]
```

Each step shows cumulative volume and any effects in the chain.

---

### Source Inspector

Deep-inspection view for the currently selected `AudioSource`. Compares serialized vs. runtime values — fields with drift are highlighted in amber.

**Sections:** Identity · Clip · Spatial Configuration · Mixer Routing · Effects Chain · Runtime State

---

### Scene Heatmap

3D spatial heatmap rendered as a Scene View overlay. Cell colour: red (high activity) → yellow → green → blue (low activity).

**Requirements:** Play Mode active · Gizmos enabled in Scene View · `ShowSpatialGizmos: true` in Settings · at least one `AudioSource` playing.

---

## INSIGHT

### Health & Metrics

Displays `AudioRegistry.HealthScore`. Updated on `HealthScoreUpdated` (~0.5 s cadence in Play Mode).

- Large grade letter (A–F) with numeric score
- Trend arrow and 5-second delta
- Five sub-score bars: Voice, CPU, Design, Steal, Mixer
- KPI cards: Active Voices, CPU %, Steal Count, Looping Sources, Unrouted Sources

---

### Insights (Intelligence Feed)

Displays insight cards from `AudioRegistry.IntelligenceInsights`. Cards ranked by severity: Critical → High → Medium → Low.

Each card shows a title, a summary with specific data, and a category: ClipSpam · CoOccurrence · StealBurst · DesignIssue · LatencySpike.

Critical cards trigger `CriticalInsightDetected` — subscribable for custom reactions.

---

## ASSETS

### Clip Search

Full-project `AudioClip` inventory. Scans `AssetDatabase` for `.wav`, `.mp3`, `.ogg`, `.aiff`, and `.flac` files.

**Columns:** Name · Path · Size (disk) · Channels · Sample Rate · Duration · Used (referencing source count)

**Filters:** All · Unused · Duplicates. **Export CSV.** **Bulk Delete** with confirmation dialog.

---

### Memory Profiler

Per-clip runtime memory analysis with load type and compression breakdown.

| Load Type | Memory | I/O |
|---|---|---|
| `DecompressOnLoad` | Full PCM in RAM | Decompress at import |
| `CompressedInMemory` | Compressed in RAM | Decompress on play |
| `Streaming` | Minimal | Disk reads during playback |

---

### Asset References

Cross-reference map: clip → list of `AudioSource` components referencing it. Select any clip to see every consumer: scene path, GameObject name, component index.

---

### Budget Guard

Per-platform voice and memory budget enforcement via `PlatformBudgetProfile` objects. When a threshold is exceeded, `EventBus.Raise(AudioAtlasEvent.BudgetViolation)` fires.

---

### Assets Browser

Project window scoped to audio asset types only: `AudioClip`, `AudioMixer`, `AudioMixerGroup`, `AudioAtlasSettings`, `AudioAtlasConfigAsset`.

---

## TOOLS

### Session Notes

Freeform notes editor. Notes persist in `AudioRegistry.ActiveSession.Notes` and are serialized with the session JSON. Cleared on `SessionCleared`.

### Bookmarks

Persists `AudioSource` `GameObject` references across Play Mode sessions via `EditorPrefs`. Bookmarked sources appear at the top of Live Sources with a highlight border.

### Waveform Scope

Real-time DSP waveform. Renders the most recent 1024 DSP samples as an oscilloscope trace. Thread safety via `Volatile.Read` / `Thread.MemoryBarrier`. Zero per-frame allocations. Displays mixed master output, not individual sources.

### Session Diff

Compares two saved `AudioSession` JSON files across six dimensions: new events, missing events, volume changes (>3 dB), trigger count changes (>20%), steal changes, and latency changes (>5 ms).

### Voice Drop Sim

Deterministic voice drop probability. No randomisation. Samples each source's audibility at 32 distance points; sources ranked beyond `VoiceLimit` are counted as dropped. `dropProbability = dropCount / totalSamples`.

---

## BUILD

### Build History

Chronological log of `BuildRecord` entries: build number, health grade and score, total errors/warnings/infos, Unity version, timestamp, voice count, memory usage. Sparkline chart shows last 10 health scores. Persisted in `EditorPrefs` — survives domain reloads.

---

[← Architecture](../core-systems/architecture.md) · [Scanner Rules →](scanner-rules.md)
