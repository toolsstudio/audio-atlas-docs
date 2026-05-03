# Quick Start

From import to a live monitoring session in under five minutes.

---

## 1 — Open the Window

```
Window → Audio → Audio Atlas     Ctrl+Shift+A
```

Dock it to any panel — the window supports Unity's full docking system and remembers its position between sessions. On first open, `AudioAtlasSettings.GetOrCreate()` creates the settings asset with defaults. The **Live Sources** panel is active and empty in Edit Mode.

---

## 2 — Enter Play Mode

Press **▶**. Audio Atlas activates automatically via:

```csharp
[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
```

The bootstrap creates two `DontDestroyOnLoad` GameObjects (`[AudioAtlas_Root]` and `[AudioAtlas_Engine]`) with `HideFlags.HideAndDontSave` — they do not appear in the Hierarchy. `AudioRegistry.Initialize(config)` pre-allocates all tracking buffers before any scene `Awake()` fires, eliminating the "Sources: 0" first-frame artifact.

The status bar switches from **EDIT** to **PLAY** and begins counting live sources, voices, and events.

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

**Filter** using the **Playing / All / Looping** buttons. **Click any row** to select that `AudioSource` and activate the Signal Chain and Source Inspector panels for it.

---

## 4 — Run a Scan

Click **▶ Scan** in the toolbar. The scanner runs asynchronously. When complete, the **Project Scanner** panel populates with findings grouped by severity:

```
● Error   (2)     AA-001  Stereo clip on 3D AudioSource       [Fix]
● Error   (1)     AA-005  Multiple AudioListeners active        —
⚠ Warning (4)     AA-002  No AudioMixerGroup assigned           —
⚠ Warning (2)     AA-008  Zero rolloff range                   [Fix]
ℹ Info    (1)     AA-007  Volume set to 0                      [Fix]
```

Click **Fix** to apply auto-fixable corrections. `Ctrl+Z` reverts with a named undo entry.

---

## 5 — Read the Health Score

Navigate to **Health & Metrics** in the sidebar. The health score updates every ~0.5 seconds in Play Mode:

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

[HIGH]      Co-occurrence: Explosion_01 + RubbleSettle_A
            Co-fired 8 times within 50ms — consider a compound clip

[MEDIUM]    Steal burst: 5 voice steals in 0.4 seconds
            Peak concurrent voices exceeded limit — review priority assignment
```

---

## 7 — Export a Report

Click **⤓ Report** in the toolbar to export the current session as Markdown, JSON, or plain text.

---

## Next Steps

| Found | Read |
|---|---|
| Scan errors | [Scanner Rules](../panels/scanner-rules.md) — root cause and fix for each rule |
| Poor health sub-score | [Intelligence Engine](../core-systems/intelligence-engine.md) — how sub-scores are calculated |
| Want CI integration | [CI Integration](../ci/ci-integration.md) |
| Want to subscribe to events | [EventBus Reference](../core-systems/event-bus.md) |
| Want to call playback APIs | [Runtime API](../api/runtime-api.md) |
| Want to tune settings | [Configuration Reference](../configuration/settings-reference.md) |

---

[← Installation](installation.md) · [Architecture →](../core-systems/architecture.md)
