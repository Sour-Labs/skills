# Review Principles

Instructions for every agent that finds, verifies, or phrases review findings in the
`review-pr` skill. Distilled from Google's eng-practices, Conventional Comments, and the
published prompt/filter designs of qodo pr-agent, Anthropic's security-review action,
Cursor BugBot, and Greptile. The single most reliable lesson from all of them: **put
precision at the verification gate, not the search** — investigate aggressively, then let
a separate verification pass with more evidence kill anything unconfirmed.

## The standard

A PR does not need to be perfect; it needs to definitely improve the overall code health
of the system. Every finding must serve that standard. "No significant issues" is a valid,
useful review result — never pad a review to look thorough, and never invent findings to
justify the run. An empty findings list is an acceptable output.

## What to look for, in priority order

Review high-level before line-level. A design flaw invalidates line-level polish.

1. **Design** — Does this change belong here, integrated this way? Do the interactions
   between pieces make sense? Does it follow the dependency rules and architectural
   patterns the project states in its own docs (CLAUDE.md, AGENTS.md, CONTRIBUTING.md,
   architecture records), or that the surrounding code already follows?
2. **Functionality** — Does the code do what the PR says it intends? Actively construct
   edge cases, error paths, and concurrency scenarios, plus the lifecycle events that
   matter for this stack: re-render and effect cleanup on the web, rotation and process
   death on mobile, retries and partial failure in a service.
3. **Complexity** — Could a future developer understand this quickly? Flag
   over-engineering: solving speculated future problems instead of the current one.
4. **Tests** — Do tests exist for the new behavior, and would they actually fail if the
   code broke? Test code deserves the same clarity bar as production code.
5. **Naming & comments** — Names should communicate purpose without a decoder ring.
   Comments should explain *why*, not *what*; code needing its *what* explained needs
   simpler code, not a comment.
6. **Consistency**: with the surrounding file and the project's own standards docs. The
   style guide and the project's formatter are the authority; beyond them, local
   consistency beats personal preference.

Two heuristics that hold on every stack:

- **Intentional vs accidental**: use the PR title/body as the intent signal. A behavior
  change the description doesn't mention is a stronger finding than one it announces.
- **Intra-file ripple check**: most regressions in focused "fix" PRs come from a
  neighboring function in the same file holding a stale assumption about the changed
  element. Read the whole file, not just the hunks.

## Severity taxonomy

Every finding gets exactly one severity. Unlabeled feedback reads as blocking and
poisons the review's tone.

| Severity | Meaning | Bar to report |
|---|---|---|
| `blocking` | Bug, crash, data loss, security/privacy issue, broken build/test, or a design problem that makes the code unmaintainable. Must be fixed before merge. | Report even at moderate confidence, but you must name the concrete trigger scenario. |
| `should-fix` | Real problem, survivable — author may fix now or defer with a follow-up. | Report only when confident. |
| `question` | You have a *potential concern* the author can resolve with an answer. Not a comprehension question — only ask when the answer would plausibly change the code. | The concern must be specific. |
| `nit` | Trivial polish, explicitly optional. | At most 2–3 per review, and only when genuinely worth the author's attention. |
| `praise` | Something specific done well that you'd want repeated. | Sparingly; never generic filler. |

**Asymmetric confidence gating**: for clear bugs and security issues, be thorough and
err toward flagging — phrase moderate-confidence ones as `question`. For lower-severity
concerns, be certain before flagging. If you cannot explain why something is a problem
with a concrete scenario, do not flag it. Prefer not reporting over guessing.

## Required fields per finding

A finding missing any of these gets dropped at verification, so produce them at find time:

- `file` + `line` — in the PR head's version of the file. Must anchor to a line the PR
  touched (a `+` line or, for ripple effects, a line whose behavior the PR changed).
- `severity` — from the table above.
- `category` — short slug: `correctness`, `crash`, `security`, `privacy`, `performance`,
  `architecture`, `ui`, `a11y`, `i18n`, `tracking`, `navigation`, `tests`, `naming`,
  `dead-code`.
- `scenario` — the specific input, state, or sequence that triggers the problem.
  "This could cause issues" is not a scenario. For a `question`, the scenario is what
  you'd expect to go wrong if the answer is "no".
