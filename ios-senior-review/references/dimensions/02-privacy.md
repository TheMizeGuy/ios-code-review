# Dimension 2: Privacy & Data Protection

Tier 1 — App Review Blocker. Maps directly to rejection causes; must pass for submission. Findings here may be rejection-grade (`[R]` / `[R?]`) subject to the evidence rules in the reviewer manual.

Privacy is one of the two dominant rejection categories and the fastest-growing one. Be thorough.

| Check | Expected | Evidence |
|---|---|---|
| Privacy manifest | `PrivacyInfo.xcprivacy` in every target; correct Required Reason API codes for all 5 categories (see reason-code table below) | `SOURCE` |
| Required Reason API codes | Codes match the ACTUAL usage scope, not just any valid-looking code — wrong-scope codes cause ITMS-91056 even when the manifest parses | `SOURCE` |
| ATT | `requestTrackingAuthorization` before any tracking; all 4 status cases handled | `SOURCE` |
| ATT timing/UX | Prompt only fires while `applicationState == .active` (silently dropped from background); `notDetermined` guard before every call (the system dialog shows once per install — re-calls are no-ops); any custom pre-prompt screen doesn't mimic the system dialog or use manipulative language | `SOURCE` |
| ATT plist string | `NSUserTrackingUsageDescription` present in Info.plist if ATT prompt shown | `SOURCE` |
| Purpose strings | Every permission has `NS*UsageDescription` in Info.plist | `SOURCE` |
| Tracking domains | If `NSPrivacyTracking: true`, `NSPrivacyTrackingDomains` lists all tracking endpoints | `SOURCE` |
| Third-party SDKs | All include compliant privacy manifests AND valid code signatures | `SOURCE` + `BUILD` |
| SDK privacy audit | All deps in `Package.swift` / `Podfile` checked for included privacy manifests | `SOURCE` |
| Privacy report | Xcode Privacy Report generated and reviewed (Product > Generate Privacy Report) | `BUILD` |
| Privacy label diff | Inferred data collection from SDKs / network / analytics / web views compared against declared App Store Connect labels | `SOURCE` + `ASC` |
| Policy-vs-wire drift | On any diff adding a network send or new endpoint: check `git diff --name-only` for a PAIRED privacy-policy/manifest edit in the same change. If a bundled privacy policy exists, grep it for data-egress claims ("no backend", "only X network call", "anonymous") and check every NEW send against each claim — a code-only diff adding off-device data collection is `[R?]`/`[R]` under 5.1.1/2.3, not a deferrable follow-up | `SOURCE` |
| Collected data types | `NSPrivacyCollectedDataTypes` matches actual collection (email, analytics, crash data) | `SOURCE` |
| GDPR/CCPA | Accessible privacy policy; data deletion flow exists | `SOURCE` + `ASC` |
| Manifest hygiene | Keep `PrivacyInfo.xcprivacy` free of XML comments — plausible (unconfirmed) ITMS-91056 contributor observed in production; rationale belongs in a sibling doc | `SOURCE` — `[~]` only |
| Account deletion | Full deletion available (required June 2022); Apple ID token revocation on delete | `SOURCE` |
| IDFA access | Only after ATT authorization (returns all-zero UUID otherwise) | `SOURCE` |

Required Reason API reason codes (the 5 categories; full tables in Apple TN3183 — codes below are the common ones, always match to actual usage):

| Category | Codes | Scope gotcha |
|---|---|---|
| UserDefaults | `CA92.1` (app-private) / `1C8F.1` (App Group shared) / `C56D.1` (third-party SDK only) / `AC6B.1` (MDM managed configuration) | **App-Group UserDefaults access MUST use `1C8F.1`** — using `CA92.1` or `C56D.1` for App-Group access is a documented repeatable cause of ITMS-91056 rejection loops even when the manifest otherwise looks correct |
| File timestamp | `C617.1` (app/group/CloudKit container files) / `3B52.1` (user-granted files) / `DDA9.1` (display to user) | `std::filesystem::exists()` triggers this category via `stat()`; user-selected files need `3B52.1`, not `C617.1` |
| System boot time | `35F9.1` (elapsed time in-app) / `8FFB.1` (absolute event timestamps) | |
| Disk space | `E174.1` (check space before writing) / `85F4.1` (display to user) | |
| Active keyboards | `3EC4.1` (keyboard apps) / `54BD.1` (custom keyboard UI) | |
