---
name: address-pr-comments
description: Triage and address reviewer comments on a GitHub PR, reply to each in a brief human voice, and push the fixes. Use when asked to address PR comments/feedback, respond to reviewers, handle review threads, or to check/look at/go through the comments on a PR (number or link), or to babysit/watch a PR for new feedback. Triggers on "address pr comments", "check the comments on <PR link>", "check comments on PR <n>", "look at the comments on this PR", "go through the PR comments", "review the comments on <PR>", "address review feedback", "respond to reviewers", "handle pr feedback", "babysit the PR", "watch the PR".
---

# Address PR comments

Read a PR's review feedback, make the changes that are warranted, push them, and leave a short
human reply on each thread so the reviewer can see it was handled. Works in any repo via `gh`.

## Inputs

- Run this **from inside the PR's local repository**.
- Argument: a PR number or URL. If none is given, infer it from the current branch.
- The user may add scope hints (e.g. "only the comments from @alice", "skip the nit about naming").
  Honor them.

## Mode: confirm-first vs. autonomous

Pick the mode from how the request was phrased:

- **Autonomous** (action phrasings) — "address", "fix", "handle", "resolve", "respond to",
  "reply to" the comments. Run the whole procedure end to end without pausing (the step-4
  decision/disagreement guardrail still applies).
- **Confirm-first** (pure read phrasings) — "check", "look at", "go through", "review" the
  comments, with no action verb. Do steps 1–4 only (gather + triage — all read-only), then
  **stop and show the user** the comment list plus your triage plan (what you'd change, what
  you'd skip and why). Make **no** edits, commits, pushes, or replies until they confirm. Once
  they say go, continue from step 5 as autonomous.

When the phrasing is mixed or unclear, default to confirm-first — it's the safer pause.

## Procedure

### 0. Preconditions
- `gh auth status` succeeds.
- Working tree is clean (`git status --porcelain` empty). If not, stop and ask — don't stash or
  discard the user's work.

### 1. Resolve the PR and repo
```sh
REPO=$(gh repo view --json owner,name -q '.owner.login + "/" + .name')   # e.g. Nextdoor/android
OWNER=${REPO%/*}; NAME=${REPO#*/}
PR=<arg, or: gh pr view --json number -q .number>     # number only; strip a URL down to its number
ME=$(gh api user -q .login)
```

### 2. Get on the PR branch, up to date
```sh
gh pr checkout "$PR"      # checks out the head branch (handles forks)
git pull --ff-only        # be current before editing
```

### 3. Gather the feedback
Pull all three kinds; reviewers use them interchangeably.

**Inline review threads (the main source) — only unresolved ones:**
```sh
gh api graphql -f query='
query($owner:String!,$repo:String!,$pr:Int!){
  repository(owner:$owner,name:$repo){ pullRequest(number:$pr){
    reviewThreads(first:100){ nodes{
      id isResolved isOutdated path line
      comments(first:50){ nodes{ databaseId author{login} body createdAt } }
    }}
  }}
}' -F owner="$OWNER" -F repo="$NAME" -F pr="$PR"
```
- Keep threads where `isResolved == false`.
- The **reply anchor** is the *first* comment's `databaseId` in the thread.
- Skip a thread if its **last** comment author is `$ME` (already replied) or a bot
  (`*[bot]`, `codecov`, `danger`, etc.) — unless the user says otherwise.
- `isOutdated == true` means the code moved since; still read it, but it's lower priority.

**Review summary bodies (non-inline):**
```sh
gh api repos/$OWNER/$NAME/pulls/$PR/reviews -q '.[] | select(.body != "") | {user:.user.login, state, body}'
```

**Top-level PR comments:**
```sh
gh pr view "$PR" --json comments -q '.comments[] | {user:.author.login, body}'
```

### 4. Triage every item
Classify each before touching code:

| Class | What it is | Action |
|---|---|---|
| **Actionable** | A concrete change request you agree with | Make the change |
| **Question** | Reviewer is asking, not requesting | Answer in the reply; change only if the answer implies one |
| **Decision / disagreement** | Needs a product call, or you think it's wrong, or it's ambiguous/large | **Don't guess.** Surface it to the user and ask before acting or replying |

Never fabricate agreement or invent a rationale. If you wouldn't stand behind the change, ask first.

### 5. Make the changes
> **Checkpoint:** in **confirm-first** mode, do not start editing, committing, pushing, or replying
> until the user has approved the triage plan. In autonomous mode, proceed.

- Group related comments into coherent edits.
- **Follow the project's conventions.** Check `CLAUDE.md` / `AGENTS.md` / `README` and any relevant
  skills for that repo before editing.
- Per the user's global rules: add or update **tests** for code changes, and make sure the code
  **builds and tests pass** before pushing. Detect the right build/test command for the touched
  modules (Makefile, `package.json` scripts, Gradle, etc.). **Do not push code that fails to
  compile or fails tests** — fix it or escalate.
- Run the repo's format check on the files you touched before committing (Nextdoor/android:
  `scripts/ktfmt.sh --dry-run --set-exit-if-changed <files>`, fix with `scripts/ktfmt.sh <files>`)
  so CI's Code Format Check can't go red on your push.

