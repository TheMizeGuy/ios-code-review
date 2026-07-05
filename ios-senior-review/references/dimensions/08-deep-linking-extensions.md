# Dimension 8: Deep Linking & Extensions

Tier 2 — High-Yield Static Quality. Improves quality and reduces reviewer flags but rarely causes outright rejection by itself. Feeds the Submission Readiness table.

| Check | Expected | Evidence |
|---|---|---|
| Universal links | Associated Domains entitlement present; AASA hosted at `/.well-known/apple-app-site-association` over HTTPS with no redirects | `SOURCE` + `SERVER` |
| `.onOpenURL` | All registered URL schemes and universal link patterns handled | `SOURCE` |
| App Clip (if present) | Size budget met (10/15/50MB tier by minimum OS and invocation type — verify the current tier against App Clip docs); no ads; invocation URLs configured; main-app transition via App Groups and `SKOverlay` | `SOURCE` + `BUILD` |
| NSE (if present) | `NSExtensionPointIdentifier` correct; `mutable-content: 1` in push payloads; 30-second time limit respected; deployment target aligned with main app | `SOURCE` |
| Widget extensions | Timeline provider implemented; App Group for shared data; interactive widgets use App Intents (iOS 17+) | `SOURCE` |
| Share/Action extensions | Correct activation rules; `NSExtensionActivationRule` not using `TRUEPREDICATE` in production | `SOURCE` |
| Safari extensions / content blockers (if present) | Correct extension point; blocker rules valid JSON; no remote code | `SOURCE` |
