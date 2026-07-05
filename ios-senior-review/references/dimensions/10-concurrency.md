# Dimension 10: Concurrency Safety

Tier 3 — Engineering Quality. Senior-engineer concerns. NEVER affects the submission verdict; feeds the engineering verdict only.

| Check | Expected | Evidence |
|---|---|---|
| MainActor | UI updates on `@MainActor`; view models annotated `@MainActor` | `SOURCE` |
| Sendable | Types crossing isolation boundaries conform to `Sendable`; `@Sendable` closures correct | `SOURCE` |
| Data races | No unprotected shared mutable state; actors for synchronization | `SOURCE` |
| Structured concurrency | `TaskGroup` / `async let` preferred over unstructured `Task {}` | `SOURCE` |
| Cancellation | Long operations check `Task.isCancelled`; cooperative cancellation | `SOURCE` |
| Language mode | Target on Swift 6 language mode (strict concurrency is the default there), or — if still on Swift 5 mode — `SWIFT_STRICT_CONCURRENCY=complete` enabled or a migration path documented (see Apple's Swift 6 migration guide) | `SOURCE` |
| TSan | Thread Sanitizer enabled in the shared test scheme (`enableThreadSanitizer` attribute on `TestAction` in `.xcscheme` — NOT a pbxproj/xcconfig key) | `SOURCE` |
