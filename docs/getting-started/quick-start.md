# Quick Start

From import to a live monitoring session in under five minutes.

---

## 1 — Open the Window

```
Window → Audio → Audio Atlas     Ctrl+Shift+A
```

The window opens floating. Dock it to any panel — it supports Unity's full docking system and remembers its position between sessions.

The first time it opens, `AudioAtlasSettings.GetOrCreate()` creates `Assets/AudioAtlas/Settings/AudioAtlasSettings.asset` with default values. You will see the **Live Sources** panel, which is empty in Edit Mode.

---

## 2 — Enter Play Mode

Press **▶**. Audio Atlas activates automatically via:

```csharp
[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
```

The bootstrap creates two `DontDestroyOnLoad` GameObjects (`[AudioAtlas_Root]` and `[AudioAtlas_Engine]`) with `HideFlags.HideAndDontSave` — they do not appear in the Hierarchy. `AudioRegistry.Initialize(config)` pre-allocates all tracking buffers before any scene `Awake()` call fires, eliminating the "Sources: 0" first-frame artifact.

The status bar at the bottom of the window switches from **EDIT** to **PLAY** and begins counting live sources, voices, and events.

---

## 3 — Read Live Sources

The Live Sources panel shows every tracked `AudioSource` with live data:

```
State  │ GameObject        │ Clip             │ Vol   │ Pitch │ Blend │ Dist  │ Pri │ Group
───────┼───────────────────┼──────────────────┼───────┼───────┼───────┼───────┼─────┼──────
LOOP   │ ForestAmbience    │ Ambient_180Hz    │ 0.60  │ 1.00  │ 0.00  │ —     │ 128 │ Amb
PLAY   │ SpeakerStand_0    │ Synth_Arp_01     │ 0.80  │ 1.00  │ 1.00  │ 4.2m  │ 100 │ SFX
PLAY   │ SpeakerStand_1    │ Synth_Bass       │ 0.75  │ 0.98  │ 1.00  │ 7.8m  │ 100 │ SFX
```

**Filter** the list using the **Playing / All / Looping** buttons above it. **Click any row** to select that `AudioSource` in the Inspector and activate the Signal Chain and Source Inspector panels for it.

---

## 4 — Run a Scan

Click **▶ Scan** in the toolbar (or use the keyboard shortcut shown in the toolbar tooltip). The scanner runs asynchronously — a progress indicator appears in the toolbar while it works.

When complete, the **Project Scanner** panel populates with findings grouped by severity:

```
● Error   (2)     AA-001  Stereo clip on 3D AudioSource  ← ForestAmbience     [Fix]
● Error   (1)     AA-005  Multiple AudioListeners active                         —
⚠ Warning (4)     AA-002  No AudioMixerGroup assigned    ← AmbientSource_0      —
⚠ Warning (2)     AA-008  Zero rolloff range             ← Column_1            [Fix]
ℹ Info    (1)     AA-007  Volume set to 0                ← SfxTrigger_2        [Fix]
```

**Fix buttons** appear for auto-fixable rules. Click **Fix** on any row to apply the correction. `Ctrl+Z` reverts with a named undo entry.

---

## 5 — Read the Health Score

Navigate to **Health & Metrics** in the sidebar. The health score updates every ~0.5 seconds while in Play Mode:

```
Grade: B    Score: 81.4 / 100    → +2.1 (improving)

Voice    ████████████████████░░  21/25
CPU      ██████████████████████  22/25
Design   ████████████████░░░░░░  17/25
Steal    █████████████░░░░░░░░░  10/15
Mixer    ████████░░░░░░░░░░░░░░   6/10
```

A score above 85 is production-ready. Below 70 indicates systemic issues that will affect player experience.

---

## 6 — Check the Insights Panel

Navigate to **Insights**. The intelligence engine surfaces ranked findings from the current session:

```
[CRITICAL]  Clip spam detected: Footstep_Concrete
            Fired 14× in 1 second — likely missing cooldown guard

[HIGH]      Co-occurrence coupling: Explosion_01 + RubbleSettle_A
            Co-fired 8 times within 50ms — consider grouping into one compound clip

[MEDIUM]    Steal burst: 5 voice steals in 0.4 seconds
            Peak concurrent voices exceeded limit — review priority assignment
```

Each card provides specific data, not generic advice.

---

## 7 — Export a Report

Click **⤓ Report** in the toolbar to export the current session. Choose Markdown, JSON, or plain text. The JSON export is also consumed by `CIBridge.Run()` in headless mode.

The report captures: health score, all scan findings, source summary, event log excerpt, insight cards, and session metadata.

---

## Next Steps

You now have a complete picture of your project's audio health. Where to go from here depends on what you found:

| Found | Read |
|---|---|
| Scan errors | [Scanner Rules Reference](../panels/scanner-rules.md) — root cause and fix for each rule |
| Poor health sub-score | [Intelligence Engine](../core-systems/intelligence-engine.md) — how each sub-score is calculated |
| Want CI integration | [CI Integration](../ci/ci-integration.md) — automated scanning in your pipeline |
| Want to subscribe to events in code | [EventBus Reference](../core-systems/event-bus.md) |
| Want to call playback APIs | [Runtime API](../api/runtime-api.md) |
| Want to tune settings | [Configuration Reference](../configuration/settings-reference.md) |

---

[← Installation](installation.md) · [Architecture →](../core-systems/architecture.md)
