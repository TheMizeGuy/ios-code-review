# Dimension 4: Security

Tier 1 — App Review Blocker. Maps directly to rejection causes; must pass for submission. Findings here may be rejection-grade (`[R]` / `[R?]`) subject to the evidence rules in the reviewer manual.

| Check | Expected | Evidence |
|---|---|---|
| Secrets storage | Keychain for credentials, never UserDefaults or plain files; correct `kSecAttrAccessible` level | `SOURCE` |
| ATS exceptions | No blanket `NSAllowsArbitraryLoads`; check `...InWebContent`, `...ForMedia`, per-domain `NSExceptionDomains` with TLS version and justification text | `SOURCE` |
| Certificate pinning | For sensitive API endpoints (optional but recommended) | `SOURCE` |
| Input validation | All user input validated; parameterized queries; WebView restrictions | `SOURCE` |
| Screenshot protection | Sensitive data covered via scene lifecycle when displaying sensitive content | `SOURCE` |
| Debug logging | No tokens in release logs; `OSLog` with `.private` for sensitive data | `SOURCE` |
| Biometric auth | `canEvaluatePolicy` before `evaluatePolicy`; `NSFaceIDUsageDescription` in plist | `SOURCE` |
| No hardcoded secrets | API keys, tokens, credentials not in source; no embedded certificates | `SOURCE` |
