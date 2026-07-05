# Dimension 6: Accessibility

Tier 2 — High-Yield Static Quality. Improves quality and reduces reviewer flags but rarely causes outright rejection by itself. Feeds the Submission Readiness table.

| Check | Expected | Evidence |
|---|---|---|
| VoiceOver | All interactive elements labeled; decorative images `.accessibilityHidden(true)`; logical reading order | `SOURCE` + `RUNTIME` |
| Interactive elements are controls | `Button` (with `.buttonStyle(.plain)` for custom looks), never a bare `.onTapGesture` as the sole interaction — gesture-recognizer-only interactions are invisible to VoiceOver | `SOURCE` + `RUNTIME` |
| Dynamic Type | System text styles or `.dynamicTypeSize`; layout doesn't break at AX sizes | `SOURCE` + `RUNTIME` |
| Color contrast | 4.5:1 minimum (3:1 for large text/UI) — WCAG 2.1 AA | `RUNTIME` |
| Touch targets | 44x44pt minimum; 8pt between targets (long-standing HIG figure — re-check the live HIG when precision matters; Liquid Glass-era spacing guidance may have shifted, unconfirmed) | `SOURCE` + `RUNTIME` |
| Reduce Motion | `accessibilityReduceMotion` respected; crossfade fallbacks | `SOURCE` |
| Voice Control | Buttons have accessible names matching visible labels; `.accessibilityInputLabels` for multiple names | `SOURCE` |
| Element grouping | `.accessibilityElement(children: .combine)` for related content — BUT if the combined children include an interactive element (Button, Toggle), its action becomes unreachable under `.combine` unless re-exposed via `.accessibilityAction(named:)`; flag any `.combine` wrapping an interactive child without a matching `.accessibilityAction` as `[W]` | `SOURCE` |
| Automated audit integrity | If `performAccessibilityAudit` exists in UI tests, its issue handler must NOT be a blanket `{ _ in true }` (that suppresses every issue — a green test with zero coverage); handlers must discriminate by `auditType`/`element.identifier`. If absent entirely, emit `[~]` recommending it | `SOURCE` |
| Accessibility Nutrition Label | ASC label reported and accurate for the 9 features (VoiceOver, Voice Control, Larger Text, Dark Interface, Differentiate Without Color Alone, Sufficient Contrast, Reduced Motion, Captions, Audio Descriptions). Voluntary today; Apple has signaled future mandatory status — zero label = forward-looking `[~]`, not `[R]` | `ASC` — `[~]` only |
| Custom actions | Swipe actions exposed as accessibility custom actions | `SOURCE` |
| Large Content Viewer | Fixed-size elements support `.accessibilityShowsLargeContentViewer` | `SOURCE` |
| Smart Invert | User content (photos, videos, maps) uses `.accessibilityIgnoresInvertColors()` | `SOURCE` |
| WCAG 2.5.7 Dragging | Drag operations have single-pointer alternative | `SOURCE` |
| WCAG 3.3.7 Redundant Entry | Forms don't require re-entering previously provided information | `SOURCE` |
| WCAG 3.3.8 Accessible Auth | Authentication supports biometrics/passkeys/password managers (no cognitive tests) | `SOURCE` |
| Focus Not Obscured | Sticky headers/toolbars don't cover focused content during keyboard/Switch Control navigation | `SOURCE` + `RUNTIME` |
