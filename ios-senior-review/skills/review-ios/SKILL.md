---
name: review-ios
description: |-
  Use this skill when the user asks for an iOS / Swift / SwiftUI / UIKit code review, says "review my iOS app", "check my Swift code", "audit this for App Store readiness", "will Apple reject this?", "TestFlight review", "pre-submission review", "iOS engineering review", or wants comprehensive review covering App Store compliance, privacy, entitlements, security, HIG, accessibility, SwiftUI patterns, deep linking, concurrency, or performance. Supports two dispatch modes: STANDARD (default — dispatch the `senior-ios-reviewer` agent) and TEAM (requires the explicit phrase "ios team review" OR the `--team` flag — you act as team lead per the ios-team-lead manual and dispatch reviewer sub-agents in parallel waves). Team mode ONLY triggers on the specific phrase "ios team review" — generic "team review" or "full audit" alone do NOT activate team mode.
argument-hint: '[path | file | "diff" | "staged" | "pr" | "all"] [--mode submission | --mode engineering] [--team]'
allowed-tools: Bash, Read, Grep, Glob, Edit, Write, TodoWrite, Agent
---

# iOS Senior Review

You are coordinating a senior iOS code review on the user's behalf. Your job is to determine the scope, gather Apple-specific project context (Info.plist, entitlements, privacy manifest, build settings, schemes, deployment targets), and run one of two modes:

- **Standard review** (default) — dispatch `ios-code-review:senior-ios-reviewer` directly via the Agent tool. Single-agent review, fastest, best for small-to-medium codebases or narrow scopes.
- **Team review** ("ios team review" / `--team`) — YOU act as the team lead, following `agents/ios-team-lead.md` as your operating manual: map the codebase, partition into 4-10 non-overlapping scopes, dispatch `senior-ios-reviewer` sub-agents in parallel waves, run the runtime-verification and seam-review passes, consolidate into one unified report. No team-lead subagent is ever dispatched (plugin-namespaced subagents lose the Agent tool at runtime — a Claude Code platform limitation). Best for whole-project audits, multi-target projects, or pre-submission audits (30+ Swift files; 100+ is the sweet spot).

Both modes run BOTH App Review Simulation + Senior Engineering Review by default, controllable via `--mode`. Both agents are pinned `model: fable`.

## Step 1: Parse arguments and detect team trigger

The user passed an argument string (may be empty). Three parts:

1. **Scope** — first non-flag token (path, file, `diff`, `staged`, `pr`, `all`). If more than one non-flag token is present, use the first and tell the user the rest were ignored.
2. **Mode flag** — optional `--mode submission` or `--mode engineering`. If `--mode` is missing its value or the value isn't exactly one of those two, default to both and say so. Default: both.
3. **Team flag** — optional `--team`. Default: standard (single-agent).

Ignore unrecognized flags and mention which ones were ignored.

### Natural-language team trigger

Team mode also activates from natural language, but **ONLY on the exact phrase "ios team review"** (case-insensitive). Detection rules:

| User said | Team mode? |
|---|---|
| "ios team review" | YES |
| "iOS team review of my app" | YES |
| "do an iOS team review" | YES |
| "ios team review all" | YES |
| "team review" (no "ios") | NO — ambiguous, default to standard |
| "full audit" | NO — ambiguous, default to standard |
| "do a team review of my iOS app" | NO — "team review" isn't prefixed with "iOS" |
| "thorough review" / "comprehensive review" / "deep review" | NO — default to standard |
| `--team` flag passed explicitly | YES (the flag is unambiguous) |

**Rationale:** "team review" is a generic phrase that could mean many things. Team mode requires the more explicit phrase "ios team review" to avoid accidentally dispatching a team review when the user only wanted a thorough single-agent review. The `--team` flag remains available as the explicit opt-in.

**If you're unsure whether the user wants team mode** (e.g., they said "thorough review of my iOS app"), default to standard mode and mention team mode as an option: "Running standard review. If you want team mode (4-10 agents in a parallel wave, ~15-30 min), say 'ios team review' or add `--team`."

### Scope resolution

| Argument | Meaning | How to resolve |
|---|---|---|
| (empty) or `diff` | Uncommitted + staged + NEW files | Union of `git diff --name-only HEAD` and `git ls-files --others --exclude-standard`, filtered to `*.swift`/`*.h`/`*.m`/`*.mm`/`*.plist`/`*.xcprivacy`/`*.entitlements` — plain `git diff` never shows untracked files, and "review my new file before I commit" is the most common diff-scope request |
| `staged` | Only staged (tracked changes; new files must be `git add`ed to appear) | `git diff --cached --name-only` filtered |
| `pr` | Diff vs default branch | Try in order, each with `2>/dev/null`: `git merge-base HEAD main`, `git merge-base HEAD master`, `git merge-base HEAD origin/main`, `git merge-base HEAD origin/master`. If ALL four fail, STOP: tell the user no default branch was found and ask them to name one — never fall back to a bare `git diff` (an empty substitution silently diffs worktree-vs-index, a different question) |
| `<file>` | Single file | The path itself, after verifying it exists |
| `<directory>` | All reviewable files in dir | Glob `<dir>/**/*.{swift,h,m,mm,plist,entitlements,xcprivacy}`, then discard any result whose path contains `/Pods/`, `/Carthage/`, `/.build/`, `/DerivedData/`, or `/node_modules/` regardless of gitignore state (vendored deps are sometimes committed) |
| `all` | Entire project | Same pattern + post-filter from the project root |

