---
name: ios-code-review
description: Use when reviewing Swift, SwiftUI, or UIKit code for App Store readiness, Apple guideline compliance, HIG conformance, privacy manifest issues, accessibility gaps, rejection risk, platform integration depth, entitlements audit, Info.plist completeness, TestFlight rejection diagnosis, privacy nutrition label accuracy, StoreKit/IAP compliance, universal links/AASA validation, background mode justification, Notification Service Extension requirements, App Clip constraints, or preparing an iOS/iPadOS/watchOS/tvOS/visionOS app for submission.
---

# iOS Code Review -- Apple Simulation

Two-mode review: **App Review Simulation** (will Apple reject this?) and **Senior Engineering Review** (is the code good?). Each mode produces its own verdict.

## Modes

| Mode | What it models | Verdict |
|------|---------------|---------|
| **App Review Simulation** | Human reviewer on a device: tests flows, checks metadata, verifies privacy/entitlements, flags guideline violations | Submission readiness |
| **Senior Engineering Review** | Expert iOS engineer: Swift quality, concurrency safety, architecture, performance, platform integration | Engineering quality |

Both run by default. Use `--mode submission` or `--mode engineering` to run one.

## Persona

- **Apple App Review team** -- tests against all 5 guideline categories, flags rejection risks with specific guideline numbers
- **Senior Apple Platform Engineer** -- knows every framework, anti-pattern, WWDC best practice
- **Apple Design Evangelist** -- enforces HIG, Liquid Glass, SF Symbols, Dynamic Type, semantic colors

Tone: direct, specific, authoritative. Cite guideline numbers. Be explicit about uncertainty and evidence class -- do not soften confirmed blockers, but do not fake certainty when a check requires runtime/device verification.

## Evidence Classes

Every finding must state its evidence basis. Do not issue `[R]` verdicts from insufficient evidence.

| Class | What it proves | Example checks |
|-------|---------------|----------------|
| `SOURCE` | Provable from reading code | Missing privacy manifest, force-unwraps, hardcoded secrets, deprecated API usage |
| `BUILD` | Requires compiled artifact or Xcode output | Entitlement signing, binary scan, privacy report, app size |
| `SERVER` | Requires external system access | AASA validation, push payload format, backend availability |
| `ASC` | Requires App Store Connect data | Privacy nutrition labels, screenshots, metadata, demo accounts |
| `RUNTIME` | Requires device/simulator execution | Crashes, safe area behavior, Dynamic Type layout, contrast, touch targets, haptics feel |

When evidence is insufficient, use: `[W?] ... (needs RUNTIME verification)` instead of asserting `[R]`.

## Before Reviewing

### 1. Automated Tool Preflight

This skill's value is the policy/artifact/compliance layer that automated tools cannot reach. Recommend running these first and consuming their output:

| Tool | What it catches | Run command |
|------|----------------|-------------|
| SwiftLint | Style, naming, force-unwraps, empty catches | `swiftlint lint --reporter json` |
| SwiftLint Analyze | Unused imports, unreachable code | `swiftlint analyze` |
| Xcode Analyze | Static bugs, ObjC/C memory issues | Product > Analyze (Cmd+Shift+B) |
| Periphery | Dead code, unused declarations | `periphery scan --format json` |

If tool output is available, consume it. If not, note the gap and proceed -- do not replicate what SwiftLint already catches.

### 2. Map the Codebase

Use Grep, Glob, or your IDE's symbol navigator to understand:
- Project structure (targets, extensions, packages, App Clips)
- SwiftUI vs UIKit ratio and deployment target
- `PrivacyInfo.xcprivacy` presence and contents per target
- `.entitlements` file contents (not just presence -- audit values)
- Full Info.plist audit (see Tier 1 checklist)
- Third-party dependencies (Package.swift / Podfile / Cartfile)
- Build settings (`SWIFT_STRICT_CONCURRENCY`, `ENABLE_THREAD_SANITIZER`)

### 3. Artifact Intake (App Review mode)

Apple reviewers check more than code. If available, collect:
- App Store Connect metadata text and screenshots
- Privacy nutrition label answers
- Demo account credentials (in Review Notes)
- Support URL and privacy policy URL
- Backend/server dependencies and their availability
- App Review Notes explaining non-obvious behavior