### 6. Commit
- One commit, or a few logically-grouped commits. Clear messages describing what was addressed
  (e.g. `fix(auth): validate token expiry per review`). Match the repo's commit/PR conventions.

### 7. Push
```sh
git push        # the branch already tracks its remote after `gh pr checkout`
```
If the push is rejected because the remote moved, `git pull --rebase` then push again. If auth
times out, just retry the push.

### 8. Reply to each handled thread — brief and human
Reply **after** pushing, so "fixed" is true. One short, first-person line per thread. Acknowledge +
say what you did. No preamble, no restating their comment, no signature.

**Inline thread reply** (anchor = first comment's `databaseId`):
```sh
gh api --method POST repos/$OWNER/$NAME/pulls/$PR/comments/<DATABASE_ID>/replies -f body="Good catch — fixed."
```
**Top-level / review-summary feedback** (no thread to reply into): post one consolidated comment:
```sh
gh pr comment "$PR" --body "Addressed the import ordering and the null check — pushed."
```
Optional — only if the user asks, or the repo's norm is author-resolves: resolve a thread after
replying:
```sh
gh api graphql -f query='mutation($id:ID!){ resolveReviewThread(input:{threadId:$id}){ thread{ isResolved } } }' -F id="<THREAD_ID>"
```

### 9. Summarize to the user
Report: what you changed, what you **skipped and why** (decisions/disagreements you flagged), the
replies you posted, and the build/test result.

## Reply voice

Sound like a teammate dashing off a quick note. Vary the wording. Keep it to a sentence.

Good:
- "Good catch — fixed."
- "Done, pulled this into a helper so it's reused."
- "Switched to `requireNotNull` like you suggested."
- "Yep, that was dead code. Removed."
- "Renamed it to `isEligible` for clarity."

Avoid:
- "Thank you for your valuable feedback. I have addressed your concern." (robotic)
- Restating the comment back at them.
- Multi-paragraph justifications. If it needs that much explaining, it's probably a *decision* —
  ask the user instead.
- Posting a reply on a thread you didn't actually address.

## Babysitting a PR (looping)

To keep a PR tended while reviews trickle in, run this skill on a loop instead of re-typing the
request:

```
/loop 20m /address-pr-comments <pr-number-or-url>
```

Each pass, besides the comment procedure above:
1. **Check CI first**: `gh pr checks "$PR"`. If checks are red, use the `fix-ci` skill's
   classification (formatting / goldens / flaky / infra / real) to remedy before or alongside
   comment handling.
2. **Nothing new?** If there are no new unresolved threads and checks are green, say "no new
   comments, checks green" and end the pass quickly — don't invent work.
3. **PR merged or closed?** Report it and tell the user to cancel the loop.

Loop passes run unattended, so in a loop the **autonomous** mode applies — but the step-4
decision/disagreement guardrail still holds: park judgment calls, list them for the user, and
don't act on them until a human weighs in.

## Guardrails
- **Ask, don't guess**, when a comment needs a decision, looks wrong, or is ambiguous/large.
- Don't touch resolved threads, bot threads, or threads you've already replied to.
- Don't reply unless the thread was genuinely addressed or answered.
- Never discard uncommitted user work to get a clean tree — stop and ask.
- Keep replies truthful: reply after the fix is pushed.
