---
name: fix-ci
description: Diagnose a PR's failing CI checks, classify each failure (formatting, screenshot goldens, flaky test, infra, real bug), and apply the known remedy for each class. Use when asked why a PR or CI is failing, to fix failing checks or tests on a PR, to get a PR green, or to investigate a red check. Triggers on "why is the PR failing", "CI is failing", "a bunch of tests are failing", "fix the failing checks", "Code Format Check is failing", "the PR is red", "get the PR green", "investigate the failing job".
---

# Fix a red PR

Classify every failing check on a PR and apply the standard remedy per class, instead of
re-diagnosing from scratch. Built for Nextdoor repos (android especially); degrade gracefully
elsewhere by skipping repo-specific remedies.

## 1. Collect the failures
```sh
PR=<number or infer from branch>
gh pr checks "$PR"                             # list red checks
gh run view <run-id> --log-failed | tail -80   # per failing job, get the real error
```
Read enough log to find the *first* real error — skip system-metrics noise, "Picked up
JAVA_TOOL_OPTIONS" lines, and cascade failures downstream of the first compile error.

## 2. Classify each failure and apply its remedy

| Class | Signature | Remedy |
|---|---|---|
| **Formatting** | `Code Format Check` / `check_kotlin_code` failed, ktfmt diff output | Run `make fix_kotlin_code` (or `scripts/ktfmt.sh <touched files>`), commit `chore(<scope>): format`, push. Never mix the format commit with logic changes. |
| **Screenshot goldens** | Roborazzi job red, `images/<name>.png is changed` | First confirm the visual change is *intended* (it follows from this PR's UI change). If intended: `gh pr edit "$PR" --add-label update-roborazzi-tests`, then rerun the failed job (`gh run rerun <id> --failed`) so goldens re-record. If NOT clearly intended, treat as a real bug — never use the label to bury an unexpected UI change. |
| **Flaky / unrelated test** | Failing test touches code this PR didn't change, passes locally, or is a known flake (e.g. `EntityViewerViewModelTest`) | `gh run rerun <id> --failed` once. If it fails again on the rerun, stop calling it flaky — reproduce locally and treat as real. |
| **Infra** | `No space left on device`, runner lost, docker pull errors, `response_code=401` from CI-side credentials (e.g. the PagerDuty oncall-approval check) | Nothing to fix in the PR. Rerun later for transient infra; for CI-credential failures report which team owns it (oncall check → android-client-platform) and move on. |
| **Real failure** | Compile error or test failure in code this PR touched | Reproduce locally on the touched module (`./gradlew :module:testReleaseUnitTest --tests "*TheTest"`), fix it properly (repo conventions + tests per global rules), push. |

Merge-conflict red (`mergeable: CONFLICTING`) isn't a CI class — hand off to the `rebase-pr` skill.

## 3. Verify and report
- After remedies, re-watch: `gh pr checks "$PR" --watch` (or poll every few minutes — don't
  hand-roll tight sleep loops).
- Report per check: classification, remedy applied, current state. Say explicitly if something is
  parked as infra/flaky so the user knows nothing was swept under the rug.

## Guardrails
- One diagnosis per failure — find the first real error, don't chase cascades.
- Local gradle failing with `Failed to obtain CodeArtifact token` / `Unable to load credentials`
  is **local AWS cred expiry**, never a code problem — ask the user to `aws sso login
  --sso-session nextdoor`, or defer local verification to CI and say so.
- Don't rerun a job more than twice on the "flaky" theory.
- Formatting commits stay separate from logic commits.
