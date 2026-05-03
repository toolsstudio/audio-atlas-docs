# Scanner Rules Reference

The Project Scanner evaluates every `AudioSource`, `AudioClip`, `AudioMixer`, and `AudioListener` in all open scenes against 12 rules. Rules run via `ScanCoordinator.RunAsync()` → `ArchitectureValidator` → `BugRuleSet.BuildRuleTable()`. Each rule is a self-contained static method — no shared state, no cross-rule dependency.

---

## Running a Scan

**Interactive:** Click **▶ Scan** in the toolbar.

**Headless (CI):**
```bash
Unity -batchmode -projectPath . -executeMethod AudioAtlas.Editor.Automation.CIBridge.Run
```

Exit codes: `0` = clean · `1` = issues found · `2` = scan failed/timeout.

---

## Severity Levels

| Level | Meaning |
|---|---|
| **Error** | Will cause an audio failure or incorrect output at runtime |
| **Warning** | Suboptimal — degrades quality, memory, or performance |
| **Info** | Best-practice recommendation — may be intentional |

---

## Rule Reference

### AA-001 — Stereo Clip on 3D AudioSource
**Severity:** Error  
**Checks:** `AudioClip.channels > 1` on a source with `spatialBlend > 0`

Stereo clips ignore Unity's spatial audio pipeline. Both channels play identically at all positions, nullifying 3D positioning, rolloff, and HRTF processing.

**Auto-fix:** Sets `AudioImporter.forceToMono = true` and reimports via `AssetImporter.SaveAndReimport()`.

---

### AA-002 — No AudioMixerGroup Assigned
**Severity:** Warning  
**Checks:** `AudioSource.outputAudioMixerGroup == null`

Sources without a mixer group route directly to the master output, bypassing all mixing, ducking, effects, and volume automation.

**Auto-fix:** None — the correct group depends on the source's semantic role.

---

### AA-003 — Play On Awake + Loop
**Severity:** Warning  
**Checks:** `AudioSource.playOnAwake && AudioSource.loop`

A looping source that plays on awake restarts every time the scene is loaded or the `GameObject` is reactivated, accumulating duplicate instances in additive loading scenarios.

**Auto-fix:** Sets `AudioSource.playOnAwake = false` via `Undo.RecordObject`. Loop setting preserved.

---

### AA-004 — Unsuitable Load Type for Frequent SFX
**Severity:** Warning  
**Checks:** `AudioClip.loadType == AudioClipLoadType.Streaming` on short clips

Streaming incurs disk I/O on every play event and introduces latency. `DecompressOnLoad` is the correct setting for short, frequently-triggered sounds.

**Auto-fix:** None — the threshold is context-dependent.

---

### AA-005 — Multiple AudioListeners Active
**Severity:** Error  
**Checks:** Count of active `AudioListener` components in loaded scenes > 1

Unity produces undefined spatial audio output when multiple `AudioListener` components are active simultaneously.

**Auto-fix:** None — disabling the wrong listener would break spatial audio entirely.

---

### AA-006 — Duplicate Priority Values
**Severity:** Warning  
**Checks:** Two or more `AudioSource` components share the same `priority` value

Identical priorities make voice stealing non-deterministic across platforms and runtime states.

**Auto-fix:** None — priority ordering requires semantic knowledge.

---

### AA-007 — Volume Set to Zero
**Severity:** Info  
**Checks:** `AudioSource.volume == 0`

A source with zero volume still occupies a voice slot, participates in rolloff calculations, and fires play/stop events.

**Auto-fix:** Restores `AudioSource.volume` to `1f` via `Undo.RecordObject`.

---

### AA-008 — Min Distance Equals Max Distance
**Severity:** Warning  
**Checks:** `AudioSource.minDistance >= AudioSource.maxDistance` on a 3D source

When min and max distance are equal, the rolloff curve has no range. The source plays at the same volume regardless of listener distance, disabling 3D attenuation entirely.

**Auto-fix:** Sets `maxDistance = minDistance * 10f` via `Undo.RecordObject`.

---

### AA-009 — Uncompressed Clip in Memory
**Severity:** Warning  
**Checks:** `AudioClip.loadType == DecompressOnLoad` on clips with `compressionFormat == PCM`

PCM clips at full resolution consume 5–8× more RAM than Vorbis-compressed equivalents with no perceptible quality difference at q0.7.

**Auto-fix:** Sets Vorbis compression at quality `0.7f` and reimports via `AssetImporter.SaveAndReimport()`.

---

### AA-010 — Extreme Pitch Value
**Severity:** Info  
**Checks:** `AudioSource.pitch < 0.1f || AudioSource.pitch > 3f`

Pitch values outside 0.1–3.0 produce audible artefacts on most audio hardware.

**Auto-fix:** None — extreme pitch may be intentional.

---

### AA-011 — Looping Source with No Mixer Group
**Severity:** Warning  
**Checks:** `AudioSource.loop && AudioSource.outputAudioMixerGroup == null`

Looping sources without mixer group assignment cannot be muted, ducked, or affected by runtime mixer automation.

**Auto-fix:** None — requires knowledge of the correct target group.

---

### AA-012 — Default Priority on a Loud Source
**Severity:** Warning  
**Checks:** `AudioSource.priority == 128` on sources with high volume or short distance

Priority 128 (Unity's default) on a critical source risks steal by a lower-importance source.

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

All fixes use `Undo.SetCurrentGroupName("AudioAtlas Fix — {ruleID}")`. `Ctrl+Z` shows the exact rule ID in Undo history.

---

## Suppress a Rule in CI

```bash
export AUDIOATLAS_SUPPRESS_RULES="AA-010,AA-011"
```

Suppressed rules are excluded from the finding count but appear in the JSON report with `"suppressed": true`.

---

[← Panels Reference](panels-reference.md) · [Back to Docs](../README.md)
