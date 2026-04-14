# ios-code-review

A Claude Code skill that simulates Apple's App Review team reviewing your iOS codebase. Runs 12 review dimensions across 4 tiers, producing findings with specific guideline references and dual verdicts (submission readiness + engineering quality).

## What it does

| Mode | Question it answers |
|------|-------------------|
| **App Review Simulation** | Will Apple reject this? |
| **Senior Engineering Review** | Is the code good? |

### 12 Review Dimensions (tiered by impact)

**Tier 1 -- App Review Blockers:**
1. App Store Rejection Risk (Guidelines 1-5, top 10 rejection causes with frequency data)
2. Privacy & Data Protection (14 checks including privacy manifests, ATT, tracking domains)
3. Entitlements & Info.plist Audit (10 checks for over-requested capabilities, plist completeness)
4. Security (8 checks from Keychain to ATS to hardcoded secrets)

**Tier 2 -- High-Yield Static Quality:**
5. Human Interface Guidelines (11 checks including Liquid Glass, SF Symbols, localization readiness)
6. Accessibility (14 checks including WCAG 2.2, Large Content Viewer, Smart Invert)
7. SwiftUI / UIKit Patterns (7 checks for modern API usage)
8. Deep Linking & Extensions (6 checks for universal links, App Clips, NSE, widgets)

**Tier 3 -- Engineering Quality:**
9. Swift Language Quality (6 checks)
10. Concurrency Safety (7 checks including Swift 6 readiness, TSan)
11. Performance & Memory (7 checks)

**Tier 4 -- Platform Opportunity (opt-in):**
12. Platform Integration Depth (12 checks for widgets, Siri, haptics, ShareLink, etc.)

### Evidence-based findings

Every finding states its evidence class (`SOURCE`, `BUILD`, `SERVER`, `ASC`, `RUNTIME`) so you know which results are confirmed vs need device verification. Hard rejection verdicts (`[R]`) require `SOURCE` or `BUILD` evidence -- no false certainty.

### Output format

Findings use Apple's review response style:

```
[R] Guideline 5.1.1 -- Privacy Manifest Missing
File: Sources/App/AppDelegate.swift
Evidence: SOURCE
Issue: No PrivacyInfo.xcprivacy found in any target. 12% of submissions
       rejected in Q1 2025 for this. UserDefaults usage detected but undeclared.
Fix: Add PrivacyInfo.xcprivacy with NSPrivacyAccessedAPICategoryUserDefaults
     reason code CA92.1 to every target.
```

Ends with two summary tables and two verdicts:
- **Submission:** READY / LIKELY READY / FIX BEFORE SUBMITTING / WILL BE REJECTED
- **Engineering:** STRONG / ACCEPTABLE / NEEDS WORK

## Installation

### Claude Code (personal skill)

```bash
# Clone into your Claude Code skills directory
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/TheMizeGuy/ios-code-review.git
```

Claude Code auto-discovers skills in `~/.claude/skills/`. The skill will be available immediately in your next session.

### Claude Code (project-level)

To make the skill available only within a specific project:

```bash
cd /path/to/your/ios-project
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/TheMizeGuy/ios-code-review.git
```

### Manual (any AI coding assistant)

Copy `SKILL.md` into your assistant's skill/prompt directory, or paste its contents as a system prompt when you want an iOS code review.

## Usage

### As a slash command in Claude Code

```
/ios-code-review
```

Then point the reviewer at your project. It will map the codebase and run all 12 dimensions.

### As a subagent

Dispatch a dedicated Opus reviewer from your main session:

```javascript
Agent({
  description: "iOS code review -- Apple simulation",
  subagent_type: "general-purpose",
  model: "opus",
  prompt: `
    Use the Skill tool to invoke: ios-code-review

    Then perform a full iOS code review of <path> in both modes
    (submission + engineering). Follow all 12 dimensions.
  `
})
```

### As a standalone prompt

Copy `SKILL.md` contents and paste as context when asking any Claude model to review iOS code.

## Recommended workflow

1. Run automated tools first (SwiftLint, Periphery, Xcode Analyze)
2. Invoke this skill for the policy/compliance/architecture layer those tools can't reach
3. Fix all `[R]` findings before submitting to App Store
4. Address `[W]` findings based on risk tolerance
5. Consider `[~]` recommendations for polish

## Requirements

- Claude Code with skill support (or any AI assistant that accepts markdown prompts)
- An iOS/Swift codebase to review
- Optionally: SwiftLint, Periphery output for the preflight step

## License

MIT
