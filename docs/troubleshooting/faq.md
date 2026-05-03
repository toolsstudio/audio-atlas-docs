# FAQ

---

## General

**Does Audio Atlas add overhead to shipped games?**

No. `AudioAtlas.Editor` is editor-only and never compiled into a player build. `AudioAtlas.Runtime` is conditionally compiled behind `#if UNITY_EDITOR || AUDIOATLAS_DEBUG` throughout its source. When `AUDIOATLAS_DEBUG` is not defined, the compiler eliminates all tracking code. `AudioAtlasBootstrap` becomes a no-op method with a single `return` statement. Your shipped binary contains zero Audio Atlas code unless you explicitly opt in.

---

**Do I need to modify my scenes or add components?**

No. Audio Atlas uses `[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]` to boot before any `Awake()` fires. It creates its own root `GameObject` with `DontDestroyOnLoad` and `HideFlags.HideAndDontSave`. Nothing is added to any scene on disk.

---

**Does Audio Atlas support HDRP, URP, and Built-in simultaneously?**

Yes. Audio Atlas has no render pipeline dependencies. It uses IMGUI for its editor window — not UIToolkit, not the Game View — and has no dependency on `GraphicsSettings`, `RenderPipeline`, or any shader. The `RenderPipelineShaderResolver` utility is only used for the Scene Heatmap overlay material (gizmo rendering), and it falls back cleanly on any pipeline.

---

**Which Unity versions are supported?**

Unity 6000.0 (Unity 6) and later. The tool targets Unity 6 API surface including the latest `AudioMixer` reflection introspection behaviour and `AudioRegistry` design. It does not support Unity 2021 or 2022.

---

**Can I use Audio Atlas with Addressables or Asset Bundles?**

The runtime tracking (Live Sources, Event Log, Health Score) works regardless of how your audio is loaded. The Project Scanner, however, can only audit `AudioClip` assets loaded into the `AssetDatabase` at scan time. Clips inside unloaded Addressable groups or Asset Bundles will not appear in scanner results. Load the relevant groups into the editor (via the Addressables window) before scanning to include them.

---

## Licensing

**Can I use Audio Atlas in client work?**

Yes. A single license covers use in any project owned or delivered by the licensee, including client work, provided the source code is not delivered to the client. The client receives the compiled output — not the Audio Atlas source.

**Can multiple developers at my studio use one license?**

A single license covers one individual developer or one studio (all employees of a registered entity). For multi-seat enterprise licensing, contact [tools.studio@zohomail.in](mailto:tools.studio@zohomail.in).

**Can I include the source in a public repository?**

No. The license prohibits including Audio Atlas source in any publicly accessible repository. Add `Assets/AudioAtlas/` to your `.gitignore` for public-facing projects, or keep the repository private.

Source code repository access (access to the development repository) is not included with purchase. It may be granted separately upon request — contact [tools.studio@zohomail.in](mailto:tools.studio@zohomail.in).

**Can I modify the source code?**

Yes, for your own internal use. You may not redistribute modified versions.

---

## Technical

**How does voice steal detection work?**

`VoiceStealTracker` polls `AudioRegistry.SourceSnapshots` every frame. It maintains a record of which sources were playing in the previous frame. When a source transitions from `isPlaying = true` to `isPlaying = false` while its `clip` still has remaining `time` and its `clip.length - time > 0.05f` (not near natural end), and a new source began playing in the same frame with a higher priority, a steal event is written to `AudioRegistry.StealRingBuffer` via `AudioRegistry.ClaimStealSlot()` (thread-safe `Interlocked.Increment`).

**Does Audio Atlas affect garbage collection?**

The runtime hot path (source tracking, event recording, health score tick) allocates zero managed memory. Pre-allocated arrays, structs, and cached strings eliminate GC pressure on the tracking path. IMGUI panel repaints do allocate (IMGUI is inherently allocation-heavy) but these allocations are editor-only and have no impact on your game's GC behaviour.

**What is the EventBus handler limit and why?**