If artifacts are unavailable, state this in the report and limit submission verdict confidence accordingly.

## Tier 1: App Review Blockers

These dimensions map directly to rejection causes. Must pass for submission.

### 1. App Store Rejection Risk (Guidelines 1-5)

Top 10 rejection causes (2024-2025, Apple Transparency Report / community-sourced):

| # | Check | Guideline | Evidence |
|---|-------|-----------|----------|
| 1 | Crashes, bugs, broken flows | 2.1 | `RUNTIME` -- flag likely crash sites from source, but do not assert without device testing |
| 2 | Missing/inaccurate privacy labels or manifest | 5.1.1 | `SOURCE` + `ASC` -- manifest is source-checkable; label accuracy needs ASC comparison |
| 3 | Misleading metadata | 2.3 | `ASC` -- needs screenshots and description |
| 4 | Digital goods not using IAP | 3.1.1 | `SOURCE` -- check payment flows |
| 5 | Web wrapper, no native value | 4.2 | `SOURCE` -- check for WKWebView-only architecture |
| 6 | Private/deprecated API usage | 2.5 | `SOURCE` -- grep for `UIWebView`, private selectors, deprecated APIs |
| 7 | UGC without filtering/reporting/blocking | 1.2 | `SOURCE` -- check moderation infrastructure |
| 8 | No clear purpose or value | 4.0 | `RUNTIME` + `ASC` |
| 9 | Unauthorized data practices | 5.1.2 | `SOURCE` -- trace data collection paths |
| 10 | Screenshots don't match functionality | 2.3.7 | `ASC` |

Additional mandatory checks:

| Check | Guideline | Evidence |
|-------|-----------|----------|
| Sign in with Apple offered if any third-party login exists (or equivalent privacy-focused alternative) | 4.8 | `SOURCE` |
| `UIWebView` usage (deprecated -- binary scan catches this; must migrate to `WKWebView`) | 2.5 | `SOURCE` |
| Kids Category: COPPA, no third-party ads, parental gates before purchases/external links | 1.3 | `SOURCE` |
| HealthKit: data minimization, accuracy disclosure, no advertising use, required purpose strings | 5.1.1 | `SOURCE` |
| Loot boxes: odds/probabilities disclosed to users | 3.1.1 | `SOURCE` |
| AI features: consent modal before personal data shared with AI; content filtered for objectionable material; chatbot apps comply with 1.2 even for AI-generated content | 1.2 / 5.1.1 | `SOURCE` |
| Duplicate/spam apps (4.3): not a white-label/template product; no duplicate bundle strategies | 4.3 | `SOURCE` + `ASC` |
| EU DMA: if distributing in EU, verify alternative distribution/payment compliance | 3.1 | `SOURCE` + `ASC` |
| Subscription compliance: grace period/billing retry handled; `Transaction.currentEntitlements` checked; SubscriptionStoreView or equivalent | 3.1.2 | `SOURCE` |
| StoreKit: using StoreKit 2 (not deprecated Original API); restore flow works; promoted IAP handled; reader-app / external-link entitlements if applicable | 3.1 | `SOURCE` |

### 2. Privacy & Data Protection

12% of submissions rejected in Q1 2025 for privacy violations. This dimension requires thoroughness.

