# Changelog

## [0.3.0] - 2026-07-03

### Added

- **Runtime verification pass** — the reviewer drives a real simulator via XcodeBuildMCP when available (build/run, UI tests incl. `performAccessibilityAudit`, screenshots at default + `.accessibility3` Dynamic Type, UI snapshots), verifying or refuting its `RUNTIME`-class `[R?]` findings. Verified reproductions are marked `RUNTIME (verified)`; `[R]` stays reserved for `SOURCE`/`BUILD` evidence. Falls back to `xcodebuild`/`xcrun simctl` via Bash, or static-only with the gap named.
- **Durable blackboard reports** — every dispatch carries a `BLACKBOARD:` path; the reviewer writes its full report there via Bash heredoc before returning (final messages truncate around 60KB), and the orchestrator reads the file with citation spot-checks rather than trusting a truncated message.
- **Mandatory seam review in team mode** — the team lead builds a seam map at partition time, reads every scope boundary from both sides after the wave returns, and resolves every cross-scope note instead of leaving it open; the Submission Artifacts agent additionally cross-checks bundled privacy-policy claims against the actual networking/auth wire code.
- **Dedicated runtime-verification agent in team mode** — exactly one agent drives the simulator after the static wave (parallel reviewers do not touch it), re-testing every flagged `RUNTIME` `[R?]`.
- 2025-2026 review-reality refresh: Required Reason API code table for all 5 categories (incl. the App-Group `1C8F.1` vs `CA92.1` ITMS-91056 gotcha), 5.1.2(i) third-party-AI disclosure specifics, 4.3(b) saturated/low-effort categories, 4.5.3 Live Activities anti-spam, new age-rating tiers (4+/9+/13+/16+/18+) and questionnaire gate, current SDK/toolchain upload gate, arm64/binary-size upload gates, code-signing ITMS diagnostics, EU DMA notarization + commission notes, Accessibility Nutrition Labels, Liquid Glass HIG checks (adoption, legibility, icon variants), StoreKit 2 lifecycle sub-checks, ATT timing/UX sub-checks, export compliance (`ITSAppUsesNonExemptEncryption`), placeholder-content/category checks, FamilyControls conditional block, `.onTapGesture`/`.combine` VoiceOver traps, `performAccessibilityAudit` handler-integrity check, Swift 6 language-mode check, top-10 rejection table with frequency estimates.
- Scope resolution hardening: `diff` now includes untracked new files (`git ls-files --others`); `pr` walks a 4-way merge-base ladder (main/master, local/origin) and stops instead of silently diffing worktree-vs-index; directory scope includes plist/entitlements/xcprivacy; explicit vendored-dir post-filtering; targets enumerated by `productType`; TSan/ASan read from `.xcscheme` (they are not build settings); argument validation for malformed flags.

### Changed

- **One dispatch story.** Team mode: the orchestrator itself acts as team lead, reading `agents/ios-team-lead.md` as an operating manual — a team-lead subagent is never dispatched (plugin-namespaced dispatch strips the Agent tool at runtime). Reviewer sub-agents go out in ONE parallel wave (≤10, batched in a single message); halved sequential waves only after an observed session-reset/rate-limit. All previous "sequential dispatch" language removed; team-mode wall clock is now ~15-30 min (20-100 min only under the sequential fallback).
- **Team verdict aggregation normalized** — `[R]`/`[R?]` remain absolute; `[W]` thresholds apply per 100 files above 100 files and pattern findings count once, so large projects aren't failed by scattered one-off warnings. The report prints both raw and normalized numbers.
- Team-mode floor documented consistently at 30+ Swift files (sweet spot 100+); sizing table clarifies target semantics and gains a mandatory-allocation bump rule and a numeric test-agent threshold.
- Corrected tooling recipes: `swiftlint analyze` two-step compile-log flow; full `xcodebuild analyze` invocation with `-project`/`-workspace` and `-destination`.
- SwiftLint/periphery/xcodebuild guidance, evidence classes, per-finding template, and tables updated throughout for the new runtime evidence flow.
- Plugin manifest and marketplace descriptions updated; author contact set to the public address.

## 2026-07-02

- Synced plugin content from the maintained source: Fable 5 model line throughout, agent fan-out bounded by the dispatch budget (≤10/wave), refreshed knowledge-base counts, removed stale references.

All notable changes to this repository will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - 2026-04-14

### Added

