# Dimension 1: App Store Rejection Risk (Guidelines 1-5)

Tier 1 — App Review Blocker. Maps directly to rejection causes; must pass for submission. Findings here may be rejection-grade (`[R]` / `[R?]`) subject to the evidence rules in the reviewer manual.

Roughly 1 in 4 submissions is rejected (App Store Transparency Report aggregate); Guideline 2.1 and 5.1.1 together account for close to half of rejections in recent community tallies. Top 10 causes with approximate frequencies (community-sourced, 2024-2026):

| # | Check | Guideline | Freq | Evidence |
|---|---|---|---|---|
| 1 | Crashes, bugs, broken flows | 2.1 | ~25-34% | `RUNTIME` — reproduce on simulator when available; do not assert `[R]` without SOURCE-provable crash sites |
| 2a | Privacy manifest missing from a target | 5.1.1 | ~18-21% | `SOURCE` |
| 2b | Declared privacy labels inaccurate vs actual collection | 5.1.1 | (same bucket) | `ASC` — never `[R]` from this half alone |
| 3 | Misleading metadata | 2.3 | ~12% | `ASC` |
| 4 | Digital goods not using IAP | 3.1.1 | ~10% | `SOURCE` |
| 5 | Web wrapper, no native value | 4.2 | ~8% | `SOURCE` |
| 6 | Private/deprecated API usage | 2.5 | ~6% | `SOURCE` — grep for `UIWebView`, private selectors, deprecated APIs |
| 7 | UGC without filtering/reporting/blocking | 1.2 | ~5% | `SOURCE` |
| 8 | Substandard design / no clear value | 4.0 (Design; cite 4.2 for minimum-functionality cases) | ~4% | `RUNTIME` + `ASC` |
| 9 | Unauthorized data practices | 5.1.2 | ~3% | `SOURCE` |
| 10 | Screenshots don't match functionality | 2.3.7 | ~3% | `ASC` |

Additional mandatory checks:

| Check | Guideline | Evidence |
|---|---|---|
| Built with the current mandatory SDK/toolchain (iOS 26 SDK / Xcode 26 required for all submissions since ~April 2026 — verify the current floor at review time); stale toolchain is an automated upload rejection before any human review | 2.1 / submission gate | `BUILD` |
| Binary includes arm64 slice; binary size within limits (simulator-only or oversized archives are rejected at upload processing, not at review) | submission gate | `BUILD` |
| Age-rating questionnaire answered under the current tier system (4+/9+/13+/16+/18+ since 2026-01-31; 12+/17+ retired) — missing responses block new submissions/updates | ASC requirement | `ASC` |
| Sign in with Apple offered if any third-party login exists (or equivalent privacy-focused alternative) | 4.8 | `SOURCE` |
| `UIWebView` usage (deprecated; binary scan catches; must migrate to `WKWebView`) | 2.5 | `SOURCE` |
| Kids Category: COPPA, no third-party ads, parental gates before purchases/external links | 1.3 | `SOURCE` |
| Screen Time / FamilyControls (if `import FamilyControls`/`DeviceActivity`/`ManagedSettings` detected): Distribution entitlement obtained; restrictions only via `FamilyActivityPicker` (opaque tokens never decoded); Settings/Phone/emergency services never shielded; user has a clear remove-restrictions path; privacy policy covers Screen Time data | 1.3 / 5.1.1 | `SOURCE` |
| HealthKit: data minimization, accuracy disclosure, no advertising use, required purpose strings; medical/health apps display the regulatory status indicator (Spring 2026 requirement) | 5.1.1 / 1.4 | `SOURCE` + `ASC` |
| Loot boxes: odds/probabilities disclosed | 3.1.1 | `SOURCE` |
| AI features: personal-data sharing with third-party AI clearly disclosed naming the specific provider (generic "service providers" language is insufficient); explicit permission obtained BEFORE sharing; user can decline AI features without losing core functionality and can revoke consent later; chatbot apps comply with 1.2 even for AI-generated content | 5.1.2(i) / 1.2 | `SOURCE` |
| UGC (incl. random/anonymous chat — always in-scope for 1.2 regardless of how the feature is framed): filtering, reporting, blocking; the developer, not the platform, is responsible for removing violating content (guideline clarification 2026-02) | 1.2 | `SOURCE` |
| UGC marketing copy: words framing anonymity as a feature ("anonymous", "no account needed") in description/screenshots/UI strings correlate with vague 1.1 re-rejections on otherwise-clean UGC apps (community-observed, not Apple-confirmed — soft flag only) | 1.1 | `ASC` — `[~]` only |
| Placeholder content: grep app copy/strings/screenshots for "lorem ipsum", "TODO", "test", "coming soon"; App Store category matches actual functionality (a standalone 2.3 rejection cause) | 2.3 | `SOURCE` + `ASC` |
| Duplicate/low-effort apps (4.3(b), tightened 2026-06): saturated utility categories (dating, flashlight, sound effects, wallpaper, simple timers, fortune telling) need a meaningfully different/improved experience; low-effort categories (drinking games, Kama Sutra, fart/burp apps) risk Developer Program removal on repeated submission | 4.3(b) | `SOURCE` + `ASC` |
| Live Activities / Push / Game Center not used for spam, phishing, or unsolicited marketing messages | 4.5.3 | `SOURCE` |
| EU DMA (if distributing in EU): Notarization required for ALL EU-distributed apps regardless of channel; IAP and external purchase links cannot be mixed in the same storefront; fee structure transitioned toward the 5% Core Technology Commission (2026-01 intent — verify the current commission structure at submission time, the transition timeline has slipped before) | 3.1.1(a) / DMA addendum | `SOURCE` + `ASC` |
| Subscription compliance: grace period/billing retry handled; `Transaction.currentEntitlements` checked; terms (price, duration, auto-renew, cancellation) displayed before purchase | 3.1.2 | `SOURCE` |
| StoreKit 2 lifecycle: `transaction.finish()` called after every processed transaction (unfinished transactions re-deliver every launch); `Transaction.updates` listener started at app launch (missed renewals/revocations otherwise); `revocationDate` checked before granting access (refunded purchases stay entitled otherwise); entitlements re-checked on foreground and post-purchase, not only at launch; restore-purchases button present; promoted IAP handled | 3.1 / 3.1.2 | `SOURCE` |