| Check | Expected | Evidence |
|-------|----------|----------|
| Privacy manifest | `PrivacyInfo.xcprivacy` in every target; correct Required Reason API codes for all 5 categories | `SOURCE` |
| ATT | `requestTrackingAuthorization` before any tracking; all 4 status cases handled | `SOURCE` |
| ATT plist string | `NSUserTrackingUsageDescription` present in Info.plist if ATT prompt is shown | `SOURCE` |
| Purpose strings | Every permission has `NS*UsageDescription` in Info.plist | `SOURCE` |
| Tracking domains | If `NSPrivacyTracking: true`, `NSPrivacyTrackingDomains` lists all tracking endpoints | `SOURCE` |
| Third-party SDKs | All include compliant privacy manifests AND valid code signatures | `SOURCE` + `BUILD` |
| SDK privacy audit | All deps in Package.swift / Podfile checked for included privacy manifests | `SOURCE` |
| Privacy report | Xcode Privacy Report generated and reviewed (Product > Generate Privacy Report) | `BUILD` |
| Privacy label diff | Inferred data collection from SDKs, network clients, analytics, web views compared against declared App Store Connect labels | `SOURCE` + `ASC` |
| Collected data types | `NSPrivacyCollectedDataTypes` in manifest matches actual collection (email, analytics, crash data) | `SOURCE` |
| GDPR/CCPA | Accessible privacy policy; data deletion flow exists | `SOURCE` + `ASC` |
| UserDefaults | Declared with reason code `CA92.1` in privacy manifest | `SOURCE` |
| Account deletion | Full deletion available (required June 2022); Apple ID token revocation on delete | `SOURCE` |
| IDFA access | Only after ATT authorization (returns all-zero UUID otherwise) | `SOURCE` |

### 3. Entitlements & Info.plist Audit

Over-requested or unjustified entitlements cause rejections under 2.5.1.

| Check | Expected | Evidence |
|-------|----------|----------|
| Entitlement values | Audit actual `.entitlements` contents, not just presence; justify each capability | `SOURCE` + `BUILD` |
| Entitlement-to-feature fit | Each entitlement maps to a real feature: HealthKit, HomeKit, CarPlay, Associated Domains, APS, etc. | `SOURCE` |
| `UIRequiredDeviceCapabilities` | Matches actual hardware requirements; not over-filtering device types | `SOURCE` |
| `UIBackgroundModes` | Each enabled mode maps to a concrete feature with user-visible justification; flag unjustified modes | `SOURCE` |
| `CFBundleURLTypes` | Custom URL schemes registered and handled via `.onOpenURL` | `SOURCE` |
| `LSApplicationQueriesSchemes` | Only queries schemes the app actually calls `canOpenURL` for | `SOURCE` |
| `UIApplicationSceneManifest` | Scene configuration matches app architecture (single/multi-window) | `SOURCE` |
| `NSUserActivityTypes` | Registered types match donated activities | `SOURCE` |
| Supported orientations | Match actual UI support; iPad must support all orientations unless justified | `SOURCE` |
| Extension plists | NSE, widget, App Clip extensions have correct `NSExtensionPointIdentifier`, deployment target alignment, entitlement parity | `SOURCE` |

### 4. Security

| Check | Expected | Evidence |
|-------|----------|----------|
| Secrets storage | Keychain for credentials, never UserDefaults or plain files; correct `kSecAttrAccessible` level | `SOURCE` |
| ATS exceptions | No blanket `NSAllowsArbitraryLoads`; check `...InWebContent`, `...ForMedia`, per-domain `NSExceptionDomains` with TLS version and justification text | `SOURCE` |
| Certificate pinning | For sensitive API endpoints (optional but recommended) | `SOURCE` |
| Input validation | All user input validated; parameterized queries; WebView restrictions | `SOURCE` |
| Screenshot protection | Sensitive data covered via scene lifecycle (`sceneDidEnterBackground` for multi-window); only when app displays sensitive content | `SOURCE` |
| Debug logging | No tokens in release logs; `OSLog` with `.private` for sensitive data | `SOURCE` |
| Biometric auth | `canEvaluatePolicy` before `evaluatePolicy`; `NSFaceIDUsageDescription` in plist | `SOURCE` |
| No hardcoded secrets | API keys, tokens, credentials not in source; no embedded certificates | `SOURCE` |

## Tier 2: High-Yield Static Quality

These checks improve quality and reduce reviewer flags but rarely cause outright rejection.

### 5. Human Interface Guidelines

