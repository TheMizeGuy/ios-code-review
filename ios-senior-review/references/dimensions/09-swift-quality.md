# Dimension 9: Swift Language Quality

Tier 3 — Engineering Quality. Senior-engineer concerns. NEVER affects the submission verdict; feeds the engineering verdict only.

| Check | Expected | Evidence |
|---|---|---|
| API design | Swift API Design Guidelines (naming clarity, argument labels, fluent usage) | `SOURCE` |
| Optionals | No force-unwrap outside `@IBOutlet`; proper optional chaining and `guard let` | `SOURCE` |
| Error handling | No empty `catch {}`; errors propagated or handled meaningfully; typed throws where possible | `SOURCE` |
| Value vs reference | Structs for data models; classes only for identity/shared mutable state | `SOURCE` |
| Access control | `private` for implementation; `internal` default; `public` only for API surface | `SOURCE` |
| Deprecated APIs | Project-wide sweep: `UIWebView`, `NSPredicate` → `#Predicate`, `PreviewProvider` → `#Preview`, deprecated StoreKit 1, old lifecycle hooks | `SOURCE` |
