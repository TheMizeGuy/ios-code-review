# Dimension 7: SwiftUI / UIKit Patterns

Tier 2 — High-Yield Static Quality. Improves quality and reduces reviewer flags but rarely causes outright rejection by itself. Feeds the Submission Readiness table.

| Check | Expected | Evidence |
|---|---|---|
| State management | `@State` private; `@Observable` for models; `@Environment` for injection | `SOURCE` |
| Navigation | `NavigationStack` with `NavigationPath`, not deprecated `NavigationView` | `SOURCE` |
| Async work | `.task {}` modifier, not `.onAppear { Task {} }` | `SOURCE` |
| Lists | Stable `.id()` identifiers; `LazyVStack` for unbounded content | `SOURCE` |
| UIKit interop | `UIViewRepresentable` with proper `Coordinator`; cleanup in `dismantleUIView` | `SOURCE` |
| Previews | `#Preview` macro, not `PreviewProvider` | `SOURCE` |
| View decomposition | No mega-views; extracted subviews for reuse/readability | `SOURCE` |
