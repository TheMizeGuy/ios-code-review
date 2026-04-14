---
name: review-ios
description: |-
  Use this skill when the user asks for an iOS / Swift / SwiftUI / UIKit code review, says "review my iOS app", "check my Swift code", "audit this for App Store readiness", "will Apple reject this?", "TestFlight review", "pre-submission review", "iOS engineering review", or wants comprehensive review covering App Store compliance, privacy, entitlements, security, HIG, accessibility, SwiftUI patterns, deep linking, concurrency, or performance. Supports two dispatch modes: STANDARD (default — single Opus 4.6 `senior-ios-reviewer` agent) and TEAM (requires the explicit phrase "ios team review" OR the `--team` flag — dispatches `ios-team-lead` which maps the codebase, partitions it into 4-10 non-overlapping scopes, dispatches a team of `senior-ios-reviewer` sub-agents sequentially, and consolidates findings into one unified report). Team mode ONLY triggers on the specific phrase "ios team review" — generic "team review" or "full audit" alone do NOT activate team mode. Use proactively before App Store submission, after a TestFlight rejection, or when finishing a substantial iOS feature.
argument-hint: '[path | file | "diff" | "staged" | "pr" | "all"] [--mode submission | --mode engineering] [--team]'
allowed-tools: Bash, Read, Grep, Glob, TodoWrite, Agent
---

# iOS Senior Review

You are coordinating a senior iOS code review on the user's behalf. Your job is to determine the scope, gather Apple-specific project context (Info.plist, entitlements, privacy manifest, build settings, deployment targets), and dispatch one of two Opus 4.6 agents:

- **Standard review** (default) — dispatch `ios-senior-review:senior-ios-reviewer` directly. Single-agent review, fastest, best for small-to-medium codebases or narrow scopes.
- **Team review** (`--team` flag) — dispatch `ios-senior-review:ios-team-lead`. The team lead maps the codebase, decides on 4-10 reviewers, partitions scope, dispatches `senior-ios-reviewer` sub-agents sequentially, and consolidates findings. Best for large codebases (50+ Swift files), multi-target projects, or pre-submission audits where you want maximum dimension coverage.

Both modes run BOTH App Review Simulation + Senior Engineering Review by default, controllable via `--mode`.

## Step 1: Parse arguments and detect team trigger

The user passed an argument string (may be empty). Three parts:

1. **Scope** — first non-flag token (path, file, `diff`, `staged`, `pr`, `all`)
2. **Mode flag** — optional `--mode submission` or `--mode engineering`. Default: both.
3. **Team flag** — optional `--team`. Default: standard (single-agent). When present, dispatches `ios-team-lead` instead of `senior-ios-reviewer`.

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

**Rationale:** "team review" is a generic phrase that could mean many things. Team mode requires the more explicit phrase "ios team review" to avoid accidentally dispatching a long-running multi-agent review when the user only wanted a thorough single-agent review. The `--team` flag remains available as the explicit opt-in.

**If you're unsure whether the user wants team mode** (e.g., they said "thorough review of my iOS app"), default to standard mode and mention team mode as an option: "Running standard review. If you want team mode (4-10 agents, 20-100 min), say 'ios team review' or add `--team`."

### Scope resolution

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

If it's **>100 files**: tell the user the count. In STANDARD mode, ask whether to continue (iOS reviews benefit from full project context but cost more). In TEAM mode, 100+ files is the sweet spot — proceed.

