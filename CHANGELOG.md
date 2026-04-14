# Changelog

All notable changes to this repository will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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

[0.1.0]: https://github.com/TheMizeGuy/ios-code-review/releases/tag/v0.1.0