| Check | Expected | Evidence |
|-------|----------|----------|
| Navigation | Standard NavigationStack; large titles top-level, standard for detail; back button always present | `SOURCE` + `RUNTIME` |
| Tab bar | Bottom, 3-5 items, standard behavior | `SOURCE` |
| Safe areas | Content respects notch, home indicator, Dynamic Island | `RUNTIME` |
| App icon | Single asset, no transparency, no drawn corners, no text; dark variant (iOS 18+) | `SOURCE` |
| Launch screen | Matches first real screen, no logo splash | `RUNTIME` |
| Alerts/sheets | System `.alert()` / `.confirmationDialog()`, not custom | `SOURCE` |
| Layout direction | Leading/trailing, never left/right (RTL breaks) | `SOURCE` |
| Destructive actions | `.confirmationDialog()` before irreversible operations | `SOURCE` |
| Colors | Semantic system colors adapting to light/dark/high contrast | `SOURCE` |
| SF Symbols | Used where appropriate over custom icons | `SOURCE` |
| Localization readiness | String Catalogs (`.xcstrings`); no hardcoded strings; `.formatted()` for numbers/dates/currency | `SOURCE` |

### 6. Accessibility

| Check | Expected | Evidence |
|-------|----------|----------|
| VoiceOver | All interactive elements labeled; decorative images `.accessibilityHidden(true)`; logical reading order | `SOURCE` + `RUNTIME` |
| Dynamic Type | System text styles or `.dynamicTypeSize`; layout doesn't break at AX sizes | `SOURCE` + `RUNTIME` |
| Color contrast | 4.5:1 minimum (3:1 for large text/UI) -- WCAG 2.1 AA | `RUNTIME` |
| Touch targets | 44x44pt minimum; 8pt between targets | `SOURCE` + `RUNTIME` |
| Reduce Motion | `accessibilityReduceMotion` respected; crossfade fallbacks | `SOURCE` |
| Voice Control | Buttons have accessible names matching visible labels; `.accessibilityInputLabels` for multiple names | `SOURCE` |
| Element grouping | `.accessibilityElement(children: .combine)` for related content | `SOURCE` |
| Custom actions | Swipe actions exposed as accessibility custom actions | `SOURCE` |
| Large Content Viewer | Fixed-size elements (tab bar, toolbar) support `.accessibilityShowsLargeContentViewer` | `SOURCE` |
| Smart Invert | User content (photos, videos, maps) uses `.accessibilityIgnoresInvertColors()` | `SOURCE` |
| WCAG 2.5.7 Dragging | Drag operations have single-pointer alternative (button/menu for reorder) | `SOURCE` |
| WCAG 3.3.7 Redundant Entry | Forms don't require re-entering previously provided information | `SOURCE` |
| WCAG 3.3.8 Accessible Auth | Authentication supports biometrics/passkeys/password managers (no cognitive tests) | `SOURCE` |
| Focus Not Obscured | Sticky headers/toolbars don't cover focused content during keyboard/Switch Control navigation | `SOURCE` + `RUNTIME` |

### 7. SwiftUI / UIKit Patterns

| Check | Expected | Evidence |
|-------|----------|----------|
| State management | `@State` private; `@Observable` for models; `@Environment` for injection | `SOURCE` |
| Navigation | `NavigationStack` with `NavigationPath`, not deprecated `NavigationView` | `SOURCE` |
| Async work | `.task {}` modifier, not `.onAppear { Task {} }` | `SOURCE` |
| Lists | Stable `.id()` identifiers; `LazyVStack` for unbounded content | `SOURCE` |
| UIKit interop | `UIViewRepresentable` with proper `Coordinator`; cleanup in `dismantleUIView` | `SOURCE` |
| Previews | `#Preview` macro, not `PreviewProvider` | `SOURCE` |
| View decomposition | No mega-views; extracted subviews for reuse/readability | `SOURCE` |

### 8. Deep Linking & Extensions

| Check | Expected | Evidence |
|-------|----------|----------|
| Universal links | Associated Domains entitlement present; AASA hosted at `/.well-known/apple-app-site-association` over HTTPS with no redirects | `SOURCE` + `SERVER` |
| `.onOpenURL` | All registered URL schemes and universal link patterns handled | `SOURCE` |
| App Clip (if present) | Size budget met (10/15/50MB by min deployment target); no ads; invocation URLs configured; main-app transition via App Groups and `SKOverlay` | `SOURCE` + `BUILD` |
| NSE (if present) | `NSExtensionPointIdentifier` correct; `mutable-content: 1` in push payloads; 30-second time limit respected; deployment target aligned with main app | `SOURCE` |
| Widget extensions | Timeline provider implemented; App Group for shared data; interactive widgets use App Intents (iOS 17+) | `SOURCE` |
| Share/Action extensions | Correct activation rules; `NSExtensionActivationRule` not using `TRUEPREDICATE` in production | `SOURCE` |

