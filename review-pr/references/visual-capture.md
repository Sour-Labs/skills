# Before/After Visual Capture

Playbook for capturing comparable screenshots of a PR's UI changes: the project built at
the PR's **merge-base** ("before") and at the PR's **head** ("after"), driven to the same
screens with the same data. Used by the `review-pr` skill; skipped entirely under
`--no-screenshots`.

§1 applies to every project. Then read only the section for the profile detected in the
skill's Step 2: §2 web, §3 Android, §4 iOS, §5 surfaces with no interactive UI. §6 is the
manifest every profile writes.

## 1. Plan the capture (all profiles)

**Pick the screens.** From the diff, list the UI-affecting files, then map each one to a
*reachable screen or route*. Useful sources, in order:

1. The project's own routing definition: a web router (`app/`, `pages/`, a routes table),
   a navigation graph, a storyboard, a deep-link declaration.
2. Component stories or demo screens (Storybook, a preview catalog, a debug menu). A
   shared component often has one there already.
3. The call sites: grep for the changed component or screen and pick the 1 or 2 most
   prominent places that render it, then say so in the caption.

Cap the plan at about 4 screens and about 8 screenshots. If a changed surface is
unreachable (error states, dialogs deep in a flow, push-triggered UI), do not burn time
simulating it. Capture the nearest reachable state and record the gap in the manifest
`notes`.

**Feature flags.** If the diff gates new UI behind a flag or config value, the "after"
capture must enable it, using whatever mechanism the project provides (an override file,
a query parameter, an env var, a debug menu, a repo-specific flag tool). The "before"
build predates or defaults the flag; capture it as-is. Record the flag state in each
caption.

**Identity discipline.** Same account, same seeded data, same navigation path, same typed
text, same window or device size, same scroll position for the before/after pair. The
only visible difference should be the PR's change. If the screen renders live content that
differs between runs, prefer a deterministic surface (settings, a form, a static entity
page) or note the noise in the caption.

**Honesty rule.** Never present screenshots from the same build as a before/after pair,
and never crop or edit beyond trimming a status bar or browser chrome.

## 2. Web

Two checkouts can run at the same time on different ports, so this profile is fast. Work
in detached worktrees; never touch the developer's checkout.

```
git fetch origin "pull/$PR/head"
git worktree add --detach .claude/worktrees/review-pr-$PR-head FETCH_HEAD
# BASE_SHA = git merge-base FETCH_HEAD origin/<baseRefName>   (baseRefName from gh pr view)
git worktree add --detach .claude/worktrees/review-pr-$PR-base <BASE_SHA>
```

In each worktree, install with the project's own lockfile command (`npm ci`,
`pnpm i --frozen-lockfile`, `yarn --immutable`) and start the app on a distinct port, for
example `PORT=5301` for base and `PORT=5302` for head. Prefer the production preview
(`build` then `preview` or `start`) when it is quick, because dev overlays and HMR banners
pollute screenshots; fall back to the dev server when the build is slow. Copy any `.env`
the project needs from the developer's checkout, and never print or commit its contents.
If a database or API is required, reuse the project's seed or fixture command so both runs
see identical data.

Capture with whatever browser automation is available, in this order: a connected browser
MCP server (Playwright or Chrome DevTools), then the project's own end-to-end tooling
(`npx playwright screenshot <url> <file>`, a Playwright or Cypress spec), then a headless
Chrome invocation. Whichever one you use, keep it identical across the two runs.

Determinism rules that matter more on web than anywhere else:

- Fix the viewport. Use 1440x900 for desktop and 390x844 for mobile. Capture the mobile
  size too when the diff touches responsive styles, and say which is which in the caption.
