# Scanner Rules Reference

The Project Scanner evaluates every `AudioSource`, `AudioClip`, `AudioMixer`, and `AudioListener` in all open scenes against 12 rules. Rules run via `ScanCoordinator.RunAsync()` → `ArchitectureValidator` → `BugRuleSet.BuildRuleTable()`.

Each rule is a self-contained static method — no shared state, no cross-rule dependency.

---

## Running a Scan

**Interactive:** Click **▶ Scan** in the toolbar. Results appear in the **Project Scanner** panel.

**Headless (CI):**
```bash
Unity -batchmode -projectPath . -executeMethod AudioAtlas.Editor.Automation.CIBridge.Run
```

Exit codes: `0` = clean, `1` = issues found, `2` = scan failed/timeout.

---

## Severity Levels

| Level | Meaning |
|---|---|
| **Error** | Will cause an audio failure or produce incorrect output at runtime |
| **Warning** | Suboptimal — degrades quality, memory, or performance |
| **Info** | Best-practice recommendation — may be intentional |

---

## Rule Reference

### AA-001 — Stereo Clip on 3D AudioSource
**Severity:** Error  
**Checks:** `AudioClip.channels > 1` on a source with `spatialBlend > 0`

Stereo clips ignore Unity's spatial audio pipeline. Both channels play identically at all positions, nullifying any 3D positioning, rolloff, or HRTF processing. The listener hears the sound as flat and directionless regardless of distance or position.

**Auto-fix:** Sets `AudioImporter.forceToMono = true` and reimports the clip via `AssetImporter.SaveAndReimport()`. The source's `spatialBlend` is unchanged.

**When to suppress:** If the source's `spatialBlend` is intentionally 0 (pure 2D) and the rule is a false positive because the blend setting wasn't captured by the scan context, set `spatialBlend` explicitly to 0 to suppress.

---

### AA-002 — No AudioMixerGroup Assigned
**Severity:** Warning  
**Checks:** `AudioSource.outputAudioMixerGroup == null`

Sources without a mixer group route directly to the master output, bypassing all mixing, ducking, effects, and volume automation. This makes runtime audio control impossible and prevents bus compression from working correctly.

**Auto-fix:** None — the correct group depends on the source's semantic role.

---

### AA-003 — Play On Awake + Loop
**Severity:** Warning  
**Checks:** `AudioSource.playOnAwake && AudioSource.loop`

A looping source that plays on awake will restart every time the scene is loaded or the `GameObject` is reactivated. In additive scene loading or scene restart scenarios, duplicate instances accumulate — each consuming a voice slot and contributing to mixer output.

**Auto-fix:** Sets `AudioSource.playOnAwake = false` via `Undo.RecordObject`. The loop setting is preserved.

---

### AA-004 — Unsuitable Load Type for Frequent SFX
**Severity:** Warning  
**Checks:** `AudioClip.loadType == AudioClipLoadType.Streaming` on clips shorter than a threshold

Streaming load type is designed for long audio (music, ambient tracks). For short clips played frequently, streaming incurs disk I/O on every play event and introduces measurable latency. `DecompressOnLoad` is the correct setting for short, frequently-triggered sounds.

**Auto-fix:** None — the threshold is context-dependent. Inspect the clip's intended use before changing.

---

### AA-005 — Multiple AudioListeners Active
**Severity:** Error  
**Checks:** Count of active `AudioListener` components in loaded scenes > 1

Unity produces undefined spatial audio output when multiple `AudioListener` components are active simultaneously. Distance calculations become ambiguous. The audio engine cannot determine a reference position.

**Auto-fix:** None — disabling the wrong listener (e.g., the main camera listener when there is a VR listener) would break spatial audio entirely.

---

### AA-006 — Duplicate Priority Values
**Severity:** Warning  
**Checks:** Two or more `AudioSource` components share the same `priority` value

When two sources compete for the last available voice slot and have identical priorities, Unity's voice stealing decision is non-deterministic across platforms and runtime states. Explicit priority separation ensures predictable voice allocation.

**Auto-fix:** None — priority ordering requires understanding of the source's semantic role.