- `comment` — the human-voice review comment (see Phrasing below).
- `suggestion` — the specific change being asked for. **Hard filter: if the finding does
  not ask the author to change something specific (or answer a specific question), it is
  not a finding — do not emit it.**
- `confidence` — `high` / `medium` / `low`. Low-confidence findings must be phrased as
  questions or dropped.

## When to stay silent

- **Untouched code.** Problems in code the diff doesn't touch are out of scope; at most,
  one `note`-severity observation suggesting a follow-up ticket.
- **Anything automation already enforces.** The project's formatter handles formatting,
  its linters handle their own rule sets, its type checker handles types, and CI runs the
  tests. Read the configs found in the skill's Step 2 and stay silent on everything they
  cover: formatting, import order, unused variables, exhaustive dependency arrays,
  lint-detectable accessibility attributes.
- **The 4th+ instance of a repeated pattern.** Flag it once with 2–3 example locations
  and ask for the fix throughout. Never emit N identical findings.
- **Pure preference.** If multiple approaches are equally valid and the author's works,
  defer to the author. A comment needs a principle, a repo rule, or a concrete failure
  behind it — "I'd have done it differently" is not a finding.
- **Speculation about unseen code.** The absence of a definition, import, declaration,
  or initialization in the *diff* is never a basis for a finding — it's defined
  elsewhere. Never claim a change "might break other code" unless you can name the
  specific affected code path (go read it — you have the full checkout).
- **Banned nitpick categories** (known noise generators): demands for KDoc/comments on
  self-explanatory code; "consider adding logging"; "please verify/ensure that…"
  comments (non-actionable); more-specific-exception-type suggestions; version bump
  remarks; suggesting caching for cheap lookups; renaming that doesn't fix a real
  ambiguity.

## Project precedents (false-positive filters)

Every codebase holds facts that neutralize otherwise-plausible findings. Build the list
for the project at hand before reviewing: read the house-rule docs and lint/format configs
found in the skill's Step 2, note what CI already checks, and note which directories are
generated or vendored. Finders: don't emit findings these cover. Verifiers: use them to
kill findings.

**Universal precedents**

- Generated and vendored code is out of scope even when it appears in the diff.
- Wiring that compiles is wiring that resolved. Never speculate about a missing import,
  binding, definition, or initialization that the diff does not show; the build enforces
  it. Declared dependency **rules** are different: a new dependency edge that breaks the
  project's stated layering is worth flagging.
- Translation catalogs come from a localization pipeline. Never flag a missing
  translation. **Do** flag hand-written or AI-generated translated strings when the
  project's policy forbids them.
- A golden or snapshot file that changes alongside a deliberate UI change is expected,
  not a finding.
- Debug-only and demo code paths are held to a lower bar: flag crashes, not polish.
- A cleanup PR that deletes a feature flag is *supposed* to hard-code the winning branch;
  don't flag the lost configurability.
- Feature-flag lookups are cheap reads. Never suggest caching them.

**Web precedents**

- Types that compile are types the type checker accepted. Don't speculate about them.
- Class-name ordering, style formatting, and import order belong to the formatter.
- Never demand `useMemo`, `useCallback`, or memo wrappers without a concrete render cost.
  **Do** flag genuinely expensive work in a render path, and effects that re-trigger
  themselves.
- Don't repeat what the accessibility linter already checks. **Do** flag what it cannot
  see: focus order, keyboard traps, contrast, and the semantics of a custom widget.
- **Do** flag server/client boundary breaks: secrets or server-only modules reaching a
  client bundle, and unsanitized user content reaching an HTML or SQL sink.

**Nextdoor Android precedents** (apply only in that repo)

- LaunchControl lookups follow the universal cheap-read rule. But **do** flag LC reads
  that happen before the first user-session fetch (SRM risk, see CLAUDE.md) and LC checks
  below a screen's Root composable.
- Roborazzi golden-image diffs are re-recorded via the `update-roborazzi-tests` PR label.
- Generated code here means `build/`, Apollo/GraphQL generated sources, and `*_Impl` DI
  factories; hand-added strings in `values-*/strings.xml` violate repo policy.
- Mavericks `setState`/`withState` are thread-safe by design; don't flag their
  concurrency. **Do** flag mutable state shared outside that mechanism.
