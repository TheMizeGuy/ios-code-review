# ios-code-review

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Plugin Version](https://img.shields.io/badge/version-0.1.0-blue.svg)](https://github.com/TheMizeGuy/ios-code-review/releases)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-8A2BE2.svg)](https://claude.com/claude-code)
[![Model](https://img.shields.io/badge/model-Opus%204.6-orange.svg)](https://www.anthropic.com/claude)
[![Platform](https://img.shields.io/badge/platform-iOS%20%7C%20iPadOS%20%7C%20watchOS%20%7C%20tvOS%20%7C%20visionOS-lightgrey.svg)](https://developer.apple.com)

A [Claude Code](https://claude.com/claude-code) plugin that dispatches an **Opus 4.6** senior iOS developer agent to review your Swift / SwiftUI / UIKit code. The agent simulates **both** the Apple App Review team **and** a senior Apple platform engineer, producing two independent verdicts.

The reviewer is a fresh-context subagent with strict **read-only** tool access. Findings come back evidence-tagged with specific guideline numbers, concrete Swift rewrites, and citations. The orchestrator presents the report and asks which findings to apply — nothing is auto-fixed without your explicit selection.

> **Note — this repo previously shipped a standalone skill (`SKILL.md`).** That skill was replaced on 2026-04-14 with a proper plugin containing an orchestrator skill and a dedicated Opus 4.6 subagent. The plugin is a strict upgrade: fresh-context reviewer, forced Opus 4.6 model, read-only tool access. Old `SKILL.md` consumers should switch to the plugin via the install instructions below.

## What it does

When you invoke the `review-ios` skill (or ask Claude to review your iOS app), the plugin:

1. **Scope resolution** — single file, directory, git diff, staged, PR diff, or whole project. For App Store submission reviews, `diff` auto-expands to `all` (Apple sees the whole app, not your diff).
2. **Apple-specific context gathering** — `Info.plist`, `*.entitlements`, `PrivacyInfo.xcprivacy`, build settings (deployment target, Swift version, strict concurrency), targets (App Clip, NSE, widgets, watchOS), third-party deps, linter config.
3. **Agent dispatch** — fresh-context Opus 4.6 subagent with read-only tools, running in two simultaneous modes:
   - **App Review Simulation** — Apple App Review team persona, checks all 5 guideline categories, top 10 rejection causes
   - **Senior Engineering Review** — senior Apple platform engineer, checks Swift quality, Swift 6 concurrency, performance, HIG conformance, accessibility, platform integration
4. **The agent** reads code, runs `swiftlint`/`periphery`/`xcodebuild analyze` if available, reviews across 12 dimensions in 4 tiers, and returns evidence-tagged findings
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
| `/ios-code-review:review-ios` | Uncommitted + staged Swift changes (default), both review modes |
| `/ios-code-review:review-ios all` | Whole project, both modes (recommended for pre-submission) |
| `/ios-code-review:review-ios all --mode submission` | Whole project, App Review Simulation only |
| `/ios-code-review:review-ios all --mode engineering` | Whole project, Senior Engineering Review only |
| `/ios-code-review:review-ios staged` | Only staged changes |
| `/ios-code-review:review-ios pr` | Diff vs `main`/`master` |
| `/ios-code-review:review-ios MyApp/Auth/` | All Swift files in `MyApp/Auth/` |
| `/ios-code-review:review-ios MyApp/Auth/LoginView.swift` | Single file |

You can also ask Claude in plain English: "review my iOS app", "will Apple reject this?", "audit my Swift code", "TestFlight rejected my build, what's wrong?". The skill description triggers automatically.

## The 12 review dimensions across 4 tiers

### Tier 1 — App Review Blockers (must pass for submission)

| # | Dimension | Examples |
|---|---|---|
| 1 | App Store Rejection Risk | Top 10 rejection causes (Guidelines 1-5), Sign in with Apple, UIWebView, IAP, AI consent, EU DMA, StoreKit 2 |
| 2 | Privacy & Data Protection | Privacy manifest, ATT, purpose strings, tracking domains, third-party SDK manifests, account deletion, GDPR/CCPA |
| 3 | Entitlements & Info.plist | Entitlement-to-feature fit, `UIBackgroundModes`, `CFBundleURLTypes`, `LSApplicationQueriesSchemes`, scene manifest |
| 4 | Security | Keychain, ATS, certificate pinning, screenshot protection, OSLog `.private`, biometric auth, hardcoded secrets |

### Tier 2 — High-Yield Static Quality (reviewer flags)

| # | Dimension | Examples |
|---|---|---|
| 5 | Human Interface Guidelines | NavigationStack, semantic colors, SF Symbols, alerts, layout direction, app icon, launch screen, localization |
| 6 | Accessibility | VoiceOver, Dynamic Type, contrast, touch targets, Reduce Motion, Voice Control, WCAG 2.1 AA + 2.2 |
| 7 | SwiftUI / UIKit Patterns | `@Observable`, `NavigationStack`, `.task {}`, `LazyVStack`, `#Preview`, `UIViewRepresentable` cleanup |
| 8 | Deep Linking & Extensions | Universal links, AASA, `.onOpenURL`, App Clip budget, NSE, widgets, share/action extensions |

### Tier 3 — Engineering Quality (does not affect submission verdict)

| # | Dimension | Examples |
|---|---|---|
| 9 | Swift Language Quality | API design, no force-unwraps, error handling, value vs reference, access control, deprecated APIs |
| 10 | Concurrency Safety | `@MainActor`, `Sendable`, structured concurrency, cancellation, `SWIFT_STRICT_CONCURRENCY=complete`, TSan |
| 11 | Performance & Memory | Launch time, retain cycles, view perf, image handling, BGTaskScheduler, app size |

### Tier 4 — Platform Opportunity (advisory only)

| # | Dimension | Examples |
|---|---|---|
| 12 | Platform Integration Depth | Widgets, Spotlight, App Intents, Haptics, Live Activities, Quick Actions, Context Menus, Drag & Drop, TipKit |

## Evidence classes

Every finding states its evidence basis. The agent will **not** issue `[R]` (rejection) from `RUNTIME` or `ASC` evidence — it uses `[R?]` and explicitly states what verification is needed.

| Class | What it proves |
|---|---|
| `SOURCE` | Provable from reading code (force-unwraps, missing privacy manifest, hardcoded secrets) |
| `BUILD` | Requires compiled artifact (entitlement signing, binary scan, app size) |
| `SERVER` | Requires external system (AASA validation, push payload, backend) |
| `ASC` | Requires App Store Connect data (privacy labels, screenshots, metadata) |
| `RUNTIME` | Requires device/simulator (crashes, safe areas, contrast, haptics, Dynamic Type layout) |

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

## Components

| Type | Name | Purpose |
|---|---|---|
| Skill | `review-ios` | User-invoked entry point; gathers Apple-specific scope and dispatches the agent |
| Agent | `senior-ios-reviewer` | Opus 4.6 reviewer that runs both modes, reads code + project artifacts, runs tooling, returns findings |

## Tool access

The agent is **read-only by design**. It has:

| Tool | Purpose |
|---|---|
| `Read`, `Grep`, `Glob` | Read Swift files, Info.plist, entitlements, privacy manifest |
| `Bash` | Run `swiftlint`, `periphery`, `xcodebuild analyze` |
| `TodoWrite` | Track findings during long reviews |
| `WebSearch`, `WebFetch` | Check latest Apple guideline updates |
| `mcp__plugin_serena_serena__*` (optional) | Symbol-level project navigation if [serena](https://github.com/oraios/serena) is installed |
| `mcp__plugin_context7_context7__*` (optional) | Live Apple framework docs if [context7](https://context7.com/) is installed |
| `mcp__plugin_goodmem_goodmem__*` (optional) | Semantic memory queries if [GoodMem](https://goodmem.ai/) MCP is configured |

It does **not** have `Edit`, `Write`, or `Agent` access. Findings are advisory — the orchestrator (your main Claude session) applies them based on your selection.

## Optional enhancements

The plugin works fine with just the built-in tools, but findings get richer with:

- **[serena](https://github.com/oraios/serena) MCP** — symbol-level project navigation. Much faster than grepping for structural understanding.
- **[Context7](https://context7.com/) MCP** — live Apple framework docs and third-party library docs.
- **[GoodMem](https://goodmem.ai/) MCP** — semantic memory search for cross-session learnings.
- **A local iOS knowledge base** — e.g., an Obsidian vault under `~/Claude/vault/iOS Development/` with reference docs the agent can cite. Any path works — just tell the agent where it is.

None are required.

## Why a plugin instead of a skill?

This repo previously shipped a standalone `SKILL.md` (469 lines loaded inline into your conversation). The plugin architecture is a strict upgrade:

1. **Fresh context** — the reviewer runs in an isolated subagent that has never seen the conversation that wrote the code. No pattern blindness.
2. **Forced model** — the agent is pinned to `model: opus`. Your main session can be on any model; the reviewer is always Opus 4.6.
3. **Read-only enforcement** — the agent has Read but not Edit/Write. Impossible to "accidentally fix" code mid-review.
4. **Clean orchestration** — the skill gathers project context (Info.plist, entitlements, targets, privacy manifest) and passes it to the agent. The old skill expected the main agent to do all of this itself.

## License

MIT. See [LICENSE](LICENSE).

## Credits

Built by [mize](https://github.com/TheMizeGuy). Backed by the [Claude Code](https://claude.com/claude-code) plugin system and Anthropic's Opus 4.6 model.
