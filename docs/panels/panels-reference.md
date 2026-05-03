# Panels Reference

Audio Atlas provides 20 panels in a docked editor window. The sidebar collapses to icon-only mode when the window is narrower than 900 px — drag the resize handle to adjust its width (160–220 px, persisted via `EditorPrefs`).

---

## MONITOR

### Live Sources

Tracks every `AudioSource` managed by `SourceTracker`. Updates every `UpdateIntervalFrames` (default: 3 frames).

**Columns:**

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

**Filters:** Playing · All · Looping — toggle in the toolbar strip above the list.  
**Search:** Filters by GameObject name or clip name (case-insensitive, contains).  
**Selection:** Click any row to set `AudioRegistry.SelectedSourceID` and fire `SourceSelected` event — consumed by Signal Chain and Source Inspector panels.

**Constraints:** Maximum visible sources = `MaxTrackedSources`. Sources beyond this are tracked in the ring buffer but not displayed. Increase the setting if you see truncation.

---

### Mixer Graph

Visual node graph of the full `AudioMixer` hierarchy. Populated by `MixerAnalyzer` via reflection on `AudioMixer.GetOutputAudioMixerGroup` and group parent chains.

Each node shows: group name, approximate volume level (bar indicator), child count. Nodes with no parent are root groups. Nodes with no children are leaf groups (typically where `AudioSource` components are routed).

**Interaction:** Click a node to fire `MixerGroupFocused` event and select the group in the Inspector.

**Constraints:** Requires at least one `AudioMixer` asset with at least one source routed to it. Groups unreachable from any `AudioSource` in the loaded scenes may not appear.

---

### Event Log

Ring buffer of play, stop, and voice steal events. Capacity: `EventRingBufferSize` (default 512). When full, oldest events are overwritten.

**Columns:** Timestamp · Type (PLAY / STOP / STEAL) · GameObject · Clip · Extra (e.g., steal victim name)

**Export CSV:** Writes the visible buffer to `Assets/AudioAtlas/Exports/event_log_YYYYMMDD_HHMMSS.csv`.

**Interaction:** Click any row to fire `EventSelected` — consumed by panels that need event-level context.

---

## ANALYZE

### Project Scanner

Runs `ScanCoordinator.RunAsync()` → `ArchitectureValidator` → 12 rules from `BugRuleSet`. Results appear grouped by severity (Error → Warning → Info).

Each finding row shows: rule ID, severity icon, summary, affected asset path, and a **Fix** button if `AudioFixer.CanFix(ruleID)` returns true.

**Fix behaviour:** Applies `AudioFixer.ApplyFix(record)`. All fixes register a named undo group. `Ctrl+Z` shows `"AudioAtlas Fix — AA-XXX"` in the Undo history.

**Simulation:** Click **Simulate** (where available) to preview changes without applying them.

See [Scanner Rules Reference](scanner-rules.md) for the complete rule catalogue.

---

### Signal Chain

Traces the audio signal from the currently selected `AudioSource` through the full mixer hierarchy to the output bus. Selection is driven by `AudioRegistry.SelectedSourceID` — click any row in Live Sources to set it.

**Rendered path:**
```
[AudioSource name] — vol: 0.8 pitch: 1.0
  └─ [MixerGroup: SFX] — vol: 0.9
       └─ [MixerGroup: Master] — vol: 1.0
            └─ [Output Bus]
```

Each step shows the cumulative volume and any effects in the chain (Reverb, Echo, Compressor, EQ).

**Constraints:** Requires `SignalPathAnalyzer` to have run at least once (triggered on `SourceSelected` event). The signal path is stored in `AudioRegistry.CurrentSignalPath`.

---

### Source Inspector

Deep-inspection view for the currently selected `AudioSource`. Compares serialized (on-disk) values against runtime values — useful for identifying drift caused by runtime modifications.

**Sections:** Identity · Clip · Spatial Configuration · Mixer Routing · Effects Chain · Runtime State

Highlights fields where runtime value differs from serialized value in amber.

---

### Scene Heatmap

3D spatial heatmap rendered as a Scene View overlay. Cells are sampled by `HeatmapSampler` every `HeatmapRefreshInterval` seconds (when `EnableHeatmapAutoRefresh` is true) or on manual refresh.

Cell colour: red (high activity) → yellow → green → blue (low activity).

**Data stored in:** `AudioRegistry.CurrentHeatmap` — a `HeatmapData` object with cell positions and weights.

**Requirements:** Play Mode active. `Gizmos` enabled in Scene View toolbar. `ShowSpatialGizmos: true` in Settings.

**Grid parameters:** `HeatmapCellSize` (world units per cell) · `HeatmapSampleHeight` (Y plane offset)

---

## INSIGHT

### Health & Metrics

Displays `AudioRegistry.HealthScore`. Updated on `HealthScoreUpdated` event (~0.5 s cadence while in Play Mode).

**Layout:**
- Large grade letter (A–F) with numeric score
- Trend arrow and 5-second delta
- Five sub-score bars: Voice, CPU, Design, Steal, Mixer
- KPI cards: Active Voices, CPU %, Steal Count, Looping Sources, Unrouted Sources

Sub-score weights: Voice 25 · CPU 25 · Design 25 · Steal 15 · Mixer 10  
Grade thresholds: A ≥90 · B ≥75 · C ≥60 · D ≥40 · F <40

---

### Insights (Intelligence Feed)

Displays insight cards from `AudioRegistry.IntelligenceInsights` (count: `IntelligenceInsightCount`). Updated on `IntelligenceInsightAvailable` event.