- Disable animation and caret blink before the shot (inject
  `* { animation: none !important; transition: none !important; }`, or use the automation
  tool's reduced-motion option), and wait for fonts and network idle.
- Use the same theme (light or dark) and the same locale for both runs.
- Log in once per run with a test account, or reuse a stored session state file, so both
  runs render the same user.
- Prefer a full-page screenshot for layout changes and a clipped element screenshot for a
  single component.

Stop both servers when the captures are done, and remove both worktrees.

## 3. Android

Cost is roughly two app builds, each up to about 30 minutes, so start them while the
review agents work, not after. Long silent stretches are normal; investigate only after
15 or more silent minutes (`./gradlew --status`).

Create the head and base worktrees exactly as in §2, then build each one sequentially,
because parallel Gradle invocations fight over daemons:

```
(cd .claude/worktrees/review-pr-$PR-base && ./gradlew :app:assembleDebug)
(cd .claude/worktrees/review-pr-$PR-head && ./gradlew :app:assembleDebug)
```

The APK lands at `<worktree>/app/build/outputs/apk/debug/app-debug.apk`. Copy each one
into the report's scratch dir as `base.apk` and `head.apk` before building the other, then
remove the worktrees. If the **base** build fails, do not fight it: proceed with
after-only screenshots and record why. If the **head** build fails, that is a blocking
review finding in itself.

Device, install, login:

- Start or pick an emulator (`adb devices`, or the repo's own device script when it has
  one). **Verify the AVD** with `adb -s $DEVICE_ID emu avd name`. A script that boots the
  first AVD alphabetically may pick a desktop or tablet profile. Captures must use a phone
  AVD; boot it directly with `emulator -avd <name> -no-snapshot-load` if the wrong one came
  up, and keep the same AVD for both runs.
- Disk: a debug APK can be around 300 MB. If installs fail on space, uninstall the app's
  package first (`adb -s "$DEVICE_ID" uninstall <applicationId>`). Never clear GMS data.
- Install and launch:
  `adb -s "$DEVICE_ID" install -r <base.apk>` then
  `adb -s "$DEVICE_ID" shell am start -a android.intent.action.MAIN -c android.intent.category.LAUNCHER <applicationId>`
- Log in once. Because installs use `-r`, the session survives the base-to-head reinstall.

Navigate by deep link where possible:
`adb -s "$DEVICE_ID" shell am start -a android.intent.action.VIEW -d "<deeplink>" <applicationId>`.
Otherwise drive the UI with taps, stop after about 5 exploratory taps, and record the gap.
Wait 3 to 5 seconds after a launch or deep link and 2 to 3 seconds after in-app
transitions. Dismiss debug overlays such as LeakCanary (force-stop and relaunch if they
persist). Screenshot with a mobile MCP tool (`mobile_save_screenshot` to
`<report-dir>/assets/<screen-slug>/{before,after}.png`) or `adb exec-out screencap -p`,
and verify each capture inline, because a screenshot of the wrong screen is worse than
none. Then install the head APK with `-r`, enable any feature flags, relaunch, and repeat
the exact same list.

**In the Nextdoor Android repo** these repo-specific helpers replace the generic steps:
`./scripts/start-device.sh --max-time-limit 30m` for the device, the `auto-login` skill
(`check-login.sh`, then `nd-auto-login list --json` and `auto-login.sh`) for the session,
the `lc-control` skill for launch controls, and for screen discovery `AGENTS_NAVROUTES.md`,
`AGENTS_FEATURES.md`, `grep -r "demoUrl" --include="*Router*.kt"`, and
`python3 .claude/skills/create-ui-test/navgraph.py navigate-to maestro/navigation-graph
--platform android --to <screen-id>`. Any `https://nextdoor.com/...` path generally works
as `nextdoor://nextdoor.com/...`, and the debug package is `com.nextdoor.android.debug`.

## 4. iOS

Same two-worktree shape as §3. Build each worktree for a simulator, for example
`xcodebuild -scheme <Scheme> -destination 'platform=iOS Simulator,name=iPhone 16' build`,
or the project's own Fastlane lane or Makefile target when it has one. Use the same
simulator model for both runs.

Boot with `xcrun simctl boot "<device>"`, install with
`xcrun simctl install booted <path>.app`, launch with
`xcrun simctl launch booted <bundleId>`, and open deep links with
`xcrun simctl openurl booted "<url>"`. Capture with
`xcrun simctl io booted screenshot <file>.png`. Everything else follows §1: same account,
same route order, same waits, and flags enabled only on the "after" run.

## 5. Surfaces with no interactive UI

A `service` or `infra` PR usually has nothing to screenshot, and the report must say so
plainly rather than invent visuals. Three exceptions are worth capturing:

- **Rendered HTML or email templates**: render the template at base and at head into
  files, open each in a browser, and screenshot as in §2.
- **CLI output**: run the same command against both builds and save the terminal text.
  Present it as a before/after code block pair, not as an image.
- **API responses**: send the same request to both builds and diff the JSON. This belongs
  in the explanation section, not the screenshots section.

Anything else: skip the visuals, and record in the coverage section that the PR had no
visual surface.

## 6. Manifest for the report

Write `<report-dir>/assets/screens.json` so the page assembler can render pairs without
guessing:

```json
{
  "screens": [
    {
      "slug": "checkout-summary",
      "title": "Checkout summary",
      "route": "http://localhost:5302/checkout/summary",
      "viewport": "desktop 1440x900",
      "flag": "new_checkout_summary=on (after only)",
      "before": "assets/checkout-summary/before.png",
      "after": "assets/checkout-summary/after.png",
      "caption": "Line-item spacing and the new tax breakdown row",
      "notes": null
    }
  ],
  "skipped": [
    { "reason": "Error state only reachable by killing the network mid-request", "files": ["src/checkout/ErrorBanner.tsx"] }
  ]
}
```

`route` holds whatever addresses the screen for that profile: a URL, a deep link, or a
navigation path. `viewport` names the window or device size, so the page assembler knows
whether to render the pair at phone width or full width. `before` may be null when the
base build failed or the screen did not exist yet; a brand-new screen legitimately has no
"before", and the page renders the after full width with a "new screen" badge.
