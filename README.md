# ios-code-review

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Plugin Version](https://img.shields.io/badge/version-0.3.0-blue.svg)](https://github.com/TheMizeGuy/ios-code-review/releases)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-8A2BE2.svg)](https://claude.com/claude-code)
[![Model](https://img.shields.io/badge/model-Fable%205-orange.svg)](https://www.anthropic.com/claude)
[![Platform](https://img.shields.io/badge/platform-iOS%20%7C%20iPadOS%20%7C%20watchOS%20%7C%20tvOS%20%7C%20visionOS-lightgrey.svg)](https://developer.apple.com)

A [Claude Code](https://claude.com/claude-code) plugin that dispatches a **Fable 5** senior iOS developer agent to review your Swift / SwiftUI / UIKit code. The agent simulates **both** the Apple App Review team **and** a senior Apple platform engineer, producing two independent verdicts — and verifies runtime behavior on a real simulator via [XcodeBuildMCP](https://xcodebuildmcp.com) when available.

Two dispatch modes:

- **Standard** (default) — single `senior-ios-reviewer` agent. 5-15 min. Best for diffs, features, or codebases under ~80 Swift files.
- **Team** (requires the exact phrase "ios team review" or `--team`) — the orchestrator acts as team lead per the `ios-team-lead` manual: maps the codebase, partitions it into 4-10 non-overlapping scopes, dispatches `senior-ios-reviewer` sub-agents in one parallel wave, adds a dedicated runtime-verification agent and a mandatory seam review at the partition boundaries, and consolidates findings into one unified report with single submission + engineering verdicts. ~15-30 min. Best for whole-project audits, multi-target apps, and pre-submission reviews (30+ Swift files; 100+ is the sweet spot).

The reviewer is a fresh-context subagent with strict **read-only** tool access. Findings come back evidence-tagged with specific guideline numbers, concrete Swift rewrites, and citations. The orchestrator presents the report and asks which findings to apply — nothing is auto-fixed without your explicit selection.

> **Note — this repo previously shipped a standalone skill (`SKILL.md`).** That skill was replaced on 2026-04-14 with a proper plugin containing an orchestrator skill and a dedicated Fable 5 subagent. The plugin is a strict upgrade: fresh-context reviewer, forced Fable 5 model, read-only tool access. Old `SKILL.md` consumers should switch to the plugin via the install instructions below.

## What it does

When you invoke the `review-ios` skill (or ask Claude to review your iOS app), the plugin:

1. **Scope resolution** — single file, directory, git diff (including untracked new files), staged, PR diff, or whole project. For App Store submission reviews, `diff` auto-expands to `all` (Apple sees the whole app, not your diff).
2. **Apple-specific context gathering** — `Info.plist`, `*.entitlements`, `PrivacyInfo.xcprivacy`, build settings (deployment target, Swift version/language mode), shared-scheme diagnostics (TSan/ASan), targets enumerated by `productType` (App Clip, NSE, widgets, watchOS), third-party deps, linter config, simulator availability.
3. **Agent dispatch** — fresh-context Fable 5 subagent with read-only tools, running in two simultaneous modes:
   - **App Review Simulation** — Apple App Review team persona, checks all 5 guideline categories, top rejection causes with real-world frequencies
   - **Senior Engineering Review** — senior Apple platform engineer, checks Swift quality, Swift 6 concurrency, performance, HIG conformance, accessibility, platform integration
4. **The agent** reads code, runs `swiftlint`/`periphery`/`xcodebuild analyze` if available, drives a simulator pass (build, UI tests, screenshots at default + `.accessibility3` Dynamic Type) when one is available, writes its full report to a durable blackboard file, and returns evidence-tagged findings
5. **Present results** — the orchestrator displays the verbatim report with **both** summary tables and **both** verdicts, then asks which findings you want applied

## Installation

```bash
# 1. Add this repo as a marketplace
claude plugin marketplace add https://github.com/TheMizeGuy/ios-code-review.git

# 2. Install the plugin
claude plugin install ios-senior-review@ios-code-review

# 3. Restart Claude Code for the plugin to load
```

After restart, verify with `claude plugin list` and look for `ios-senior-review@ios-code-review`.

## Usage

| Invocation | What it reviews |
|---|---|
| `/ios-code-review:review-ios` | Uncommitted + staged + new (untracked) Swift changes (default), both review modes |
| `/ios-code-review:review-ios all` | Whole project, both modes (recommended for pre-submission) |
| `/ios-code-review:review-ios all --mode submission` | Whole project, App Review Simulation only |
| `/ios-code-review:review-ios all --mode engineering` | Whole project, Senior Engineering Review only |
| `/ios-code-review:review-ios staged` | Only staged changes |
| `/ios-code-review:review-ios pr` | Diff vs the default branch (main/master, local or origin; stops and asks if none exists) |
| `/ios-code-review:review-ios MyApp/Auth/` | All Swift files in `MyApp/Auth/` |
| `/ios-code-review:review-ios MyApp/Auth/LoginView.swift` | Single file |

### Team mode (explicit opt-in required)

| Invocation | What it does |
|---|---|
| `/ios-code-review:review-ios all --team` | Team review of the whole project (team lead picks 4-10 agents) |
| `/ios-code-review:review-ios all --team --mode submission` | Team review, submission mode only |
| `/ios-code-review:review-ios all --team --mode engineering` | Team review, engineering mode only |

Natural-language trigger: the exact phrase **"ios team review"** (case-insensitive). Examples:
- "do an ios team review"
- "ios team review before I submit"
- "run an iOS team review on this project"

Generic phrases like "team review", "full audit", "thorough review", or "do a team review of my iOS app" do NOT activate team mode — they default to standard. This avoids accidentally dispatching a long-running multi-agent review when the user wanted a thorough single-agent one.

You can also ask Claude in plain English: "review my iOS app", "will Apple reject this?", "audit my Swift code", "TestFlight rejected my build, what's wrong?". The skill description triggers automatically.

## The 12 review dimensions across 4 tiers

### Tier 1 — App Review Blockers (must pass for submission)

| # | Dimension | Examples |
|---|---|---|
| 1 | App Store Rejection Risk | Top 10 rejection causes with frequencies (Guidelines 1-5), current SDK/toolchain gate, age-rating questionnaire, Sign in with Apple, UIWebView, IAP + StoreKit 2 lifecycle, AI data-sharing disclosure (5.1.2(i)), UGC/anonymous chat (1.2), Live Activities anti-spam (4.5.3), 4.3(b) saturated categories, EU DMA |
| 2 | Privacy & Data Protection | Privacy manifest + full Required Reason API code table (incl. the App-Group `1C8F.1` gotcha), ATT timing/UX, purpose strings, tracking domains, third-party SDK manifests, policy-vs-wire drift check, account deletion, GDPR/CCPA |
| 3 | Entitlements & Info.plist | Entitlement-to-feature fit, export compliance (`ITSAppUsesNonExemptEncryption`), code-signing diagnostics (ITMS-90034/90046), arm64/binary-size upload gates, `UIBackgroundModes`, `CFBundleURLTypes`, scene manifest |
| 4 | Security | Keychain, ATS, certificate pinning, screenshot protection, OSLog `.private`, biometric auth, hardcoded secrets |

### Tier 2 — High-Yield Static Quality (reviewer flags)

| # | Dimension | Examples |
|---|---|---|
| 5 | Human Interface Guidelines | NavigationStack, Liquid Glass adoption + legibility (iOS 26), semantic colors, SF Symbols, alerts, layout direction, app icon (dark + tinted variants, Icon Composer), launch screen, localization |
| 6 | Accessibility | VoiceOver (incl. `.onTapGesture` invisibility, `.combine` swallowing interactive children), Dynamic Type, contrast, touch targets, Reduce Motion, Voice Control, `performAccessibilityAudit` integrity, Accessibility Nutrition Labels, WCAG 2.1 AA + 2.2 |
| 7 | SwiftUI / UIKit Patterns | `@Observable`, `NavigationStack`, `.task {}`, `LazyVStack`, `#Preview`, `UIViewRepresentable` cleanup |
| 8 | Deep Linking & Extensions | Universal links, AASA, `.onOpenURL`, App Clip budget, NSE, widgets, share/action extensions |

### Tier 3 — Engineering Quality (does not affect submission verdict)

| # | Dimension | Examples |
|---|---|---|
| 9 | Swift Language Quality | API design, no force-unwraps, error handling, value vs reference, access control, deprecated APIs |
| 10 | Concurrency Safety | `@MainActor`, `Sendable`, structured concurrency, cancellation, Swift 6 language mode / `SWIFT_STRICT_CONCURRENCY`, TSan (scheme-level) |
| 11 | Performance & Memory | Launch time, retain cycles, view perf, image handling, BGTaskScheduler, app size |

### Tier 4 — Platform Opportunity (advisory only)

| # | Dimension | Examples |
|---|---|---|
| 12 | Platform Integration Depth | Widgets, Spotlight, App Intents, Haptics, Live Activities, Quick Actions, Context Menus, Drag & Drop, TipKit |

## Evidence classes

Every finding states its evidence basis. The agent will **not** issue `[R]` (rejection) from `RUNTIME` or `ASC` evidence — it uses `[R?]` (marked "verified" with reproduction steps when the simulator pass confirmed it) and explicitly states what verification is needed.

| Class | What it proves |
|---|---|
| `SOURCE` | Provable from reading code (force-unwraps, missing privacy manifest, hardcoded secrets) |
| `BUILD` | Requires compiled artifact (entitlement signing, binary scan, app size, SDK version gate) |
| `SERVER` | Requires external system (AASA validation, push payload, backend) |
| `ASC` | Requires App Store Connect data (privacy labels, age rating, screenshots, metadata) |
| `RUNTIME` | Requires device/simulator (crashes, safe areas, contrast, haptics, Dynamic Type layout) — marked `RUNTIME (verified)` when the reviewer reproduced it on the simulator |

## Severity levels

| Tag | Meaning |
|---|---|
| `[R]` | REJECTION — Will cause App Store rejection. Must fix. SOURCE/BUILD evidence only. |
| `[R?]` | LIKELY REJECTION — Probable rejection but needs verification (RUNTIME or ASC). |
| `[W]` | WARNING — Significant quality risk or likely reviewer flag. |
| `[~]` | RECOMMENDATION — Best practice gap. Improves quality. |
| `[+]` | PRAISE — Well-implemented pattern worth highlighting. |

## Verdicts

The agent produces **two independent verdicts**:

**Submission Verdict:**
- **READY** — 0 `[R]`, 0 `[R?]`, 0-2 `[W]`
- **LIKELY READY** — 0 `[R]`, 1-2 `[R?]`, 0-2 `[W]`
- **FIX BEFORE SUBMITTING** — 0 `[R]`, 3+ `[W]` or 3+ `[R?]`
- **WILL BE REJECTED** — 1+ `[R]` confirmed from `SOURCE`/`BUILD`

**Engineering Verdict:**
- **STRONG** — 0-2 `[W]`, good `[+]` coverage
- **ACCEPTABLE** — 3-5 `[W]`, no critical patterns
- **NEEDS WORK** — 6+ `[W]` or fundamental patterns missing

An app can be `READY` for submission AND `NEEDS WORK` for engineering — those are different concerns and the agent keeps them separate on purpose. Tier 3-4 findings never affect the submission verdict.

**Team-mode aggregation:** `[R]` and `[R?]` are absolute (rejection risks don't dilute with project size). `[W]` thresholds apply to a per-100-files normalized count above 100 files, and pattern findings (the same issue across many files) count once — so ten independently-clean scopes can't sum their way into a failing verdict on scattered one-off warnings. The report shows both the raw total and the normalized figure.

## Components

| Type | Name | Purpose |
|---|---|---|
| Skill | `review-ios` | User-invoked entry point; gathers Apple-specific scope and either dispatches the standard reviewer or acts as team lead per the `ios-team-lead` manual |
| Agent | `senior-ios-reviewer` | Fable 5 reviewer that runs both modes, reads code + project artifacts, runs static tooling and the simulator verification pass, writes its report to a blackboard file, returns findings. Used directly in standard mode and as the sub-agent in team mode |
| Agent | `ios-team-lead` | Team-lead operating manual. Read and executed by the orchestrator in team mode — never dispatched as a subagent (plugin-namespaced dispatch strips the Agent tool at runtime). Covers partitioning, the parallel-wave dispatch contract, the runtime-verification pass, the mandatory seam review, and consolidation with normalized verdicts |

## Tool access

The agent is **read-only by design**. It has:

| Tool | Purpose |
|---|---|
| `Read`, `Grep`, `Glob` | Read Swift files, Info.plist, entitlements, privacy manifest, schemes |
| `Bash` | Run `swiftlint`, `periphery`, `xcodebuild analyze`, `xcrun simctl`; write the blackboard report via heredoc |
| `TodoWrite` | Track findings during long reviews |
| `WebSearch`, `WebFetch` | Check latest Apple guideline updates |
| `mcp__XcodeBuildMCP__*` (optional) | The runtime verification pass — simulator build/run/test/screenshot/snapshot — when [XcodeBuildMCP](https://xcodebuildmcp.com) is configured |

It does **not** have `Edit`, `Write`, or `Agent` access (the blackboard heredoc via Bash is its one sanctioned file output). Findings are advisory — the orchestrator (your main Claude session) applies them based on your selection.

## Optional enhancements

The plugin works fine with just the built-in tools, but the review gets stronger with:

- **[XcodeBuildMCP](https://xcodebuildmcp.com)** — enables the full simulator runtime-verification pass (build/run/test/screenshot/snapshot). Without it the agent falls back to `xcodebuild`/`xcrun simctl` via Bash, or static-only review.
- **[serena](https://github.com/oraios/serena) MCP** — symbol-level project navigation. Much faster than grepping for structural understanding.
- **[Context7](https://context7.com/) MCP** — live Apple framework docs and third-party library docs.
- **[GoodMem](https://goodmem.ai/) MCP** — semantic memory search for cross-session learnings.
- **A local iOS knowledge base** — any directory of reference docs (Obsidian vault, markdown repo, WWDC notes). Tell the agent where it is in the dispatch prompt and it will read + cite relevant files.

None are required.

## Why a plugin instead of a skill?

This repo previously shipped a standalone `SKILL.md` (469 lines loaded inline into your conversation). The plugin architecture is a strict upgrade:

1. **Fresh context** — the reviewer runs in an isolated subagent that has never seen the conversation that wrote the code. No pattern blindness.
2. **Forced model** — the agent is pinned to `model: fable`. Your main session can be on any model; the reviewer is always Fable 5.
3. **Read-only enforcement** — the agent has Read but not Edit/Write. Impossible to "accidentally fix" code mid-review.
4. **Clean orchestration** — the skill gathers project context (Info.plist, entitlements, targets, privacy manifest) and passes it to the agent. The old skill expected the main agent to do all of this itself.

## License

MIT. See [LICENSE](LICENSE).

## Credits

Built by [TheMizeGuy](https://github.com/TheMizeGuy). Backed by the [Claude Code](https://claude.com/claude-code) plugin system and Anthropic's Fable 5 model.
