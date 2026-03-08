# GitHub Actions CI Integration

This guide shows how to run **submitgate** in GitHub Actions and optionally upload
SARIF results to **GitHub Code Scanning**.

Because mobile build pipelines vary widely (native iOS/Android, Flutter,
React Native, custom CI layouts), this guide uses placeholders you must replace.

---

# Support

report issues at [submitgate-support](https://github.com/toybox-software/submitgate-support/issues)

## Placeholders

Replace the following values in all examples:

- `<VERSION>` — the submitgate version to pin (e.g. `0.34.0`)
- `<PLATFORM>` — `ios` or `android`
- `<ARTIFACT_PATH>` — path to the built app artifact in your CI workspace
  - Android: `.apk` or `.aab`
  - iOS: `.ipa`, `.xcarchive`, or `.app`

How you produce the artifact is intentionally **out of scope** for this guide.

---

## Prerequisites

Add `SUBMITGATE_LICENSE` as a GitHub Actions secret in your repo or org settings.

---

## Run a scan and fail CI on findings

This workflow runs `submitgate` and fails the job if findings meet the
`--fail-on` severity threshold.

```yaml
name: submitgate

on:
  pull_request:
  push:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4

      # TODO:
      # Build or download your app artifact here.
      # Ensure it exists at <ARTIFACT_PATH> before scanning.

      - name: Run submitgate (JSON + Markdown)
        env:
          SUBMITGATE_LICENSE: ${{ secrets.SUBMITGATE_LICENSE }}
        run: |
          npx -y submitgate@<VERSION> scan <PLATFORM> <ARTIFACT_PATH> \
            --format both \
            --fail-on BLOCKER
```

**Notes:**

- `--format both` produces `report.json` and `report.md`
- Exit code 0 = success, 1 = findings at or above `--fail-on` threshold, 6 = license error
- No network calls occur during the scan

---

## Use a policy file (recommended for CI stability)

Policy files let you disable noisy rules, override severities, or set per-rule fail thresholds
without changing your CI command.

**Quick start:** Run `submitgate init` locally to scaffold a starter policy file:

```bash
npx submitgate init                       # creates submitgate.policy.json
npx submitgate init --ci github           # also creates .github/workflows/submitgate.yml
npx submitgate init --ci github --baseline  # also creates submitgate.baseline.md
```

Commit the generated files to your repository. Use `--force` to overwrite existing files.

Example policy (demote pre-existing findings when using baseline):

```yaml
- name: Write submitgate policy
  run: |
    cat > submitgate.policy.json << 'JSON'
    {
      "schemaVersion": 1,
      "rules": [
        { "ruleId": "IOS_PRIVACY_MANIFEST", "when": { "baselineState": "unchanged" }, "severity": "INFO" },
        { "ruleId": "IOS_PRIVACY_MANIFEST", "when": { "baselineState": "new" }, "severity": "BLOCKER" }
      ]
    }
    JSON

- name: Run submitgate (JSON + Markdown)
  env:
    SUBMITGATE_LICENSE: ${{ secrets.SUBMITGATE_LICENSE }}
  run: |
    npx -y submitgate@<VERSION> scan <PLATFORM> <ARTIFACT_PATH> \
      --format both \
      --fail-on BLOCKER \
      --policy ./submitgate.policy.json
```

Example policy with `pathMatches` (match findings by evidence path):

```yaml
- name: Write submitgate policy with pathMatches
  run: |
    cat > submitgate.policy.json << 'JSON'
    {
      "schemaVersion": 1,
      "rules": [
        { "ruleId": "IOS_PRIVACY_MANIFEST", "when": { "pathMatches": ["**/*.xcprivacy"] }, "severity": "BLOCKER" },
        { "ruleId": "ANDROID_DEBUGGABLE_BUILD", "when": { "pathMatches": ["AndroidManifest.xml"] }, "severity": "WARNING" }
      ]
    }
    JSON
```

**Policy exit codes:**

- Exit 2: policy file not found / unreadable / invalid JSON
- Exit 4: policy schema or domain error (unknown keys, invalid rule IDs)

---

## Upload SARIF to GitHub Code Scanning

When using `--format sarif`, submitgate writes SARIF to stdout only.
Redirect the output to a file and upload it in a separate step.

**Requirements:**

- Workflow permission: `security-events: write`
- SARIF upload step must run even if findings exist

```yaml
name: submitgate (SARIF)

on:
  pull_request:
  push:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4

      # TODO:
      # Build or download your app artifact here.
      # Ensure it exists at <ARTIFACT_PATH> before scanning.

      - name: Run SubmitGate (SARIF)
        id: submitgate
        continue-on-error: true
        env:
          SUBMITGATE_LICENSE: ${{ secrets.SUBMITGATE_LICENSE }}
        run: |
          npx -y submitgate@<VERSION> scan <PLATFORM> <ARTIFACT_PATH> \
            --format sarif \
            --ci --ci-provider github \
            > submitgate.sarif

      - name: Upload SARIF to GitHub Code Scanning
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: submitgate.sarif

      - name: Fail job if submitgate failed
        if: steps.submitgate.outcome != 'success'
        run: exit 1
```

**Notes:**

- If policy validation fails, the scan exits 2 or 4 (and SARIF won't be produced).
- The `--policy` flag is optional; omit it if you don't need rule customization.

## Platform notes

### Android

- Use `ubuntu-latest`
- Supported artifacts: `.apk`, `.aab`

Network Security Config analysis (including cleartext checks) runs automatically when
scanning APK files. All rules run by default; to fail the build on specific findings,
configure `failOn` severity thresholds or per-rule severity overrides in your policy file.

### iOS

- Use `macos-latest` if your workflow builds the artifact
- If scanning a prebuilt artifact from another job, `ubuntu-latest` may suffice
- Supported artifacts: `.ipa`, `.xcarchive`, `.app`

---

## Aggregate (multi-artifact CI)

For monorepos or pipelines that scan multiple artifacts, use `submitgate aggregate` to
combine individual scan reports into a single SARIF, Markdown, or JSON report:

```yaml
- name: Scan iOS
  env:
    SUBMITGATE_LICENSE: ${{ secrets.SUBMITGATE_LICENSE }}
  run: |
    npx -y submitgate@<VERSION> scan ios path/to/MyApp.xcarchive \
      --format json --out-dir scan-results/ios --ci

- name: Scan Android
  env:
    SUBMITGATE_LICENSE: ${{ secrets.SUBMITGATE_LICENSE }}
  run: |
    npx -y submitgate@<VERSION> scan android path/to/MyApp.apk \
      --format json --out-dir scan-results/android --ci

- name: Aggregate
  id: aggregate
  continue-on-error: true
  env:
    SUBMITGATE_LICENSE: ${{ secrets.SUBMITGATE_LICENSE }}
  run: |
    npx -y submitgate@<VERSION> aggregate scan-results/ios scan-results/android \
      --format sarif --out-dir scan-results --fail-on BLOCKER

- name: Upload SARIF
  if: always()
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: scan-results/aggregate-report.sarif

- name: Fail if issues found
  if: steps.aggregate.outcome == 'failure'
  run: exit 1
```

With `--out-dir`, SARIF is written directly to a file (no stdout redirection needed).

---

## Using Baselines in GitHub Actions

Baselines allow CI to fail only on **new** findings (regressions) rather than
all findings. This is useful when adopting submitgate on a codebase with existing
issues that cannot be fixed immediately.

### One-time baseline creation

Baseline creation is intended to be run **manually or once**, not on every CI run.

```yaml
- name: Create baseline (one-time or manual)
  env:
    SUBMITGATE_LICENSE: ${{ secrets.SUBMITGATE_LICENSE }}
  run: |
    npx -y submitgate@<VERSION> baseline create <PLATFORM> <ARTIFACT_PATH> \
      --out baseline.json
```

**Important:**

- This command **always exits 0** on success, regardless of how many findings exist
- It should **not** be run on every CI build
- The generated `baseline.json` should be committed to the repository

### CI enforcement using an existing baseline

Once a baseline is committed, use it in CI to fail only on regressions:

```yaml
- name: Scan using baseline (fail on regressions only)
  env:
    SUBMITGATE_LICENSE: ${{ secrets.SUBMITGATE_LICENSE }}
  run: |
    npx -y submitgate@<VERSION> scan <PLATFORM> <ARTIFACT_PATH> \
      --baseline baseline.json \
      --fail-on BLOCKER
```

**Behavior:**

- CI fails **only if new findings** meet or exceed the `--fail-on` threshold
- Existing findings present in the baseline do **not** fail CI
- Exit codes in baseline mode:
  - `0`: No new findings at threshold
  - `1`: New findings at threshold
  - `2`: CLI / IO error (baseline file not found, unreadable, invalid JSON)
  - `4`: Policy domain error (schema validation failure, if `--policy` is used)
  - `5`: Invalid baseline file (schema error, platform mismatch, invalid fingerprint format)
  - `6`: License validation error (missing, invalid, or expired token)

### Notes

- Do not regenerate the baseline on every CI run
- Baseline files are deterministic and should be version-controlled
- A passing CI run may still report findings if they already exist in the baseline
- When `--baseline` is used, SARIF output includes only new findings

### Show all findings with baseline annotations

Use `--baseline-mode all` to see all findings while still gating CI on regressions:

```yaml
- name: Scan with full visibility
  env:
    SUBMITGATE_LICENSE: ${{ secrets.SUBMITGATE_LICENSE }}
  run: |
    npx -y submitgate@<VERSION> scan <PLATFORM> <ARTIFACT_PATH> \
      --baseline baseline.json \
      --baseline-mode all \
      --format sarif > submitgate.sarif
```

The SARIF output includes `properties.baselineState` for each finding (`"new"` or `"unchanged"`),
allowing GitHub Code Scanning to display the full picture while CI still gates on new issues only.

---

## Debug: Policy Evaluation Explanation

Use `--explain-policy` to debug policy rule evaluation in CI:

```yaml
- name: Run submitgate with policy explanation
  env:
    SUBMITGATE_LICENSE: ${{ secrets.SUBMITGATE_LICENSE }}
  run: |
    npx -y submitgate@<VERSION> scan <PLATFORM> <ARTIFACT_PATH> \
      --format sarif \
      --policy ./submitgate.policy.json \
      --explain-policy \
      > submitgate.sarif 2> policy-explain.log
```

**Important:**

- Explanation output goes to **stderr** (redirected to `policy-explain.log` above)
- SARIF output goes to **stdout** (redirected to `submitgate.sarif`)
- Requires a policy file; without one, prints a one-line note to stderr
- Useful for debugging why rules matched or didn't match specific findings

---

## Design guarantees

- SARIF output is deterministic (no timestamps or UUIDs)
- Coverage gaps are emitted as SARIF tool execution notifications
- Scans do not mutate the workspace except for explicit output files
- CI behavior is stable across runs for identical inputs
