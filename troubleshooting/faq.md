# FAQ

---

## General

**Does Audio Atlas add overhead to shipped games?**

No. `AudioAtlas.Editor` is editor-only and never compiled into a player build. `AudioAtlas.Runtime` is conditionally compiled behind `#if UNITY_EDITOR || AUDIOATLAS_DEBUG`. When `AUDIOATLAS_DEBUG` is undefined, the compiler eliminates all tracking code. The bootstrap becomes a no-op `return` statement. Your shipped binary contains zero Audio Atlas code unless you explicitly opt in.

---

**Do I need to modify my scenes or add components?**

No. Audio Atlas uses `[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]` to boot before any `Awake()` fires. It creates its own root `GameObject` with `DontDestroyOnLoad` and `HideFlags.HideAndDontSave`. Nothing is added to any scene on disk.

---

**Does Audio Atlas support HDRP, URP, and Built-in simultaneously?**

Yes. Audio Atlas has no render pipeline dependencies. It uses IMGUI for its editor window. The `RenderPipelineShaderResolver` utility is only used for the Scene Heatmap overlay material and falls back cleanly on any pipeline.

---

**Which Unity versions are supported?**

Unity 6000.0 (Unity 6) and later. Unity 2021 and 2022 are not supported.

---

**Can I use Audio Atlas with Addressables or Asset Bundles?**

Runtime tracking works regardless of how audio is loaded. The Project Scanner can only audit `AudioClip` assets loaded into `AssetDatabase` at scan time. Load relevant Addressable groups before scanning to include them.

---

## Licensing

**Can I use Audio Atlas in client work?**

Yes. A single license covers use in any project owned or delivered by the licensee, provided the source code is not delivered to the client.

---

**Can multiple developers at my studio use one license?**

A single license covers one individual or one studio (all employees of a registered entity). For multi-seat enterprise licensing, [Contact Support](mailto:tools.studio@zohomail.in).

---

**Can I include the source in a public repository?**

No. The license prohibits including Audio Atlas source in any publicly accessible repository. Add `Assets/AudioAtlas/` to your `.gitignore` for public-facing projects.

---

**Can I modify the source code?**

Yes, for your own internal use. Redistribution of modified versions is not permitted.

---

**Is the source repository included with purchase?**

No. The development repository is private and is not a standard purchase entitlement. Purchasing the asset gives you a Unity package with the compiled tool and source files for integration purposes — it does not grant access to the development repository or its version history. Repository access may be granted separately upon written request — [Contact Support](mailto:tools.studio@zohomail.in).

---

## Technical

**How does voice steal detection work?**

`VoiceStealTracker` polls `AudioRegistry.SourceSnapshots` every frame. It tracks which sources were playing in the previous frame. When a source transitions from `isPlaying = true` to `false` while its clip still has remaining `time`, and a new source began playing with a higher priority in the same frame, a steal event is written to `AudioRegistry.StealRingBuffer` via thread-safe `Interlocked.Increment`.

---

**Does Audio Atlas affect garbage collection?**

The runtime hot path (source tracking, event recording, health score tick) allocates zero managed memory. Pre-allocated arrays, structs, and cached strings eliminate GC pressure. IMGUI panel repaints do allocate, but these are editor-only and have no impact on your game's GC behaviour.

---

**What is the EventBus handler limit and why?**

32 handlers per event. A fixed-size array eliminates the heap allocation that `List<T>` or `delegate.GetInvocationList()` would require on the `Raise` hot path. Exceeding the limit logs a `LogWarning`.

---

**How does the AudioAtlasRuntime pool work?**

The pool is an `AudioSource[32]` array on the `[AudioAtlas_Engine]` GameObject. `PlayOneShotAuto()` picks the next slot in a circular fashion (`_poolNext % 32`). If the slot is still playing, it is stopped first. For games requiring higher concurrency, route through your own pooling system and use the EventBus to log plays.

---

**Can I access Audio Atlas data from my own editor tools?**

Yes. All public members of `AudioRegistry`, `EventBus`, and `ScanCoordinator` are accessible from any assembly referencing `AudioAtlas.Runtime` or `AudioAtlas.Editor` respectively.

```csharp
void OnEnable()  => EventBus.Subscribe(AudioAtlasEvent.HealthScoreUpdated, _ => Repaint());
void OnDisable() => EventBus.Unsubscribe(AudioAtlasEvent.HealthScoreUpdated, _ => Repaint());

void OnGUI()
{
    var score = AudioRegistry.HealthScore;
    if (score.IsValid)
        GUILayout.Label($"Audio: {score.Grade} ({score.Score:F0})");
}
```

---

**How many sources can Audio Atlas track simultaneously?**

The default is 128, configured by `MaxTrackedSources`. In testing on mid-range hardware, 512 sources tracked every 3 frames adds approximately 0.2ms per tick.

---

## Integration

**Can I use the Runtime API without the editor window?**

Yes. Reference `AudioAtlas.Runtime`, define `AUDIOATLAS_DEBUG`, and the bootstrap will initialise on its own.

---

**Can I call `ScanCoordinator.RunAsync()` from my own tool?**

Yes:
```csharp
ScanCoordinator.OnScanComplete += result =>
{
    Debug.Log($"My tool: {result.TotalErrors} errors found");
};
ScanCoordinator.RunAsync();
```

---

**Can I add custom scan rules?**

Not in v1.0.0. Custom rules are planned for a future minor version. [Contact Support](mailto:tools.studio@zohomail.in) to register interest.

---

## Support

**How do I report a bug?**

Use **Window → Audio → Audio Atlas → About → Report a Bug** — this opens a pre-populated email with your Unity version, OS, and Audio Atlas version. Alternatively, [Report a Bug](mailto:tools.studio@zohomail.in?subject=Bug%20Report:%20AudioAtlas) directly.

**What information should I include?**

- Unity version and OS
- Audio Atlas version (`AudioAtlasEdition.Version`)
- Console output including stack traces
- Steps to reproduce
- Whether the issue occurs in a fresh project or only your project

---

[← Troubleshooting](common-issues.md) · [Back to Docs](../README.md)
