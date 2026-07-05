# Dimension 3: Entitlements & Info.plist Audit

Tier 1 — App Review Blocker. Maps directly to rejection causes; must pass for submission. Findings here may be rejection-grade (`[R]` / `[R?]`) subject to the evidence rules in the reviewer manual.

Over-requested or unjustified entitlements cause rejections under 2.5.1.

| Check | Expected | Evidence |
|---|---|---|
| Entitlement values | Audit actual `.entitlements` contents, not just presence; justify each capability | `SOURCE` + `BUILD` |
| Entitlement-to-feature fit | Each entitlement maps to a real feature: HealthKit, HomeKit, CarPlay, Associated Domains, APS, etc. | `SOURCE` |
| Export compliance | `ITSAppUsesNonExemptEncryption` declared: `NO` if only standard HTTPS/TLS/system crypto; `YES` + export-compliance documentation if the app implements custom encryption. A false `NO` with custom crypto is a legal/compliance risk, not just a prompt-skip | `SOURCE` |
| Code-signing diagnostics | ITMS-90034 = unsigned embedded framework (check "Embed & Sign"); ITMS-90046 = entitlement not enabled on the App ID before the profile was generated; "profile doesn't include the X capability" = provisioning profiles are point-in-time snapshots — adding a capability to the bundle ID does NOT retroactively update an existing profile; regenerate explicitly (manual signing especially; `-allowProvisioningUpdates` does not fix it) | `BUILD` |
| `UIRequiredDeviceCapabilities` | Matches actual hardware requirements; not over-filtering | `SOURCE` |
| `UIBackgroundModes` | Each enabled mode maps to a concrete feature with user-visible justification | `SOURCE` |
| `CFBundleURLTypes` | Custom URL schemes registered and handled via `.onOpenURL` | `SOURCE` |
| `LSApplicationQueriesSchemes` | Only queries schemes the app actually calls `canOpenURL` for | `SOURCE` |
| `UIApplicationSceneManifest` | Scene configuration matches app architecture | `SOURCE` |
| `NSUserActivityTypes` | Registered types match donated activities | `SOURCE` |
| Supported orientations | Match actual UI; iPad must support all orientations unless justified | `SOURCE` |
| Extension plists | NSE, widget, App Clip extensions have correct `NSExtensionPointIdentifier`, deployment target alignment, entitlement parity | `SOURCE` |