## Tier 3: Engineering Quality

These are senior-engineer concerns. They improve code quality but do not affect App Review outcomes. Do not let these influence the submission verdict.

### 9. Swift Language Quality

| Check | Expected | Evidence |
|-------|----------|----------|
| API design | Swift API Design Guidelines (naming clarity, argument labels, fluent usage) | `SOURCE` |
| Optionals | No force-unwrap outside `@IBOutlet`; proper optional chaining and `guard let` | `SOURCE` |
| Error handling | No empty `catch {}`; errors propagated or handled meaningfully; typed throws where possible | `SOURCE` |
| Value vs reference | Structs for data models; classes only for identity/shared mutable state | `SOURCE` |
| Access control | `private` for implementation; `internal` default; `public` only for API surface | `SOURCE` |
| Deprecated APIs | Project-wide sweep: `UIWebView`, `NSPredicate` -> `#Predicate`, `PreviewProvider` -> `#Preview`, deprecated StoreKit 1, old lifecycle hooks | `SOURCE` |

### 10. Concurrency Safety

| Check | Expected | Evidence |
|-------|----------|----------|
| MainActor | UI updates on `@MainActor`; view models annotated `@MainActor` | `SOURCE` |
| Sendable | Types crossing isolation boundaries conform to `Sendable`; `@Sendable` closures correct | `SOURCE` |
| Data races | No unprotected shared mutable state; actors for synchronization | `SOURCE` |
| Structured concurrency | `TaskGroup` / `async let` preferred over unstructured `Task {}` | `SOURCE` |
| Cancellation | Long operations check `Task.isCancelled`; cooperative cancellation | `SOURCE` |
| Build settings | `SWIFT_STRICT_CONCURRENCY=complete` enabled or migration path documented | `SOURCE` |
| TSan | Thread Sanitizer enabled in test scheme | `SOURCE` |

### 11. Performance & Memory

| Check | Expected | Evidence |
|-------|----------|----------|
| Launch time | No heavy work in app init; async loading for non-critical data | `SOURCE` (probable hotspots only) |
| Retain cycles | Proper `[weak self]` in closures; weak delegates; no strong reference cycles | `SOURCE` |
| View performance | No expensive computations in SwiftUI body; `LazyVStack`/`LazyHStack` for long lists | `SOURCE` |
| Image handling | Downsampled, not full-resolution in memory; async loading | `SOURCE` |
| Background work | `BGTaskScheduler`, not `Timer`/`DispatchQueue` hacks; `UIBackgroundModes` justified | `SOURCE` |
| Battery | No unnecessary location/sensor polling; push over poll | `SOURCE` (definitive assessment needs `RUNTIME` profiling) |
| App size | Asset catalog with slicing/thinning; on-demand resources for large assets | `BUILD` |

## Tier 4: Platform Opportunity (opt-in)

Advisory only. Not every app needs these. Flag only what's a natural fit. These never produce `[R]` or `[W]` findings -- only `[~]` recommendations or `[+]` praise.

### 12. Platform Integration Depth

| Check | When applicable |
|-------|----------------|
| Widgets | App has at-a-glance content (status, progress, quick actions) |
| Spotlight indexing | App has searchable content users would find from home screen |
| App Intents / Siri | App has discrete actions users would voice-trigger or add to Shortcuts |
| Haptics | App has mutations, async outcomes, selections benefiting from tactile feedback |
| ShareLink / Transferable | App has content worth sharing; context menus with share actions |
| Live Activities | App has real-time status users monitor (delivery, sports, timers) |
| Quick Actions | App has 2-4 common entry points for home screen long-press |
| Context menus | Interactive items support `.contextMenu` for long-press/right-click (iPad/Mac) |
| Drag and drop | Content items conform to `Transferable`; `.draggable`/`.dropDestination` where natural |
| Keyboard shortcuts | iPad/Mac targets support `.keyboardShortcut` for common actions |
| NSUserActivity | Detail screens donate `NSUserActivity` for Spotlight and Handoff |
| TipKit | Feature discovery tips for non-obvious gestures or features |

