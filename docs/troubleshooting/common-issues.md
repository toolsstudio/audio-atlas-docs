# Troubleshooting

Root causes and resolutions for every commonly reported issue.

---

## Live Sources Panel is Empty in Play Mode

**Root cause A — Bootstrap did not initialise**

Check the Console for `[AudioAtlas] CIBridge` or `[AudioAtlas]` startup messages. If absent, the bootstrap did not run.

- Verify `AUDIOATLAS_DEBUG` is defined (for player builds) or that you are in the Unity Editor
- Confirm `AudioAtlas.Runtime` is referenced by your game assembly's asmdef
- Check `Assets/AudioAtlas/Resources/AudioAtlasConfig.asset` exists — if missing, Bootstrap creates a default config but logs a warning

**Root cause B — MaxTrackedSources reached**

If more sources are playing than `MaxTrackedSources` (default 128), excess sources are silently dropped from the display. Increase the value in `AudioAtlasSettings.asset`.

**Root cause C — Refresh rate too slow**

Sources that play and stop between refresh intervals will not appear. Decrease `UpdateIntervalFrames` to `1` for debugging rapid one-shots.

---

## "Missing Script" Warnings on Scene Load

**Root cause — Meta file GUID mismatch**

Unity assigned random GUIDs to scripts during a previous import because `.meta` files were absent or incorrect. The scene references GUIDs that no longer match.

**Resolution:**
1. Delete the `Library/` folder
2. Reimport the complete `.unitypackage` — do not import over an existing installation
3. If the warnings persist, contact [tools.studio@zohomail.in](mailto:tools.studio@zohomail.in) with your Unity version

---

## ExtensionOfNativeClass Error

**Symptom:** Console shows `AudioAtlas.Editor.Windows.AudioAtlasWelcome is missing the class attribute 'ExtensionOfNativeClass'`

**Root cause:** A partial stub of `AudioAtlasWelcome.cs` was left in the project from a previous incomplete update. The class is not a valid `EditorWindow` subclass in its current form.

**Resolution:** Delete `Assets/AudioAtlas/Editor/Windows/AudioAtlasWelcome.cs` and its `.meta` file. The Welcome window functionality was consolidated into the About tab in v1.0.0. If you do not see the files, a clean reimport will resolve it.

---

## Health Score Shows Grade F on First Open

**Root cause — Score requires Play Mode data**

The health score is computed by `AudioIntelligenceEngine`, which runs only when `AudioRegistry.IsInitialized` is true (i.e., in Play Mode with the runtime active). In Edit Mode, `AudioRegistry.HealthScore.IsValid` is false and the panel shows an unscored state.

**Resolution:** Enter Play Mode. The first health tick fires within ~0.5 seconds of bootstrap completing.

---

## Waveform Scope Shows a Flat Line

**Root cause A — No audio playing**  
The waveform scope reads the DSP output buffer. If no `AudioSource` is playing, the buffer is silent.

**Root cause B — AudioListener disabled or missing**  
Without an `AudioListener` in the scene, Unity does not process spatial audio. The DSP buffer will be silent or missing.

**Root cause C — Audio muted in Unity Editor**  
Check the **Mute Audio** toggle in the Game View toolbar (speaker icon). If muted, the DSP buffer is silent.

---

## Mixer Graph Shows No Nodes

**Root cause — No mixer groups connected to active sources**  
`MixerAnalyzer` builds the graph by walking `AudioMixer` groups reachable from `AudioSource.outputAudioMixerGroup` references in loaded scenes. `AudioMixer` assets that exist in the project but have no sources routed through them in any currently loaded scene will not appear.

**Resolution:** Ensure at least one `AudioSource` in the loaded scene has an `AudioMixerGroup` assigned and that group belongs to an `AudioMixer` asset.

---

## Scene Heatmap Not Visible

**Checklist:**
- [ ] Play Mode is active (`HeatmapSampler` only runs in Play Mode)
- [ ] **Gizmos** is enabled in the Scene View toolbar
- [ ] `ShowSpatialGizmos: true` in `AudioAtlasSettings`
- [ ] `EnableHeatmapAutoRefresh: true`, or manual refresh triggered via the panel button
- [ ] At least one `AudioSource` is playing — the heatmap samples source positions, not listener position

---

## CI Scan Exits with Code 2

**Root cause A — Compilation errors**  
If the project has compilation errors, `ScanCoordinator.RunBlocking()` cannot execute. The batch-mode log will show the compile error before the scan attempt. Fix all compile errors before running the CI scan.

**Root cause B — Scan timeout**  
The default timeout for `RunBlocking()` is 120 seconds. Large projects with many `AudioClip` assets may exceed this.

Set `AUDIOATLAS_CI_TIMEOUT_SECONDS` environment variable to a higher value:
```bash
export AUDIOATLAS_CI_TIMEOUT_SECONDS=300
```

**Root cause C — Unity license issue**  
Batch mode requires a valid Unity license activated on the CI machine. Check the Unity batch-mode log for license-related errors.

---

## AudioAtlasSettings Reset After Domain Reload

**Root cause — Asset not saved**  
If Unity exits uncleanly (crash, force-quit) before the settings asset is written to disk, the asset resets to defaults on next load.

`AudioAtlasSettings.GetOrCreate()` creates the asset only if it is absent. Modifications made through the Settings panel are saved immediately via `EditorUtility.SetDirty` + `AssetDatabase.SaveAssets`.

**Resolution:** Make changes, then force-save via `File → Save Project` or `Ctrl+S` before closing Unity.

---

## URP Global Settings Message on First Open

**Symptom:** `URP Global Settings Asset has been created for you...`

This is a **Unity engine message**, not from Audio Atlas. It is emitted once by `UnityEditor.AssetPostprocessingInternal` when a project opens without an existing `UniversalRenderPipelineGlobalSettings.asset`. It requires no action and will not reappear after the first import completes. You cannot suppress it from plugin code.

---

## Performance Impact with Window Open

**Root cause — Waveform Scope or Mixer Graph repainting each frame**

The Waveform Scope triggers a panel repaint on every `WaveformScopeUpdated` event, which fires each update cycle. The Mixer Graph recalculates layout on each repaint when many nodes are visible.

**Resolution:**
- Close panels you are not actively using
- Increase `UpdateIntervalFrames` from 3 to 10 in Settings to reduce repaint frequency
- Resize the window smaller — the sidebar icon-only mode reduces drawing surface significantly

---

## EventBus Handler Not Firing

**Root cause A — Subscribed before bootstrap, cleared by `EventBus.Clear()`**  
`EventBus.Clear()` is called on `SessionCleared` (Play Mode exit). If you subscribe once in `Awake()` on a persistent object that survives Play Mode transitions, you must re-subscribe after each session start.

Subscribe to `SessionStarted` to re-register:
```csharp
EventBus.Subscribe(AudioAtlasEvent.SessionStarted, _ =>
{
    EventBus.Subscribe(AudioAtlasEvent.BugReportGenerated, OnScanDone);
});
```

**Root cause B — Slot limit reached**  
Each event supports 32 handler slots. If all slots are filled, new subscriptions are silently dropped and a `LogWarning` is emitted. Check the Console for `EventBus slot limit reached`.

---

[← Configuration](../configuration/settings-reference.md) · [FAQ →](faq.md)
