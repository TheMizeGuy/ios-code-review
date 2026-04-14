---
name: review-ios
description: |-
  Use this skill when the user asks for an iOS / Swift / SwiftUI / UIKit code review, says "review my iOS app", "check my Swift code", "audit this for App Store readiness", "will Apple reject this?", "TestFlight review", "pre-submission review", "iOS engineering review", or wants comprehensive review covering App Store compliance, privacy, entitlements, security, HIG, accessibility, SwiftUI patterns, deep linking, concurrency, or performance. Also use proactively before App Store submission, after a TestFlight rejection, or when finishing a substantial iOS feature. Dispatches the ios-senior-review:senior-ios-reviewer agent (Opus 4.6) which simulates BOTH the Apple App Review team AND a senior Apple platform engineer, citing the local 88-file iOS Development vault.
argument-hint: '[path | file | "diff" | "staged" | "pr" | "all"] [--mode submission | --mode engineering]'
allowed-tools: Bash, Read, Grep, Glob, TodoWrite, Agent
---

# iOS Senior Review

You are coordinating a senior iOS code review on the user's behalf. Your job is to determine the scope, gather Apple-specific project context (Info.plist, entitlements, privacy manifest, build settings, deployment targets), and dispatch the `ios-senior-review:senior-ios-reviewer` agent (Opus 4.6) which does the actual review in two simultaneous modes — Apple App Review Simulation + Senior Engineering Review.

## Step 1: Parse arguments

The user passed an argument string (may be empty). Two parts:

1. **Scope** — first non-flag token (path, file, `diff`, `staged`, `pr`, `all`)
2. **Mode flag** — optional `--mode submission` or `--mode engineering`. Default: both.

| Argument | Meaning | How to resolve |
|---|---|---|
| (empty) or `diff` | Uncommitted + staged Swift/Obj-C changes | `git diff --name-only HEAD` filtered to `*.swift`/`*.h`/`*.m`/`*.mm`/`*.plist`/`*.xcprivacy`/`*.entitlements` |
| `staged` | Only staged | `git diff --cached --name-only` filtered |
| `pr` | Diff vs `main`/`master` | `git diff --name-only $(git merge-base HEAD main 2>/dev/null \|\| git merge-base HEAD master)` filtered |
| `<file>` | Single file | The path itself, after verifying it exists |
| `<directory>` | All Swift files in dir | Use Glob: `<dir>/**/*.{swift,h,m,mm}` excluding `.build`, `DerivedData`, `Pods`, `Carthage` |
| `all` | Entire project | Glob from project root excluding `.build`, `DerivedData`, `Pods`, `Carthage`, `.git` |

For **App Store submission readiness** reviews, default to `all` even if the user passed `diff` — Apple reviewers don't see your diff, they see the whole app. Tell the user you're expanding scope and why.

If the resolved file list is **empty**: tell the user, suggest a different scope, stop.

If it's **>100 files**: tell the user the count and ask whether to continue. iOS reviews benefit from full project context but cost more.

## Step 2: Find the project root and Apple-specific artifacts (run in parallel)

Gather these in a single message with parallel tool calls:

1. **Project root** — Run `git rev-parse --show-toplevel` (Bash). Then locate the `.xcodeproj` or `.xcworkspace` or `Package.swift`.
2. **Info.plist files** — Glob for `**/Info.plist` excluding `.build`, `Pods`, etc. Read each found.
3. **Entitlements files** — Glob for `**/*.entitlements`. Read each.
4. **Privacy manifests** — Glob for `**/PrivacyInfo.xcprivacy`. Read each. Note targets MISSING manifests.
5. **Package.swift / Podfile / Cartfile** — Read whichever exists; identify third-party deps.
6. **Build settings** — Read `*.xcodeproj/project.pbxproj` or look for `.xcconfig` files. Note: `SWIFT_VERSION`, `IPHONEOS_DEPLOYMENT_TARGET`, `SWIFT_STRICT_CONCURRENCY`, `ENABLE_THREAD_SANITIZER`, `SWIFT_TREAT_WARNINGS_AS_ERRORS`.
7. **Localization** — Glob for `*.xcstrings` and `*.strings` to know if String Catalogs are in use.
8. **App Clip / extensions** — Look for separate targets (`*App Clip*`, `*Extension*`, `*Widget*`).
9. **CI / linter config** — Glob for `.swiftlint.yml`, `.periphery.yml`, `Mintfile`, `.github/workflows/*.yml`.

If `git` is unavailable (not a repo) and the user asked for a diff-based scope, fall back to `all` and tell the user.

## Step 3: Construct the agent prompt

Build a single self-contained prompt for the agent. Zero conversation context inheritance — bake everything in.