**Team mode scope handling:**
- Team mode always uses a broad scope. If the user passed `diff` or `staged` with `--team`, expand to `all` automatically and tell the user (team review of just a diff defeats the purpose — it's for whole-project audits).
- If the user passed `--team` with a codebase under 30 Swift files, tell the user team mode is overkill and offer to run standard review instead. Ask before dispatching.

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

Build a single self-contained prompt for the agent. Zero conversation context inheritance — bake everything in. The prompt structure is the same for standard and team modes; the difference is which `subagent_type` you dispatch and how the SCOPE field is filled.

**Standard mode** — pass the resolved file list directly:

```
SCOPE — review these files:
<absolute path 1>
<absolute path 2>
...
```

**Team mode** — pass the project root; the team lead decides partitioning. The SCOPE field becomes:

```
SCOPE — full-project team review. Map and partition internally.
PROJECT ROOT: <absolute path>
(The team lead should dispatch 4-10 sub-agents based on project size and structure.)
```

Full prompt template (both modes share everything below SCOPE):

```
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

MODE: <both | submission | engineering>

TASK (standard mode — single agent):
1. Read every file in scope completely.
2. Run the tooling if available:
   - swiftlint lint --reporter json (capture findings)
   - periphery scan --format json (dead code)
   - xcodebuild analyze (if scheme can be determined and build is feasible)
3. Review across all 12 dimensions per your system prompt. Run BOTH modes unless --mode passed.
4. Return findings in the strict format from your system prompt: tag, evidence class, file:line, current code, suggested fix, authoritative citation.
5. Both summary tables, both verdicts, recommended next steps, raw tooling output.

TASK (team mode — team lead):
1. Map the full codebase structure and count files per target/module.
2. Decide agent count (4-10) based on your system prompt's sizing table.
3. Partition the codebase into non-overlapping scopes.
4. Show the partition plan to the user before dispatching.
5. Dispatch `senior-ios-reviewer` sub-agents SEQUENTIALLY (one at a time — never parallel).
6. Collect and deduplicate findings.
7. Produce a single consolidated report with unified tables and verdicts.

CONSTRAINTS:
- Read-only review. Do NOT modify any files.
- Cite authoritative sources (App Review Guideline numbers, HIG sections, WCAG criteria) in every finding where applicable.
- Cite Apple guideline numbers for App Review findings.
- Don't issue [R] from RUNTIME / ASC evidence — use [R?] and state the evidence gap.
- Tier 3-4 (engineering quality) findings do NOT affect the submission verdict.
- Don't manufacture findings. Signal > noise.
- Don't use fluff, hedges, or emojis.
```

## Step 4: Dispatch

**Standard mode** — use the Agent tool:
- `subagent_type`: `"ios-senior-review:senior-ios-reviewer"`
- `description`: `"Senior iOS review of N files (mode: <mode>)"`
- `prompt`: the prompt constructed in Step 3
- Run in **foreground** (do NOT use `run_in_background: true`)

**Team mode** — YOU act as the team lead. Do NOT dispatch a subagent as team lead.

**Why no team-lead subagent**: Subagents do NOT reliably receive the Agent tool at runtime (confirmed Claude Code platform limitation). The team lead's job is to dispatch 4-10 senior-ios-reviewer sub-agents -- a subagent can't do that. You (the main agent running this skill) DO have the Agent tool, so reviewer dispatches work from here.

Team mode workflow (you execute all of it):

1. **Load the team lead's operating manual**: Read the plugin's `ios-senior-review/agents/ios-team-lead.md` (everything after the second `---` frontmatter delimiter). This is your manual for partitioning, dispatch rules, and consolidation format.

2. **Map + partition the codebase** per the manual's Steps 1-3:
   - Run the codebase mapping commands (file counts, target enumeration, framework detection)
   - Apply the partitioning rules based on Swift file count
   - Decide on 4-10 scopes, each non-overlapping
   - Present the partition table to the user BEFORE dispatching anything

3. **Dispatch senior-ios-reviewer sub-agents sequentially** (one at a time, NEVER parallel). For each scope:
   - Build the scope-specific reviewer prompt per the manual's Step 5 template
   - `Agent({ subagent_type: "ios-senior-review:senior-ios-reviewer", description: "Scope <N>: <name>", model: "opus", prompt: "<scope prompt>" })`
   - Wait for the reviewer to return before dispatching the next
   - Collect each reviewer's report

   **EXECUTION RULE — override Claude Code's default parallel-tool-call bias for this step.** Your system prompt tells you to batch independent tool calls in one `function_calls` block. That rule does NOT apply here — the reviewer dispatches look independent (non-overlapping scopes, self-contained prompts), but batching 3+ of them in one assistant turn triggers session reset on the Max plan. Emit **exactly ONE Agent tool call per assistant turn** during reviewer dispatch. Never put two Agent calls in the same `function_calls` block. If you find yourself composing multiple dispatches together, STOP and split across turns. Update TodoWrite between each dispatch — that gap is the load-bearing serializer.

4. **Consolidate the reports** per the manual's consolidation section:
   - Deduplicate findings across boundaries
   - Build the unified tables with reporter attribution
   - Produce BOTH verdicts (submission + engineering) for the whole project
   - Present the unified report verbatim to the user

Do NOT run anything in the background. Team reviews take 20-100 minutes and the user wants to see each sub-agent complete in real time.

## Step 5: Present results

When the agent returns:

1. Display the agent's full report verbatim. Do not summarize, condense, or reformat. The user wants the raw output including both summary tables and both verdicts.
   - In team mode, the report includes the team partition table, consolidated tables, all findings with reporter attribution, pattern findings, and unified verdicts.

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

## Notes on agent behavior

- Both agents are fresh-context Opus 4.6. They do NOT see this conversation. Everything they need goes in the dispatch prompt.
- `senior-ios-reviewer` has Read, Grep, Glob, Bash, WebSearch, WebFetch, TodoWrite, plus optional serena/Context7/GoodMem MCP tools if installed. It does NOT have Edit/Write/Agent — by design.
- `ios-team-lead` has the same tools PLUS the Agent tool so it can dispatch sub-agents. It also does NOT have Edit/Write — by design.
- Both modes run BOTH review modes by default: App Review Simulation + Senior Engineering Review. Pass `--mode submission` or `--mode engineering` to limit.
- The team lead dispatches sub-agents SEQUENTIALLY (one at a time). Rate limits can cause session resets at 3+ parallel Agent calls.
- Submission verdict is independent of engineering verdict. An app can be `READY` for submission and `NEEDS WORK` for engineering — those are different concerns.
- **Team vs standard mode trade-off:** Team mode gives more thorough dimension coverage on large codebases at the cost of longer runtime (4-10 × 5-10 min sequential = 20-100 minutes total). Standard mode is faster and usually sufficient for codebases under ~80 Swift files.

## When to skip parts of the workflow

- **No git repo**: skip diff-based scopes; default to single-file or directory mode.
- **No Xcode project / Package.swift**: tell the user; dispatch the agent anyway with file-level scope. Findings will be SOURCE-only.
- **No swiftlint / periphery installed**: skip those steps; tell the agent in the prompt that they're unavailable.
- **No App Store Connect access**: tell the agent; submission verdict will be limited to SOURCE-derivable findings. The agent will not assert `[R]` for ASC-only findings.
- **CI / sandboxed environment**: pass that context; no `xcodebuild`, no `swiftlint`, just static review.

## Anti-patterns to avoid

- Don't dispatch the agent without project context — Apple-specific findings need entitlements/plist/manifest data.
- Don't dispatch on diff scope for an App Store submission review — Apple sees the whole app, you should too.
- Don't dispatch team mode on diff/staged scope — team mode is for whole-project audits. Expand to `all` or fall back to standard review.
- Don't run the agent in the background — iOS reviews are long, the user wants progress.
- Don't dispatch team mode on tiny codebases (< 30 Swift files) — the overhead isn't worth it. Use standard review instead.
- Don't dispatch team lead AND standard reviewer for the same review — pick one. Team mode already includes standard reviewers as sub-agents.
- Don't try to dispatch `senior-ios-reviewer` sub-agents yourself when in team mode — the team lead does all sub-agent dispatch internally.
- Don't summarize the agent's findings — show them verbatim with both tables and both verdicts.
- Don't auto-apply findings — wait for the user's explicit selection.
- Don't mix submission and engineering findings into a single verdict — they're separately scored on purpose.
- Don't dispatch this skill recursively.