- **Team mode** — new `ios-team-lead` agent (Opus 4.6) for large-codebase reviews. When triggered, the team lead maps the codebase, decides on 4-10 reviewers, partitions the project into non-overlapping scopes, dispatches `senior-ios-reviewer` sub-agents sequentially, and consolidates all findings into a single unified report with one submission verdict and one engineering verdict. Best for codebases with 50+ Swift files, multi-target projects, or pre-App-Store-submission audits.
- `--team` flag on the `review-ios` skill and natural-language trigger via the exact phrase "ios team review" (case-insensitive). Generic phrases like "team review", "full audit", or "thorough review" do NOT activate team mode — this is intentional to avoid accidentally dispatching a 20-100 minute multi-agent review.
- Team composition table in the consolidated output (partition plan, file counts, per-agent finding totals).
- Pattern-finding grouping in team-mode output — when the same issue appears across multiple agents' scopes (e.g., force-unwraps in many files), consolidated into one entry with affected locations rather than duplicated per file.
- Cross-scope reference handling — agents flag when a file in their scope depends on a file in another agent's scope; team lead surfaces these in the final report.
- Plugin `license: "MIT"` field in `plugin.json`.

### Changed

- Reviewer agent description now explicitly frames authoritative sources (Apple App Review Guidelines, HIG, WCAG, Swift.org API Design Guidelines, WWDC sessions, Apple documentation) as the default citation targets. Local knowledge bases remain supported but optional — cited only if the orchestrator confirms one exists.
- MCP integrations (serena, Context7, GoodMem) explicitly documented as optional — the plugin works fine with just Read/Grep/Glob/Bash/WebSearch/WebFetch, and each MCP enhances the review when available.
- Skill orchestrator updated to dispatch either `ios-senior-review:senior-ios-reviewer` (standard) or `ios-senior-review:ios-team-lead` (team mode) based on flag/trigger detection.

### Removed

- References to specific local vault sizes ("88 files"), specific local vault paths, and specific GoodMem space UUIDs. The plugin no longer assumes any particular local knowledge base layout — users bring their own (optional) resources and tell the agent where they are via the dispatch prompt.

## [0.1.0] - 2026-04-14

### Changed

- **BREAKING**: replaced the standalone `SKILL.md` (loaded inline into the main Claude conversation) with a proper Claude Code plugin containing an orchestrator skill and a dedicated Opus 4.6 subagent. Old `SKILL.md` consumers must reinstall as a plugin (see README install section).

### Added

- `ios-senior-review` plugin (under `ios-senior-review/`)
- `review-ios` skill: scope resolution, Apple-specific context gathering (`Info.plist`, `*.entitlements`, `PrivacyInfo.xcprivacy`, build settings, targets, third-party deps, linter config), `--mode submission|engineering|both` flag, and dispatch of the reviewer subagent
- `senior-ios-reviewer` agent: Opus 4.6, read-only tools (Read, Grep, Glob, Bash, TodoWrite, WebSearch, WebFetch, optional serena MCP, optional Context7 MCP, optional GoodMem MCP), simulates BOTH the Apple App Review team AND a senior Apple platform engineer
- 12 review dimensions across 4 tiers: Tier 1 (App Review blockers — rejection risk, privacy, entitlements, security), Tier 2 (high-yield static quality — HIG, accessibility, SwiftUI/UIKit patterns, deep linking & extensions), Tier 3 (engineering quality — Swift language, concurrency safety, performance & memory), Tier 4 (platform integration depth, advisory)
- 5 evidence classes (`SOURCE` / `BUILD` / `SERVER` / `ASC` / `RUNTIME`) with strict rules forbidding `[R]` rejection findings from `RUNTIME` or `ASC` evidence (uses `[R?]` with explicit evidence-gap callout instead)
- 5-level severity tags (`[R]` rejection / `[R?]` likely rejection / `[W]` warning / `[~]` recommendation / `[+]` praise)
- Two independent verdicts: Submission Verdict (READY / LIKELY READY / FIX BEFORE SUBMITTING / WILL BE REJECTED) and Engineering Verdict (STRONG / ACCEPTABLE / NEEDS WORK)
- Marketplace manifest at `.claude-plugin/marketplace.json` for one-step `claude plugin marketplace add` install

### Removed

- Standalone `SKILL.md` from repo root (replaced by the plugin structure under `ios-senior-review/`)

### Why the change

Three upgrades from the standalone-skill architecture:

1. **Fresh context** — the reviewer runs in an isolated subagent with no view of the conversation that wrote the code. No pattern blindness.
2. **Forced Opus 4.6** — the agent is pinned to `model: opus` regardless of main session model.
3. **Read-only enforcement** — the agent has Read/Grep/Glob/Bash/WebSearch but NOT Edit/Write/Agent. Impossible to "accidentally fix" code mid-review.

[0.2.0]: https://github.com/TheMizeGuy/ios-code-review/releases/tag/v0.2.0
[0.1.0]: https://github.com/TheMizeGuy/ios-code-review/releases/tag/v0.1.0
