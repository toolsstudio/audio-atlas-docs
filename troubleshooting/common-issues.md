# Troubleshooting

Root causes and resolutions for every commonly reported issue.

---

## Live Sources Panel is Empty in Play Mode

**Root cause A — Bootstrap did not initialise**

Check the Console for `[AudioAtlas]` startup messages. If absent, the bootstrap did not run.

- Verify `AUDIOATLAS_DEBUG` is defined (for player builds) or that you are in the Unity Editor
- Confirm `AudioAtlas.Runtime` is referenced by your game assembly's asmdef
- Check `Assets/AudioAtlas/Resources/AudioAtlasConfig.asset` exists

**Root cause B — MaxTrackedSources reached**

If more sources are playing than `MaxTrackedSources` (default 128), excess sources are silently dropped from the display. Increase the value in `AudioAtlasSettings.asset`.

**Root cause C — Refresh rate too slow**

Sources that play and stop between refresh intervals will not appear. Decrease `UpdateIntervalFrames` to `1` for debugging rapid one-shots.

---

## "Missing Script" Warnings on Scene Load

**Root cause — Meta file GUID mismatch**

**Resolution:**
1. Delete the `Library/` folder
2. Reimport the complete `.unitypackage` — do not import over an existing installation
3. If warnings persist, [Contact Support](mailto:tools.studio@zohomail.in) with your Unity version

---

## ExtensionOfNativeClass Error

**Symptom:** `AudioAtlas.Editor.Windows.AudioAtlasWelcome is missing the class attribute 'ExtensionOfNativeClass'`

**Resolution:** Delete `Assets/AudioAtlas/Editor/Windows/AudioAtlasWelcome.cs` and its `.meta` file. This stub was left from a previous incomplete update; the Welcome window was consolidated into the About tab in v1.0.0.

---

## Health Score Shows Grade F on First Open

**Root cause — Score requires Play Mode data**

`AudioIntelligenceEngine` only runs when `AudioRegistry.IsInitialized` is true (Play Mode with runtime active). In Edit Mode, `AudioRegistry.HealthScore.IsValid` is false.

**Resolution:** Enter Play Mode. The first health tick fires within ~0.5 seconds.

---

## Waveform Scope Shows a Flat Line

**Root cause A — No audio playing.** The scope reads the DSP output buffer. Silence produces a flat line.

**Root cause B — AudioListener disabled or missing.** Without an `AudioListener`, Unity does not process spatial audio — the DSP buffer is silent.

**Root cause C — Audio muted in Unity Editor.** Check the **Mute Audio** toggle in the Game View toolbar (speaker icon).

---

## Mixer Graph Shows No Nodes

**Root cause — No mixer groups connected to active sources**

`MixerAnalyzer` builds the graph by walking groups reachable from `AudioSource.outputAudioMixerGroup` in loaded scenes. Mixers with no sources routed through them will not appear.

**Resolution:** Ensure at least one `AudioSource` in the loaded scene has an `AudioMixerGroup` assigned.

---

## Scene Heatmap Not Visible

- [ ] Play Mode is active
- [ ] **Gizmos** is enabled in the Scene View toolbar
- [ ] `ShowSpatialGizmos: true` in `AudioAtlasSettings`
- [ ] `EnableHeatmapAutoRefresh: true`, or manual refresh triggered via the panel button
- [ ] At least one `AudioSource` is playing

---

## CI Scan Exits with Code 2

**Root cause A — Compilation errors.** Fix all compile errors before running the CI scan.

**Root cause B — Scan timeout.** Default timeout: 120 seconds. Increase via:
```bash
export AUDIOATLAS_CI_TIMEOUT_SECONDS=300
```

**Root cause C — Unity license issue.** Batch mode requires a valid Unity license on the CI machine.

---

## AudioAtlasSettings Reset After Domain Reload

**Root cause — Asset not saved before Unity closed**

Changes are saved immediately via `EditorUtility.SetDirty` + `AssetDatabase.SaveAssets`. Force-save via `File → Save Project` or `Ctrl+S` before closing Unity.

---

## URP Global Settings Message on First Open

`URP Global Settings Asset has been created for you...` is a **Unity engine message**, not from Audio Atlas. It fires once when opening a project without an existing `UniversalRenderPipelineGlobalSettings.asset`. No action required.

---

## Performance Impact with Window Open

**Root cause — Waveform Scope or Mixer Graph repainting each frame**

- Close panels you are not actively using
- Increase `UpdateIntervalFrames` from 3 to 10 in Settings
- Resize the window smaller — sidebar icon-only mode reduces drawing surface

---

## EventBus Handler Not Firing

**Root cause A — Cleared by `EventBus.Clear()`**

`EventBus.Clear()` is called on `SessionCleared` (Play Mode exit). Re-subscribe after each session start:

```csharp
EventBus.Subscribe(AudioAtlasEvent.SessionStarted, _ =>
{
    EventBus.Subscribe(AudioAtlasEvent.BugReportGenerated, OnScanDone);
});
```

**Root cause B — Slot limit reached.** Each event supports 32 handler slots. Check the Console for `EventBus slot limit reached`.

---

[← Configuration](../configuration/settings-reference.md) · [FAQ →](faq.md)
