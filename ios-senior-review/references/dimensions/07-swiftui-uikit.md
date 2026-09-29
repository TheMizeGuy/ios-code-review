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
| Launch-path view type size and depth | App-root, scene-root and first-screen views keep each `Body` under 8,192 B and their generic nesting shallow; heavy subtrees sit behind nominal `View`/`ViewModifier` boundaries; observer modifiers are mounted as a sibling, never wrapped around the root. Full check below | `SOURCE` (risk) — confirmed only by measurement or a device Release launch |

## Launch-path stack exhaustion (device-only launch crash)

SwiftUI can run the main thread out of stack during the first render and crash the app before it draws a frame. The crash log shows `EXC_BAD_ACCESS` / `KERN_PROTECTION_FAILURE` on the main-thread stack guard page, reported as "Thread stack size exceeded due to excessive recursion" or "Could not determine thread index for stack guard region". The stack is Swift demangler or `TypeDecoder` recursion under SwiftUI `_makeView` frames, with either no app frames but `main` or a single root-view `body.getter`. The OS elides the middle of such stacks, so frame numbers jump by hundreds or more (213 -> 1870), and that gap is the recursion depth.

It is not tied to one phone model: whether a build crashes depends on the stack left for that (device, OS, restored state). In the incidents behind this check, three builds of one production app crashed on launch for every tester, two of them within a day of a hotfix build that launched. Two later builds, one of them live on the App Store, crashed on an iPhone 11 / iOS 26.3.1 and an iPhone 15 Pro Max / iOS 26.3: an old phone and a recent Pro Max, both on iOS 26.3.x, while the developer's own phone and testers on 26.6 and 27 launched fine. A later build crashed again through a different consumer. Three consumers of the same stack add up:

1. **Attribute byte size.** AttributeGraph names every attribute type of 8,192 B or more at first render (its "large attribute" INFO log, on by default on every stock device). Naming a type runs the Swift demangler, which recurses once per nested generic level (`swift_getTypeName` -> `_swift_buildDemanglingForMetadata`). An inline subtree inside `.overlay {}`, `.background {}`, `.safeAreaInset {}` or container content is stored by value, so it inflates the parent's `Body` size.
2. **Type nesting.** Every modifier wraps the view in another `ModifiedContent<...>`, so an outermost modifier chain's length is the type's nesting depth, and transitive `some View` helpers add to it. Device Release codegen can instantiate that metadata from a mangled name at runtime (`__swift_instantiateConcreteTypeFromMangledNameV2` -> `TypeDecoder` recursion). One modifier separated a build that launched (42 outermost modifiers) from one that crashed on every launch (43).
3. **Native construction depth.** Each modifier wrapping the root adds `_makeView` frames before the child is built: 96 `.onChange(of:)` observers wrapped around one app's root view added 728 frames, about 7-8 each.

Identify the launch path before judging a diff. Start at the `@main` App's `WindowGroup` content and follow every wrapper view, `ViewModifier` and `View` extension applied at or above the scene root (they often live in other files, such as a settings-autosave extension on the root view), down to the first screen and everything it renders in its first frame: overlays, insets, toolbars, list rows, `UIHostingConfiguration` cells, and the screen restored from saved state. A changed file or helper is on the launch path if you can reach it from that tree by Grepping for its type and function names and following the callers up to the scene root. Then flag:

- A launch-path attribute value (a view struct, a `Body`, or a closure-produced value such as a `ForEach` row) that plausibly reaches 8 KB: large inline subtrees in `.overlay {}` / `.background {}` / `.safeAreaInset {}` / container content, many stored properties, deep transitive `some View` helpers (one crashing build's growth sat in 16 of them).
- Long outermost modifier chains on root and first-screen bodies (in one app 42 launched and 43 crashed; its guard now caps the chain at 39), and any new wrapper or modifier at or above the scene root (one crashing build added seven).
- Observer and side-effect modifiers (`.onChange`, `.task`, `.onReceive`, and presenters that do not anchor to a source view, such as `.sheet`, `.fullScreenCover` and `.alert`) wrapped around the root view instead of mounted on a sibling (each costs roughly 7-8 native frames; one app's guard fails above +100, about a dozen). `.popover` and `.confirmationDialog` stay on the control that triggers them, or they point at the middle of the window at regular width.
- A diff that raises a launch guard's budget or drops types from what it measures.

Fixes that work: extract subtrees into nominal `View` structs, or collapse a run of modifiers into one non-generic `ViewModifier`; store heavy inline content behind a boundary view that holds the builder closure instead of the built value, so the parent stores a closure; mount observers on a zero-size sibling. A boundary does not shrink what it holds (`ClosureBoundary<C>.Body` is `C`), so content that is itself 8 KB or more still has to be split into smaller nominal views, and a `some View` property or helper is not a boundary at all. The boundary's body re-runs whenever its parent's does, since SwiftUI cannot compare closures, so use it only where the size budget needs it.

```swift
struct ClosureBoundary<Content: View>: View {
    private let content: () -> Content
    init(@ViewBuilder _ content: @escaping () -> Content) { self.content = content }
    var body: some View { content() }
}
// .overlay { ClosureBoundary { HeavyOverlayView(model: model) } }   // stores a closure, not the subtree's value

content.background {
    Color.clear.frame(width: 0, height: 0)
        .allowsHitTesting(false).accessibilityHidden(true)
        .modifier(RootObservers(settings: settings))   // observers as a sibling, not a wrapper
}
```

What does not work: `ViewModifier.concat` or re-ordering (same node count); fixing a different consumer than the one that overflowed (shrinking the root `Body` from 7,480 to 712 B did not stop an observer-depth crash); raising a guard's budget to get green.

The guard is a test that measures the runtime facts from the scene root: `MemoryLayout<V.Body>.size` under 8,192 for every launch-path body, generic nesting from `_typeName(V.Body.self, qualified: true)`, and native frames added above a mounted child. It needs a non-vacuity check (a minimum number of types measured) and a positive control (a deliberately oversize type it must flag), because a walker that silently stops matching looks like a healthy tree. Its reach includes closure-produced values (`ForEach` rows, `GeometryReader` bodies, `UIHostingConfiguration` cell roots), not only named bodies: in one crash the oversize type was an overlay child that an earlier budget test did not watch. Sizes are the linked SwiftUI's runtime layout and moved by tens of bytes between iOS 26.5 and 27, so keep margin below 8,192 B rather than passing by a few bytes. A lexical modifier count is only a proxy: it went green on builds that crashed.

Why nothing else catches it: test suites run on the Simulator, where every one of these builds launched; Debug builds also emit the nesting metadata statically (consumer 2); `xcodebuild archive` succeeds. The recorded reproductions are Release launches on physical devices, plus one optimized Release copy with its main-thread stack forced down to 768 KB (`LC_MAIN.stacksize`). A Debug launch, even on a device, does not clear it.

Severity: a scope that changes the launch path (the triggers above, or the Dimension 1 launch-path row) with no device Release launch on record is ONE finding: `[R?]` Guideline 2.1, filed under Dimension 1, Evidence `SOURCE`, gap "needs device Release launch"; do not also file it here. Risk on an unchanged launch path (a whole-project review) and a missing or proxy-only measuring guard are `[W]` here. In `--mode engineering`, file the launch-path change here as `[W]` and name the device Release launch in the Suggested fix.