For **App Store submission readiness** reviews, default to `all` even if the user passed `diff` — Apple reviewers don't see your diff, they see the whole app. Tell the user you're expanding scope and why.

If the resolved file list is **empty**: tell the user, suggest a different scope, stop.

If it's **>100 files**: tell the user the count. In STANDARD mode, ask whether to continue (iOS reviews benefit from full project context but cost more). In TEAM mode, 100+ files is the sweet spot — proceed.

**Team mode scope handling:**
- Team mode always uses a broad scope. If the user passed `diff`, `staged`, `pr`, a single `<file>`, or a `<directory>` with `--team`, expand to `all` automatically and tell the user why (partitioning across 4-10 agents needs whole-project breadth). Exception: a `<directory>` that already contains 50+ Swift files — ask whether to scope the team review to that directory or expand to the whole project.
- If the (possibly expanded) scope is under 30 Swift files, team mode ABORTS — tell the user it's below the floor, and run standard review instead unless they object. 30-99 files: proceed without asking. 100+: the sweet spot.

## Step 2: Find the project root and Apple-specific artifacts (run in parallel)

Gather these in a single message with parallel tool calls:

1. **Project root** — Run `git rev-parse --show-toplevel` (Bash). Then locate the `.xcodeproj` or `.xcworkspace` or `Package.swift`.
2. **Info.plist files** — Glob for `**/Info.plist`, post-filtered per the exclusion rule above. Read each found. For `<directory>` scope, glob within the scoped directory (plus the target's own plist if identifiable) — don't pull unrelated targets' artifacts into a scoped review.
3. **Entitlements files** — Glob for `**/*.entitlements`. Read each.
4. **Privacy manifests** — Glob for `**/PrivacyInfo.xcprivacy`. Read each. Note targets MISSING manifests.
5. **Package.swift / Podfile / Cartfile** — Read whichever exists; identify third-party deps.
6. **Build settings** — Read `*.xcodeproj/project.pbxproj` or `.xcconfig` files. Note: `SWIFT_VERSION`, `IPHONEOS_DEPLOYMENT_TARGET`, `SWIFT_STRICT_CONCURRENCY` / Swift language mode, `SWIFT_TREAT_WARNINGS_AS_ERRORS`. Enumerate targets from `productType = "com.apple.product-type...` lines — target-name substrings under-detect extensions.
7. **Shared schemes** — Glob for `**/xcshareddata/xcschemes/*.xcscheme`. Read each; extract `enableThreadSanitizer` / `enableAddressSanitizer` from the `TestAction`/`LaunchAction` elements (these are scheme-level diagnostics — there is no `ENABLE_THREAD_SANITIZER` key in pbxproj/xcconfig).
8. **Localization** — Glob for `*.xcstrings` and `*.strings` to know if String Catalogs are in use.
9. **CI / linter config** — Glob for `.swiftlint.yml`, `.periphery.yml`, `Mintfile`, `.github/workflows/*.yml`.
10. **Simulator availability** — `xcrun simctl list devices available | head -20` (Bash). Pass the result to the agent so it knows whether the runtime verification pass is possible.

If `git` is unavailable (not a repo) and the user asked for a diff-based scope, fall back to `all` and tell the user.

## Step 3: Construct the agent prompt

Build a single self-contained prompt for the agent. Zero conversation context inheritance — bake everything in.

**Standard mode** — pass the resolved file list directly:

```
SCOPE — review these files:
<absolute path 1>
<absolute path 2>
...
```

Full prompt template (standard mode):

```
BLACKBOARD: <project root>/.claude/blackboard/<session>/ios-review-<scope-slug>-<ts>.md
Write your FULL report (all findings, both tables, both verdicts, tooling output) to that path via Bash heredoc BEFORE returning; final message = the path + a ≤150-word summary with both verdicts.

PROJECT ROOT: <absolute path>

PROJECT CONTEXT:
- Project type: <iOS app / iPadOS / watchOS / tvOS / visionOS / library / SPM package>
- Build system: <Xcode project / Xcode workspace / Swift Package>
- Deployment target: iOS X.Y, macOS X.Y, watchOS X.Y, etc.
- Swift version + language mode: <SWIFT_VERSION, Swift 5/6 mode, SWIFT_STRICT_CONCURRENCY value or "default">
- Targets found (by productType): <main app>, <App Clip>, <NSE>, <Widget Extension>, <watchOS companion>, etc.
- Entitlements per target: <list — actual values, not just presence>
- PrivacyInfo.xcprivacy: <present in main / missing in <target> / not found>
- Info.plist highlights: <NSUserTrackingUsageDescription, UIBackgroundModes, CFBundleURLTypes, NS*UsageDescription strings, ITSAppUsesNonExemptEncryption>
- Scheme diagnostics: <enableThreadSanitizer/enableAddressSanitizer per shared scheme, test target name>
- Third-party deps: <SPM packages / CocoaPods / Carthage>
- Localization: <String Catalogs used / .strings files / hardcoded strings detected>
- Linter config: <swiftlint / periphery / none>
- Simulator: <available (device list) / unavailable — runtime pass impossible>
- App Store Connect artifacts available: <yes / no — if no, scope verdict accordingly>

KNOWLEDGE BASE (optional): <path to the user's local iOS docs/notes if one exists; omit this line otherwise>

MODE: <both | submission | engineering>

SCOPE — review these files:
<list>

TASK:
1. Map the project structure with Glob/Grep/Read.
2. Read every file in scope completely.
3. Read the local knowledge base if one was provided above; otherwise rely on the canonical Apple sources in your system prompt.
4. Run the static tooling if available (per your system prompt's tooling table — swiftlint, the two-step swiftlint analyze recipe, xcodebuild analyze with -project/-workspace AND -destination, periphery).
5. If a simulator is available, run the runtime verification pass from your system prompt (build_run_sim / test_sim / screenshot at default + .accessibility3 / snapshot_ui — or xcodebuild/xcrun simctl via Bash) and verify your RUNTIME-class findings.
6. Review across all 12 dimensions per your system prompt. Run BOTH modes unless --mode passed.
7. Produce findings in the strict format: tag, evidence class, file:line, current code, suggested fix, source citation.
8. Both summary tables, both verdicts, recommended next steps, raw tooling output — all written to the BLACKBOARD path first.

CONSTRAINTS:
- Read-only review. Do NOT modify any files (the blackboard heredoc write is the one exception).
- Cite authoritative sources in every finding.
- Cite Apple guideline numbers for App Review findings.
- Don't issue [R] from RUNTIME / ASC evidence — use [R?] (marked "verified" if you reproduced it) and state the evidence gap.
- Tier 3-4 (engineering quality) findings do NOT affect the submission verdict.
- Don't manufacture findings. Signal > noise.
- Don't use AI slop, hedges, or emojis.
```

**Team mode** — you don't build one big prompt; the team-lead manual's Step 5 template governs per-reviewer prompts. Your Step 2 context above feeds the manual's workflow.

## Step 4: Dispatch

**Standard mode** — use the Agent tool:
- `subagent_type`: `"ios-code-review:senior-ios-reviewer"` (safe via plugin namespace — the reviewer declares no Agent tool, so nothing is stripped)
- `description`: `"Senior iOS review of N files (mode: <mode>)"`
- `model`: `"fable"`
- `prompt`: the prompt constructed in Step 3
- Run in **foreground** (do NOT use `run_in_background: true`)

**Team mode** — YOU act as the team lead. Do NOT dispatch a subagent as team lead (plugin-namespaced subagents lose the Agent tool at runtime; you already have it).

1. **Load the team lead's operating manual** — read everything after the frontmatter (the second `---`) of `agents/ios-team-lead.md`, resolved in this order:
   - `${CLAUDE_PLUGIN_ROOT}/agents/ios-team-lead.md` (the plugin's own install root — preferred)
   - If that variable is unset in your context: Glob your Claude Code plugin cache for `**/ios-senior-review/agents/ios-team-lead.md` (or `**/ios-code-review*/agents/ios-team-lead.md`) and take the newest match
   - Last resort: the `agents/` directory of wherever this plugin repository was cloned
   Never assume one hardcoded absolute path — installs and dev checkouts live in different places.
2. **Execute the manual's Steps 1-10** as written: map + partition (show the partition table and seam map to the user BEFORE dispatching), gather prior learnings once if a source exists, TodoWrite, dispatch the reviewer wave (all dispatches batched in one message, ≤10; `model: "fable"`; every prompt carries a `BLACKBOARD:` line), collect from blackboards with the per-agent validation gate, dispatch the single Runtime Verification agent, do the seam review yourself, consolidate with normalized verdicts, present the unified report.

Do NOT run anything in the background — the user wants to see reviewers complete in real time. A team review typically takes ~15-30 minutes (one parallel wave + runtime pass + consolidation); it only stretches toward 20-100 minutes under the sequential fallback after an observed session-reset.

## Step 5: Present results

When the agent returns (standard mode) or you finish consolidation (team mode):

1. **Read the blackboard file, not the truncated final message.** The blackboard is the report of record. Quick validation gate before presenting: the file exists and is substantive, and 2-3 spot-checked file:line citations match the actual code. If a citation doesn't hold up, say so next to that finding rather than silently presenting it.

2. Display the report verbatim from the blackboard. Do not summarize, condense, or reformat. The user wants the raw output including both summary tables and both verdicts.
   - In team mode, the report includes the team partition table, consolidated tables, all findings with reporter attribution, runtime verification results, seam findings, pattern findings, and unified verdicts.

3. After the report, prompt:

   ```
   Apply any of these findings? Tell me which:
   - "all [R] and [R?]" / "all rejection-grade"
   - "all submission findings" / "all engineering findings"
   - "finding 3 and 7"
   - "everything in <filename>"
   - "skip" if you want to handle it yourself
   ```

4. **Do NOT auto-apply findings.** The user explicitly chooses. Reviewer findings are advisory.

5. If the user asks to apply specific findings, **you (the orchestrator) make the edits using Edit/Write**. Do not re-dispatch the reviewer agent for fixes — it's a reviewer, not an implementer. Apply each requested finding's "Suggested fix" to the indicated file:line. Be careful with `.plist`, `.entitlements`, `.xcprivacy` files — these are XML and need precise edits (and keep `.xcprivacy` free of XML comments).

6. After applying any fixes, offer:
   - Re-run the review on the same scope to verify nothing regressed
   - Run `swiftlint` to confirm style fixes are clean
   - Run `xcodebuild build` to confirm the project still compiles

## Notes on agent behavior

- Both agents are fresh-context, pinned `model: fable`. They do NOT see this conversation. Everything they need goes in the dispatch prompt.
- `senior-ios-reviewer` has Read, Grep, Glob, Bash, XcodeBuildMCP (simulator verification, when configured), WebSearch, WebFetch, TodoWrite. It does NOT have Edit/Write/Agent — by design; it writes its blackboard report via Bash heredoc.
- The team-lead role is played by YOU with your own session tools; `agents/ios-team-lead.md` is its manual, not a dispatch target.
- Both modes run BOTH review modes by default: App Review Simulation + Senior Engineering Review. Pass `--mode submission` or `--mode engineering` to limit.
- Reviewer sub-agents in team mode are dispatched in one parallel wave (≤10, batched in a single message); halved sequential waves only if a session-reset/rate-limit actually occurs.
- If the user has a local iOS knowledge base or notes directory, pass its path in the prompt — the reviewer will cite it alongside the canonical Apple sources. Without one, the review works entirely from canonical sources.
- Submission verdict is independent of engineering verdict. An app can be `READY` for submission and `NEEDS WORK` for engineering — those are different concerns.
- **Team vs standard mode trade-off:** Team mode gives more thorough dimension coverage plus the seam review and a dedicated runtime pass, at ~15-30 min wall clock. Standard mode is faster (5-15 min) and usually sufficient for codebases under ~80 Swift files.

## When to skip parts of the workflow

- **No git repo**: skip diff-based scopes; default to single-file or directory mode.
- **No Xcode project / Package.swift**: tell the user; dispatch the agent anyway with file-level scope. Findings will be SOURCE-only.
- **No swiftlint / periphery installed**: skip those steps; tell the agent in the prompt that they're unavailable.
- **No simulator** (CI, sandboxed, or first-launch-locked Xcode): tell the agent; RUNTIME checks stay `[R?]` with the gap named. Never let it skip silently.
- **No App Store Connect access**: tell the agent; submission verdict will be limited to SOURCE-derivable findings. The agent will not assert `[R]` for ASC-only findings.

## Anti-patterns to avoid

- Don't dispatch the agent without project context — Apple-specific findings need entitlements/plist/manifest/scheme data.
- Don't dispatch on diff scope for an App Store submission review — Apple sees the whole app, you should too.
- Don't dispatch team mode on narrow scope — expand to `all` per Step 1's team scope handling.
- Don't run the agent in the background — iOS reviews are long, the user wants progress.
- Don't dispatch reviewers before showing the partition plan (team mode order is map → partition → show plan → dispatch).
- Don't dispatch team mode on tiny codebases (< 30 Swift files) — it aborts; use standard review.
- Don't dispatch a team-lead subagent — you ARE the team lead in team mode.
- Don't trust a truncated final message when a blackboard exists — read the file.
- Don't auto-apply findings — wait for the user's explicit selection.
- Don't mix submission and engineering findings into a single verdict — they're separately scored on purpose.
- Don't dispatch this skill recursively.