```
SCOPE — review these files:
<absolute path 1>
<absolute path 2>
...

PROJECT ROOT: <absolute path>

PROJECT CONTEXT:
- Project type: <iOS app / iPadOS / watchOS / tvOS / visionOS / library / SPM package>
- Build system: <Xcode project / Xcode workspace / Swift Package>
- Deployment target: iOS X.Y, macOS X.Y, watchOS X.Y, etc.
- Swift version: <SWIFT_VERSION>
- Strict concurrency: <SWIFT_STRICT_CONCURRENCY value or "default">
- Targets found: <main app>, <App Clip>, <NSE>, <Widget Extension>, <watchOS companion>, etc.
- Entitlements per target: <list — actual values, not just presence>
- PrivacyInfo.xcprivacy: <present in main / missing in <target> / not found>
- Info.plist highlights: <NSUserTrackingUsageDescription, UIBackgroundModes, CFBundleURLTypes, NS*UsageDescription strings>
- Third-party deps: <SPM packages / CocoaPods / Carthage>
- Localization: <String Catalogs used / .strings files / hardcoded strings detected>
- Linter config: <swiftlint / periphery / none>
- Test scheme: <ENABLE_THREAD_SANITIZER value, test target name>
- App Store Connect artifacts available: <yes / no — if no, scope verdict accordingly>

KNOWLEDGE BASE: Use training knowledge, Context7 for Apple framework docs, and WebSearch for fresh guideline updates. If the user maintains a local iOS vault or docs folder, pass its path here and the agent will read it. If the user has a GoodMem server with a Learnings space, pass the space UUID here and the agent will query it.

MODE: <both | submission | engineering>

TASK:
1. Activate serena on the project root: mcp__plugin_serena_serena__activate_project(<project root>)
2. Map the project structure with serena get_symbols_overview / list_dir / list_memories.
3. Read every file in scope completely.
4. Read the relevant vault files (match scope to vault per your system prompt's dimension table).
5. Search GoodMem Learnings with iOS-specific queries (frameworks, features, patterns you see).
6. Run the tooling if available:
   - swiftlint lint --reporter json (capture findings)
   - periphery scan --format json (dead code)
   - xcodebuild analyze (if scheme can be determined and build is feasible)
7. Review across all 12 dimensions per your system prompt. Run BOTH modes unless --mode passed.
8. Return findings in the strict format from your system prompt: tag, evidence class, file:line, current code, suggested fix, vault citation.
9. Both summary tables, both verdicts, recommended next steps, raw tooling output.

CONSTRAINTS:
- Read-only review. Do NOT modify any files.
- Cite vault files in every finding where applicable.
- Cite Apple guideline numbers for App Review findings.
- Don't issue [R] from RUNTIME / ASC evidence — use [R?] and state the evidence gap.
- Tier 3-4 (engineering quality) findings do NOT affect the submission verdict.
- Don't manufacture findings. Signal > noise.
- Don't use AI slop, hedges, or emojis.
```

## Step 4: Dispatch the agent

Use the Agent tool with these arguments:

- `subagent_type`: `"ios-senior-review:senior-ios-reviewer"`
- `description`: `"Senior iOS review of N files (mode: both/submission/engineering)"`
- `prompt`: the prompt constructed in Step 3
- Run in **foreground** (do NOT use `run_in_background: true`) — the user wants real-time progress and iOS reviews can take a while

## Step 5: Present results

When the agent returns:

1. Display the agent's full report verbatim. Do not summarize, condense, or reformat. The user wants the raw output including both summary tables and both verdicts.

2. After the report, prompt:

   ```
   Apply any of these findings? Tell me which:
   - "all [R] and [R?]" / "all rejection-grade"
   - "all submission findings" / "all engineering findings"
   - "finding 3 and 7"
   - "everything in <filename>"
   - "skip" if you want to handle it yourself
   ```

3. **Do NOT auto-apply findings.** The user explicitly chooses. Reviewer findings are advisory.

4. If the user asks to apply specific findings, **you (the orchestrator) make the edits using Edit/Write**. Do not re-dispatch the reviewer agent for fixes — it's a reviewer, not an implementer. Apply each requested finding's "Suggested fix" to the indicated file:line. Be careful with `.plist`, `.entitlements`, `.xcprivacy` files — these are XML and need precise edits.

5. After applying any fixes, offer:
   - Re-run the review on the same scope to verify nothing regressed
   - Run `swiftlint` to confirm style fixes are clean
   - Run `xcodebuild build` to confirm the project still compiles
   - Write a learning to GoodMem if a non-obvious rejection pattern came up

## Notes on agent behavior

- The reviewer is a fresh-context Opus agent. It does NOT see this conversation. Everything it needs goes in the dispatch prompt.
- The agent has Read, Grep, Glob, Bash, GoodMem retrieve, serena, Context7, WebSearch, WebFetch, TodoWrite. It does NOT have Edit/Write/Agent — by design.
- The agent runs BOTH modes by default: App Review Simulation + Senior Engineering Review. Pass `--mode submission` or `--mode engineering` to limit.
- If the user has a local iOS reference vault or GoodMem Learnings space, include the path/UUID in the dispatch prompt so the agent can cite it. Without them, the agent relies on training knowledge + Context7 + WebSearch + Apple's published guidelines.
- Submission verdict is independent of engineering verdict. An app can be `READY` for submission and `NEEDS WORK` for engineering — those are different concerns.

## When to skip parts of the workflow

- **No git repo**: skip diff-based scopes; default to single-file or directory mode.
- **No Xcode project / Package.swift**: tell the user; dispatch the agent anyway with file-level scope. Findings will be SOURCE-only.
- **No swiftlint / periphery installed**: skip those steps; tell the agent in the prompt that they're unavailable.
- **No App Store Connect access**: tell the agent; submission verdict will be limited to SOURCE-derivable findings. The agent will not assert `[R]` for ASC-only findings.
- **CI / sandboxed environment**: pass that context; no `xcodebuild`, no `swiftlint`, just static review.

## Anti-patterns to avoid

- Don't dispatch the agent without project context — Apple-specific findings need entitlements/plist/manifest data.
- Don't dispatch on diff scope for an App Store submission review — Apple sees the whole app, you should too.
- Don't run the agent in the background — iOS reviews are long, the user wants progress.
- Don't summarize the agent's findings — show them verbatim with both tables and both verdicts.
- Don't auto-apply findings — wait for the user's explicit selection.
- Don't mix submission and engineering findings into a single verdict — they're separately scored on purpose.
- Don't dispatch this skill recursively.