32 handlers per event. This is a deliberate trade-off: a fixed-size array eliminates the heap allocation that `List<T>` or `delegate.GetInvocationList()` would require on the `Raise` hot path. 32 handlers is sufficient for all internal Audio Atlas panels plus reasonable user code. If you hit the limit, the subscription is dropped and a `LogWarning` is emitted — check the Console.

**How does AudioAtlasRuntime pool work?**

The pool is a `AudioSource[32]` array stored on the `[AudioAtlas_Engine]` GameObject. `PlayOneShotAuto()` picks the next slot in a circular fashion (`_poolNext % 32`). If the slot is still playing, it is stopped first. This means very long clips may be cut short if the pool is exhausted. 32 concurrent one-shots is the practical limit — for games with higher concurrency, route through your own pooling system and use the EventBus to log the plays.

**Does the Session Diff work across different Unity versions?**

Yes. Session JSON is versioned and the diff engine (`SessionDiffEngine.Compute`) operates on `AudioSession` fields that are stable across minor Unity versions. Fields added in future Audio Atlas versions will be absent from older sessions — the diff engine treats absent fields as unchanged.

**Can I access Audio Atlas data from my own editor tools?**

Yes. All public members of `AudioRegistry`, `EventBus`, and `ScanCoordinator` are accessible from any editor assembly that references `AudioAtlas.Runtime` or `AudioAtlas.Editor` respectively. Subscribe to events in `OnEnable`/`OnDisable` in your `EditorWindow` and read `AudioRegistry` properties on repaint.

```csharp
// Example: custom editor panel reading health score
void OnEnable()
{
    EventBus.Subscribe(AudioAtlasEvent.HealthScoreUpdated, _ => Repaint());
}

void OnGUI()
{
    var score = AudioRegistry.HealthScore;
    if (score.IsValid)
        GUILayout.Label($"Audio: {score.Grade} ({score.Score:F0})");
}
```

**How many sources can Audio Atlas track simultaneously?**

The default is 128, configured by `MaxTrackedSources`. The practical ceiling is whatever your hardware can track in `UpdateIntervalFrames` frames without impacting frame rate. In testing on mid-range hardware, 512 sources tracked every 3 frames adds approximately 0.2ms per tick. Sources beyond the limit are untracked but play normally.

---

## Integration

**Can I use the Runtime API without the editor window?**

Yes. If you only need `EventBus`, `AudioRegistry`, and `AudioAtlasRuntime` in a build, reference `AudioAtlas.Runtime`, define `AUDIOATLAS_DEBUG`, and the bootstrap will initialize on its own. You do not need to open the editor window.

**Can I call `ScanCoordinator.RunAsync()` from my own tool?**

Yes. Subscribe to `ScanCoordinator.OnScanComplete` to receive the result:

```csharp
ScanCoordinator.OnScanComplete += result =>
{
    Debug.Log($"My tool: {result.TotalErrors} errors found");
};
ScanCoordinator.RunAsync();
```

Results are also always available in `ScanCoordinator.LastResult` after the scan completes.

**Can I add custom scan rules?**

Not in v1.0.0. The `BugRuleSet.BuildRuleTable()` method returns a fixed array. Custom rules are planned for a future minor version. Contact [tools.studio@zohomail.in](mailto:tools.studio@zohomail.in) to express interest.

---

## Support

**How do I report a bug?**

Use **Window → Audio → Audio Atlas → About → Report a Bug** — this opens a pre-populated email with your Unity version, OS, and Audio Atlas version. Alternatively, email [tools.studio@zohomail.in](mailto:tools.studio@zohomail.in) directly with subject `Bug Report: AudioAtlas`.

**What information should I include in a bug report?**

- Unity version and OS
- Audio Atlas version (`AudioAtlasEdition.Version`)
- Console output (copy the relevant lines, including stack traces)
- Steps to reproduce
- Whether the issue occurs in a fresh project or only your project

---

[← Common Issues](common-issues.md) · [Back to Docs](../../README.md)