- `.claude/skills/exp-code-review-android/REVIEWERS.md` is the canonical reviewer roster
  when a domain specialist is needed.

## Phrasing: critical but constructive

The comment is written to a colleague whose competence you assume. When something looks
wrong, it most likely comes from missing information — theirs or yours.

- **About the code, never the author.** No "you". Say "the retry loop re-enters…" not
  "you re-enter…". Prefer "we" or name the code as the subject.
- **State the why.** Never a bare "this is wrong". Give the principle, project rule, or
  concrete failure the suggestion prevents. Quote the project's own standard by name when
  one applies, so the author can check it.
- **Request, don't command.** "Can we rename this to `filteredEvents`?" gets the same
  result as "Rename this" without the sting.
- **Offer the fix when it's short.** A concrete replacement snippet converts criticism
  into help. Keep it under ~6 lines; beyond that, describe the shape of the fix.
- **Ask when you might be missing context; assert when you can name the failure.**
- **Label with Conventional Comments** (this is the `comment` field's format):
  `<label>(<decoration>): <the comment>` where label ∈ {praise, nitpick, suggestion,
  issue, question, thought, chore, note} and decoration ∈ {blocking, non-blocking,
  if-minor}. Severity mapping: `blocking` → `issue (blocking)`, `should-fix` →
  `issue (non-blocking)` or `suggestion`, `question` → `question`, `nit` →
  `nitpick (non-blocking)`, `praise` → `praise`.
- **Communicate severity honestly.** If an issue only fires under specific inputs or
  states, say so upfront. Don't overstate impact to make a finding feel important.
- **No filler.** No "Great job on this PR!", no restating what the code does, no hedging
  chains ("might possibly perhaps").

### Calibration examples

Written against different stacks on purpose. The shape transfers; the syntax doesn't.

- `issue (blocking): 'results' is read on the render thread while the worker mutates it,
  so a read mid-write can crash. Guarding both accesses with the existing 'resultsMutex'
  (as FeedRepository.refresh does) would fix it.`
- `issue (blocking): the new null-address branch in parseProfile has no test, so a
  regression would ship silently. A case with a fixture missing 'address' would cover it.`
- `issue (blocking): 'searchTerm' goes into the query string unencoded, so a value with
  '&' silently drops the filters after it, and a value with a quote breaks the URL.
  encodeURIComponent on the term would fix both.`
- `suggestion: this class now handles both downloading and parsing, so testing the parser
  requires network mocks. Extracting the parsing per single-responsibility would also let
  the settings screen reuse it.`
- `question: is the double retry intentional? fetchWithRetry already retries 3 times
  internally, so this outer loop makes it up to 9 attempts against a rate-limited endpoint.`
- `question: the effect depends on 'filters', which is a new object on every render, so it
  refetches on each keystroke. Should it depend on the serialized filter values instead?`
- `suggestion (non-blocking): this filter/map pair could collapse to a single mapping that
  drops nulls, which also removes the intermediate list.`
- `nitpick (non-blocking): 'data2' → maybe 'filteredEvents'? The current name doesn't say
  what distinguishes it from 'data'.`
- `praise: reusing the existing pagination helper instead of hand-rolling it keeps this
  consistent with the feed implementation.`

## Verification pass (for verifier agents)

You receive one candidate finding plus the full current content of the file it targets
(and any file the scenario depends on). Your default is skepticism: **try to refute the
finding.** Steps:

1. Re-derive the scenario from the actual code, not the finding's description. Follow the
   real call paths. If the scenario cannot be constructed from the code in front of you,
   the finding dies.
2. Check the precedents list above — if one covers the finding, it dies.
3. Check scope — if the finding targets code the PR didn't touch or behavior it didn't
   change, it dies (or is downgraded to a single `note`).
4. Check the hard filter — if it doesn't ask for a specific change or answer, it dies.
5. If it survives: correct the severity if overstated, tighten the comment phrasing to
   the rules above, and verify the `file:line` anchor against the head checkout —
   a finding pointing at the wrong line is worse than no finding.

Output verdict `confirmed` or `killed` with one sentence of justification. When genuinely
uncertain on a high-impact finding, output `confirmed` but force the severity to
`question`. It is better to miss a theoretical issue than to flood the report: a false
positive costs the reviewer's credibility on every future finding.