Cards are ranked by severity: Critical → High → Medium → Low. Each card shows:
- Title (e.g., "Clip spam detected: Footstep_Stone")
- Summary with specific data (e.g., "Fired 14× in 1 second")
- Category: ClipSpam · CoOccurrence · StealBurst · DesignIssue · LatencySpike

Critical cards also trigger `CriticalInsightDetected`, which can be subscribed to for custom handling.

---

## ASSETS

### Clip Search

Full-project `AudioClip` inventory. Populated by scanning `AssetDatabase` for all `.wav`, `.mp3`, `.ogg`, `.aiff`, and `.flac` files.

**Columns:** Name · Path · Size (disk) · Channels · Sample Rate (Hz) · Duration · Used (referencing source count)

**Filters:** All · Unused · Duplicates  
**Sort:** Any column  
**Export CSV:** Writes full inventory to `Assets/AudioAtlas/Exports/clip_search_YYYYMMDD.csv`  
**Bulk Delete:** Select multiple unused clips → Delete (with confirmation dialog)

Fires `ClipAnalysisCompleted` when scan finishes.

---

### Memory Profiler

Per-clip runtime memory analysis. For each clip: file size on disk, decompressed size in memory (when `DecompressOnLoad`), compression ratio, and `AudioClipLoadType`.

**Load type memory impact:**

| Load Type | Memory footprint | I/O pattern |
|---|---|---|
| `DecompressOnLoad` | Full PCM in RAM | Decompress at import, RAM hit |
| `CompressedInMemory` | Compressed in RAM | Decompress on play, CPU hit |
| `Streaming` | Minimal | Disk reads during playback |

Fires `MemoryProfileUpdated` when analysis refreshes.

---

### Asset References

Cross-reference map: clip → list of `AudioSource` components referencing it. Useful before deleting or modifying a clip to understand blast radius.

Select any clip in the list to see every consumer: scene path, GameObject name, and component index.

---

### Budget Guard

Per-platform voice and memory budget enforcement. Configure budgets for each `RuntimePlatform` via `PlatformBudgetProfile` objects.

During Play Mode, the panel shows live usage vs. budget for the current platform. When either voice count or memory usage exceeds the threshold, it raises `EventBus.Raise(AudioAtlasEvent.BudgetViolation)` with a platform-specific payload.

---

### Assets Browser

Project window view scoped to audio asset types only: `AudioClip`, `AudioMixer`, `AudioMixerGroup`, `AudioAtlasSettings`, `AudioAtlasConfigAsset`. Mirrors standard Project window interaction — double-click to open, right-click for context menu.

---

## TOOLS

### Session Notes

Freeform rich-text notes editor. Notes are stored in `AudioRegistry.ActiveSession.Notes` and serialized with the session JSON when saved.

Notes persist across Play Mode sessions within the same editor session. They are cleared on `SessionCleared` event.

---

### Bookmarks

Persist `AudioSource` `GameObject` references across Play Mode sessions via `EditorPrefs`. Bookmarked sources appear at the top of the Live Sources list with a highlight border.

Click **+ Bookmark** to add the currently selected source. Click **✕** on any entry to remove it.

---

### Waveform Scope

Real-time DSP waveform display. `WaveScopePanel` renders the most recent 1024 DSP samples as an oscilloscope trace.

**Implementation:** `WaveScopePanel` uses `AudioAtlas.Editor.Utilities.WaveformRenderer`, which reads from a `float[]` buffer written by `OnAudioFilterRead`. Thread safety is achieved via `Volatile.Read` / `Thread.MemoryBarrier` — no lock contention on the DSP thread.

**Constraints:** Display pauses when no audio is playing. The scope shows the mixed master output, not individual source waveforms.

Fires `WaveformScopeUpdated` each repaint cycle when active.

---

### Session Diff

Compares two saved `AudioSession` JSON files across six behavioral dimensions.

**Diff dimensions:**
1. **New events** — clips playing in Session B but absent in Session A
2. **Missing events** — clips present in A but absent in B
3. **Volume changes** — clips with >3 dB average volume difference
4. **Trigger count changes** — clips with >20% play count change
5. **Steal changes** — clips with increased or decreased steal frequency
6. **Latency changes** — sessions with >5 ms latency delta

Load sessions using the file pickers. The diff renders inline with colour-coded change indicators. Fires `DiffCompleted` when computation finishes.

---

### Voice Drop Sim

Deterministic voice drop probability simulator using `VoiceDropSimulator.Run(scenario)`.

**Configure:**
- `VoiceLimit` — maximum simultaneous voices
- Per-source entries: min distance, max distance, volume, priority, `AudioRolloffMode`

**Output:** Drop probability (0–100%) per source, peak voice count, total sample count.

**Algorithm:** For each of 32 distance sample points, all sources are ranked by audibility score. Sources ranked beyond `VoiceLimit` are counted as dropped. `dropProbability = dropCount / totalSamples`. Fully deterministic — no randomisation.

---

## BUILD

### Build History

Chronological log of `BuildRecord` entries. Each entry captures:
- Build number (sequential per project)
- Health grade and score
- Total errors / warnings / infos
- Unity version and timestamp
- Voice count and memory usage at scan time

A sparkline chart renders the last 10 health scores as a mini trend line. Records are persisted in `EditorPrefs` and survive domain reloads.

---

## FOOTER

### Settings

Opens `Project Settings → AudioAtlas` — the `AudioAtlasSettingsProvider` page that mirrors `AudioAtlasSettings.asset`.

### About

Displays publisher, version, Unity compatibility, pipeline support, and namespace. Links to documentation, Asset Store, Discord, and email support. Includes **Report a Bug** button that pre-populates an email with Unity version and platform info.

---

[← Architecture](../core-systems/architecture.md) · [Scanner Rules →](scanner-rules.md)