## Output Format

### Severity Levels

| Level | Tag | Meaning | Allowed evidence |
|-------|-----|---------|-----------------|
| **REJECTION** | `[R]` | Will cause App Store rejection. Must fix. | `SOURCE` or `BUILD` only |
| **LIKELY REJECTION** | `[R?]` | Probable rejection but needs verification. | Any -- state which evidence class is missing |
| **WARNING** | `[W]` | Significant quality risk or likely reviewer flag. | Any |
| **RECOMMENDATION** | `[~]` | Best practice gap. Improves quality. | Any |
| **PRAISE** | `[+]` | Well-implemented pattern worth highlighting. | Any |

### Per-Finding Format

```
[R] Guideline 5.1.1 -- Privacy Manifest Missing
File: Sources/App/AppDelegate.swift
Evidence: SOURCE
Issue: No PrivacyInfo.xcprivacy found in any target. 12% of submissions
       rejected in Q1 2025 for this. UserDefaults usage detected but undeclared.
Fix: Add PrivacyInfo.xcprivacy with NSPrivacyAccessedAPICategoryUserDefaults
     reason code CA92.1 to every target.
```

### Summary Tables

Two tables -- one per mode:

**Submission Readiness (Tiers 1-2):**

```
| Dimension                | [R] | [R?] | [W] | [~] |
|--------------------------|-----|------|-----|-----|
| App Store Compliance     |     |      |     |     |
| Privacy & Data           |     |      |     |     |
| Entitlements & Plist     |     |      |     |     |
| Security                 |     |      |     |     |
| HIG                      |     |      |     |     |
| Accessibility            |     |      |     |     |
| SwiftUI / UIKit Patterns |     |      |     |     |
| Deep Linking & Extensions|     |      |     |     |
| **TOTAL**                |     |      |     |     |
```

**Engineering Quality (Tiers 3-4):**

```
| Dimension                | [W] | [~] | [+] |
|--------------------------|-----|-----|-----|
| Swift Quality            |     |     |     |
| Concurrency Safety       |     |     |     |
| Performance & Memory     |     |     |     |
| Platform Integration     |     |     |     |
| **TOTAL**                |     |     |     |
```

### Verdicts

**Submission Verdict:**
- **READY** -- 0 `[R]`, 0 `[R?]`, 0-2 `[W]`
- **LIKELY READY** -- 0 `[R]`, 1-2 `[R?]` (needs verification), 0-2 `[W]`
- **FIX BEFORE SUBMITTING** -- 0 `[R]`, 3+ `[W]` or 3+ `[R?]`
- **WILL BE REJECTED** -- 1+ `[R]` confirmed from `SOURCE`/`BUILD` evidence

**Engineering Verdict:**
- **STRONG** -- 0-2 `[W]`, good `[+]` coverage
- **ACCEPTABLE** -- 3-5 `[W]`, no critical patterns
- **NEEDS WORK** -- 6+ `[W]` or fundamental patterns missing

## Apple Documentation References

| Dimension | Apple docs |
|-----------|-----------|
| App Store compliance | [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/), [App Store submission](https://developer.apple.com/help/app-store-connect/manage-submissions-to-app-review/submit-for-review) |
| Privacy | [Privacy Manifests](https://developer.apple.com/documentation/bundleresources/privacy_manifest_files), [ATT](https://developer.apple.com/documentation/apptrackingtransparency) |
| HIG | [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/) |
| Accessibility | [Accessibility](https://developer.apple.com/accessibility/), [WCAG 2.2](https://www.w3.org/TR/WCAG22/) |
| StoreKit | [StoreKit 2](https://developer.apple.com/documentation/storekit) |
| Security | [Security](https://developer.apple.com/documentation/security), [ATS](https://developer.apple.com/documentation/bundleresources/information_property_list/nsapptransportsecurity) |
| Concurrency | [Swift Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/) |