---

### AA-007 — Volume Set to Zero
**Severity:** Info  
**Checks:** `AudioSource.volume == 0`

A source with zero volume still occupies a voice slot, participates in distance and rolloff calculations, and fires play/stop events. If the intent is to silence a source, it should be disabled or have its priority reduced to allow higher-priority sources to claim the voice.

**Auto-fix:** Restores `AudioSource.volume` to `1f` via `Undo.RecordObject`. Severity is Info — this may be intentional (e.g., faded out via code).

---

### AA-008 — Min Distance Equals Max Distance
**Severity:** Warning  
**Checks:** `AudioSource.minDistance >= AudioSource.maxDistance` on a 3D source

When min and max distance are equal (or max < min), the rolloff curve has no range to operate over. The source plays at the same volume regardless of listener distance. This effectively disables 3D attenuation.

**Auto-fix:** Sets `maxDistance = minDistance * 10f` via `Undo.RecordObject`. This is a reasonable starting point — tune to match the source's spatial intent.

---

### AA-009 — Uncompressed Clip in Memory
**Severity:** Warning  
**Checks:** `AudioClip.loadType == DecompressOnLoad` on clips with `compressionFormat == PCM`

PCM clips decompressed into memory at full resolution consume significantly more RAM than their compressed counterparts. For most game audio, Vorbis at quality 0.7 is perceptually indistinguishable and 5–8× smaller in memory.

**Auto-fix:** Sets `TextureImporterType` to Vorbis at quality `0.7f` and reimports via `AssetImporter.SaveAndReimport()`. Use the simulation mode to preview which clips would be affected before applying.

---

### AA-010 — Extreme Pitch Value
**Severity:** Info  
**Checks:** `AudioSource.pitch < 0.1f || AudioSource.pitch > 3f`

Pitch values outside the range 0.1–3.0 produce audible artefacts on most audio hardware. Very low values create sub-bass rumble with undefined frequency content. Very high values cause aliasing and pitch-tracking inaccuracies. This rule surfaces the value for manual review.

**Auto-fix:** None — extreme pitch may be intentional (e.g., a horror ambience effect).

---

### AA-011 — Looping Source with No Mixer Group
**Severity:** Warning  
**Checks:** `AudioSource.loop && AudioSource.outputAudioMixerGroup == null`

Looping sources without mixer group assignment cannot be muted, ducked, or affected by any runtime mixer automation. They are uncontrolled background audio that cannot be managed by the game's audio systems at runtime.

**Auto-fix:** None — requires knowledge of the correct target group.

---

### AA-012 — Default Priority on a Loud Source
**Severity:** Warning  
**Checks:** `AudioSource.priority == 128` (Unity's default) on sources with high volume or short distance

Priority 128 is Unity's default — it means no explicit priority decision was made. On a loud, close-range, or critical source (explosion, UI feedback, dialogue) this risks the source being stolen by a lower-importance source that happened to receive a lower priority number.

**Auto-fix:** None — priority assignment requires semantic context.

---

## Auto-Fix Summary

| Rule | Fix | Undo Registered |
|---|---|---|
| AA-001 | Force To Mono on AudioImporter | ✅ |
| AA-003 | Disable Play On Awake | ✅ |
| AA-007 | Restore volume to 1.0 | ✅ |
| AA-008 | Set maxDistance = minDistance × 10 | ✅ |
| AA-009 | Apply Vorbis q0.7 compression | ✅ |
| AA-002, 004, 005, 006, 010, 011, 012 | Manual — no auto-fix | — |

All fixes use `Undo.SetCurrentGroupName("AudioAtlas Fix — {ruleID}")` and `Undo.CollapseUndoOperations`. `Ctrl+Z` shows the exact rule ID in the Undo history.

---

## Suppress a Rule in CI

```bash
export AUDIOATLAS_SUPPRESS_RULES="AA-010,AA-011"
```

Suppressed rules are excluded from the finding count but appear in the JSON report with `"suppressed": true`.

---

[← Panels Reference](panels-reference.md) · [Back to Docs](../../README.md)
