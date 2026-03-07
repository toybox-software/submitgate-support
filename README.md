# SubmitGate Support

This repository is the public home for **SubmitGate** support:

- 🐛 Bug reports
- 🚨 Crash reports
- ✅ False positives / false negatives
- 💡 Feature requests
- 📚 Documentation issues

> SubmitGate’s source repository is private. This repo is intentionally public so anyone can file issues and share feedback.

---

## Quick Links

- **Report a bug / crash:** open a GitHub Issue
- **Ask a question / share feedback:** open a GitHub Discussion (if enabled)
- **Security issues:** do **not** post publicly — see [Security](#security)

---

## Before you file an issue

### 1) Check duplicates
Search existing issues first — you may find a workaround or existing tracking issue.

### 2) Gather the basics
Please include:

- SubmitGate version: `submitgate --version`
- OS + Node version:
  - OS (macOS / Linux / Windows)
  - `node --version`
- Command you ran (redact secrets)
- What you expected vs what happened
- If it’s a scan issue: which platform/artifact type (iOS `.ipa/.xcarchive/.app` or Android `.apk/.aab`)

---

## Reporting a bug

Open an issue and include:

- **Steps to reproduce**
- **Command + flags**
- **Relevant config/policy/baseline** (if used)
- **Output format**
  - If you ran with `--format json` or `--format sarif`, attach the generated file if you can
  - If the report contains sensitive paths, redact them

### Useful flags to include
- `--format json` (easier for us to reason about)
- `--format sarif` (great for CI + finding fingerprints)
- `--debug` (if available)

---

## Reporting a false positive / false negative

False positives are best reported with **the smallest possible reproduction**.

Please include:

- **Rule ID** (e.g., `IOS_PRIVACY_MANIFEST`, `ANDROID_EXPORTED_COMPONENTS`)
- **Evidence excerpt** (copy/paste the specific section of the report)
- **Why it’s wrong** (what should have happened instead)
- If possible: a redacted JSON/SARIF report showing the finding

If you can’t share artifacts publicly:
- Share a redacted report (remove app identifiers / paths)
- Or contact us privately (see [Security](#security))

---

## Crash reports

If SubmitGate crashes, please include:

- Full stack trace (copy/paste)
- The command you ran (redact secrets)
- Artifact type (iOS/Android + file extension)
- Whether it crashes deterministically or intermittently

If you have a large log, paste the most relevant part and attach the rest as a file or a Gist.

---

## CI / GitHub Actions issues

Include:
- Workflow YAML snippet (the job/steps running SubmitGate)
- Whether you’re uploading SARIF
- `permissions:` block (especially `security-events: write`)
- Any relevant logs from the failed step

---

## License / Portal issues

If you’re having trouble with tokens, portal links, or email delivery:

- Do **not** post your full token publicly.
- You can share:
  - the first/last 6 characters (e.g., `sg_abc123…def456`)
  - the error message you received
  - whether you’re using Preview vs Production portal

---

## Security

Please **do not** open public issues for:
- token disclosure
- vulnerabilities that allow bypassing licensing
- anything that could expose private app artifacts or secrets

Instead, email: **security@YOUR_DOMAIN.com**  
(Replace this with your real address.)

If you don’t have a security inbox yet, use: **support@YOUR_DOMAIN.com** and include “SECURITY” in the subject.

---

## What happens after you file

We’ll usually respond by asking for one of:
- a short repro
- a redacted JSON/SARIF report
- environment details

If we confirm a bug, we’ll label it and link a release/version when it’s fixed.

---

## Privacy

SubmitGate is designed to be offline-first and deterministic.  
We will never ask you to share private artifacts publicly. When reproduction requires an artifact, we’ll suggest ways to redact or share privately.

---

## Maintainers

- Toybox Software LLC

> If you’re an early user and want tighter feedback loops, open an issue and ask about joining the beta.
