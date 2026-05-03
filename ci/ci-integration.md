# CI Integration

Audio Atlas can run headless project scans from any CI/CD pipeline. The entry point is `CIBridge.Run()`, invoked via Unity's `-executeMethod` flag in batch mode.

---

## Entry Point

```
AudioAtlas.Editor.Automation.CIBridge.Run
```

This method:
1. Calls `ScanCoordinator.RunBlocking()` — pumps the Unity editor loop without `Thread.Sleep`
2. Writes a JSON health report to disk
3. Appends a `BuildRecord` to `BuildHistory`
4. Exits Unity with a meaningful code

---

## Command

**macOS / Linux:**
```bash
/Applications/Unity/Hub/Editor/6000.0.0f1/Unity.app/Contents/MacOS/Unity \
  -batchmode -quit \
  -projectPath "$WORKSPACE" \
  -executeMethod AudioAtlas.Editor.Automation.CIBridge.Run \
  -logFile "$WORKSPACE/logs/audio_atlas.log"
```

**Windows:**
```powershell
"C:\Program Files\Unity\Hub\Editor\6000.0.0f1\Editor\Unity.exe" `
  -batchmode -quit `
  -projectPath "C:\Projects\MyGame" `
  -executeMethod AudioAtlas.Editor.Automation.CIBridge.Run `
  -logFile "logs\audio_atlas.log"
```

---

## Exit Codes

| Code | Meaning | Recommended Action |
|---|---|---|
| `0` | Clean — zero errors | ✅ Pass |
| `1` | Issues found (`TotalErrors > 0`) | ❌ Fail or ⚠ Warn |
| `2` | Scan failed, cancelled, or timed out | ❌ Fail — investigate |

> During initial integration, treat exit code `1` as a warning. Fix issues incrementally, then promote to a hard failure once the baseline is clean.

---

## Report Output

**Default path:** `AudioAtlasReports/AudioAtlasHealthReport.json`  
**Override:** Set `AUDIOATLAS_REPORT_PATH` environment variable to a full file path.

### JSON Schema

```json
{
  "HealthGrade":    "B",
  "HealthScore":    81.4,
  "TotalErrors":    1,
  "TotalWarnings":  3,
  "TotalInfos":     2,
  "ScanDurationMs": 1240,
  "Timestamp":      "2025-06-01T14:32:00Z",
  "UnityVersion":   "6000.0.0f1",
  "Records": [
    {
      "RuleID":      "AA-001",
      "Severity":    "Error",
      "Summary":     "Stereo clip on 3D AudioSource",
      "Message":     "Stereo clip 'Ambient_Forest.wav' on 'ForestAmbience' (Spatial Blend: 1.0)",
      "AssetPath":   "Assets/Audio/Ambient/Ambient_Forest.wav",
      "ScenePath":   "Assets/Scenes/Forest.unity",
      "AutoFixable": true
    }
  ]
}
```

---

## Suppressing Rules

```bash
export AUDIOATLAS_SUPPRESS_RULES="AA-010,AA-011"
```

Suppressed rules are excluded from `TotalErrors`/`TotalWarnings` but appear in `Records` with `"Suppressed": true`.

---

## GitHub Actions

```yaml
name: Audio Quality Gate

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  audio-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/cache@v4
        with:
          path: Library
          key: Library-${{ hashFiles('Assets/**', 'Packages/**', 'ProjectSettings/**') }}
          restore-keys: Library-

      - uses: game-ci/unity-builder@v4
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
          UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}
          AUDIOATLAS_SUPPRESS_RULES: "AA-010"
        with:
          unityVersion: 6000.0.0f1
          buildMethod: AudioAtlas.Editor.Automation.CIBridge.Run
          allowDirtyBuild: true

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: audio-atlas-report-${{ github.sha }}
          path: AudioAtlasReports/AudioAtlasHealthReport.json
          retention-days: 30

      - name: Post Health Score to Summary
        if: always()
        run: |
          REPORT="AudioAtlasReports/AudioAtlasHealthReport.json"
          if [ -f "$REPORT" ]; then
            GRADE=$(jq -r '.HealthGrade' $REPORT)
            SCORE=$(jq '.HealthScore' $REPORT)
            ERRORS=$(jq '.TotalErrors' $REPORT)
            WARNINGS=$(jq '.TotalWarnings' $REPORT)
            echo "## Audio Atlas" >> $GITHUB_STEP_SUMMARY
            echo "| Grade | Score | Errors | Warnings |" >> $GITHUB_STEP_SUMMARY
            echo "|---|---|---|---|" >> $GITHUB_STEP_SUMMARY
            echo "| **$GRADE** | $SCORE/100 | $ERRORS | $WARNINGS |" >> $GITHUB_STEP_SUMMARY
          fi
```

---

## Jenkins

```groovy
pipeline {
    agent any
    environment {
        UNITY_PATH = '/Applications/Unity/Hub/Editor/6000.0.0f1/Unity.app/Contents/MacOS/Unity'
        AUDIOATLAS_REPORT_PATH = "${WORKSPACE}/reports/audio_atlas_${BUILD_NUMBER}.json"
    }
    stages {
        stage('Checkout') { steps { checkout scm } }
        stage('Audio Quality Gate') {
            steps {
                sh '''
                    $UNITY_PATH -batchmode -quit                       -projectPath $WORKSPACE                       -executeMethod AudioAtlas.Editor.Automation.CIBridge.Run                       -logFile $WORKSPACE/logs/audio_atlas.log
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/*.json', allowEmptyArchive: true
                    archiveArtifacts artifacts: 'logs/audio_atlas.log', allowEmptyArchive: true
                }
            }
        }
    }
    post {
        failure {
            emailext(
                subject: "Audio Quality Gate Failed — Build #${BUILD_NUMBER}",
                body: "Scan found errors. Report: ${env.AUDIOATLAS_REPORT_PATH}",
                to: 'team@studio.com'
            )
        }
    }
}
```

---

## Unity Cloud Build

UCB does not support arbitrary `-executeMethod` via its standard pipeline. Use a **Pre-Export Script** or configure a separate headless job via the Unity Editor CLI after the UCB build completes.

---

## Build History Integration

`CIBridge.Run()` automatically appends to `BuildHistory`. The **Build History** panel shows health grade trends across recorded builds as a sparkline chart. Pass your CI build number via environment variable and read it in a custom `CIBridge` subclass if you need external build number correlation.

---

[← Runtime API](../api/runtime-api.md) · [Troubleshooting →](../troubleshooting/common-issues.md)
