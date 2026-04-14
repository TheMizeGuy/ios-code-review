# Changelog

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
