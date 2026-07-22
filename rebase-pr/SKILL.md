---
name: rebase-pr
description: Rebase a GitHub PR (or a whole PR stack) onto latest main, resolve conflicts following repo conventions, verify locally, and force-push safely. Use when asked to rebase a PR with/onto main, update a PR with main, bring a PR up to date, resolve merge conflicts on a PR, or restack a stack of PRs. Triggers on "rebase the PR", "rebase #<n> with main", "rebase <PR link> with main", "can you rebase with main", "resolve the conflict on <PR>", "update the PR with main", "restack the PRs", "rebase all the PRs".
---

# Rebase a PR onto main

Bring a PR's branch up to date with `main` (or its stack base), resolve conflicts properly, verify
the result compiles and is formatted, and force-push. Works in any repo via `gh`.

**Force-push policy:** `git push --force-with-lease` is part of this skill and pre-approved — do
NOT pause to ask before it. Only stop if the lease fails after a re-fetch (someone else pushed to
the branch) — that needs the user. Never use bare `--force`. Do not create backup branches; the
reflog plus `--force-with-lease` is the safety net.

## Procedure

### 0. Preconditions
- `gh auth status` succeeds.
- Working tree clean (`git status --porcelain` empty). If not, stop and ask — never stash or
  discard the user's work silently.

### 1. Resolve PR, branch, and base
```sh
PR=<arg number, or strip the URL to its number, or infer: gh pr view --json number -q .number>
gh pr view "$PR" --json baseRefName,headRefName,isDraft,mergeable -q '{base:.baseRefName, head:.headRefName, mergeable:.mergeable}'
git fetch origin
gh pr checkout "$PR"
```
- **Base is `main`** → plain rebase (step 2).
- **Base is another PR branch** → this is a stacked PR: rebase the *bottom* of the stack onto main
  first, then each child onto its updated parent, bottom-up. If the repo uses git-spice (`gs ls`
  works), prefer `gs stack restack` + `gs stack submit` over manual per-branch rebases.
- Nextdoor/android note: CI auto-restacks children **when a parent merges** (`restack-prs.yml`).
  This skill is for the manual case — a PR drifting behind main while still open.

### 2. Rebase
```sh
git rebase origin/main        # or origin/<base> for a stack child
```

### 3. Resolve conflicts — conventions first
- Read both sides fully before resolving; never keep "ours" or "theirs" blindly.
- **Load the repo's conventions**: check `CLAUDE.md`/`AGENTS.md` and relevant skills for the
  conflicted files (e.g. in Nextdoor/android, `viewmodels` for ViewModel files, `ui` for Compose).
- A conflict that is really a semantic collision (both sides changed the same behavior, an API the
  PR uses was removed/renamed) is a **decision** — reconcile it as the PR author would, and if
  that's ambiguous, stop and ask rather than guess.
- After resolving, search the result for leftover markers: `git diff --check` and grep `<<<<<<<`.

### 4. Verify before pushing
- **Compile the touched modules** (don't build the world). Nextdoor/android:
  `./gradlew -q :libraries:foo:compileReleaseKotlin :features:bar:impl:compileReleaseKotlin` for
  each module the PR touches. In KMP library modules (`blocks-res` etc.) use
  `assembleAndroidMain` / `compileKotlinMetadata` — `assembleDebug` doesn't exist there.
- If the rebase touched test files or the conflict resolution changed logic, run the affected
  module's unit tests (`:module:testReleaseUnitTest`).
- **Format check the touched Kotlin files** so CI's Code Format Check can't fail:
  `scripts/ktfmt.sh --dry-run --set-exit-if-changed <files>` — if it flags files, run
  `scripts/ktfmt.sh <files>` and amend.
- If the environment can't build (e.g. `Failed to obtain CodeArtifact token` — expired AWS creds),
  say so, ask the user to run `aws sso login --sso-session nextdoor` if they're around; if not,
  push anyway and state clearly that verification is deferred to CI.

### 5. Push
```sh
git push --force-with-lease
```
Lease failure → `git fetch` and inspect: if the remote branch moved (new commits by someone/CI),
stop and show the user; do not overwrite.

### 6. Post-push
- Confirm mergeable: `gh pr view "$PR" --json mergeable,mergeStateStatus` (poll briefly; GitHub
  recomputes async).
- Kick off a checks watch: `gh pr checks "$PR" --watch` in the background, or check back once.
- If screenshot/golden tests fail purely because goldens moved with main, that's the fix-ci
  skill's territory — hand off to it rather than improvising.

### 7. Report
One short summary: what was rebased onto what, conflicts resolved (files + one line each on how),
verification run and its result, pushed or blocked-and-why, CI status.

## Multiple PRs at once
"Rebase all the phase-N PRs" → order them by stack dependency (bottom-up), run steps 1–6 per
branch, and reuse one `git fetch`. Report per-PR at the end, not per-step.
