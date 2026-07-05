# Dimension 5: Human Interface Guidelines

Tier 2 — High-Yield Static Quality. Improves quality and reduces reviewer flags but rarely causes outright rejection by itself. Feeds the Submission Readiness table.

| Check | Expected | Evidence |
|---|---|---|
| Navigation | Standard `NavigationStack`; large titles top-level, standard for detail; back button always present | `SOURCE` + `RUNTIME` |
| Tab bar | Bottom, 3-5 items, standard behavior | `SOURCE` |
| Safe areas | Content respects notch, home indicator, Dynamic Island | `RUNTIME` |
| Liquid Glass adoption (iOS 26+) | System glass materials (`.glassEffect()` / `GlassEffectContainer`) rather than custom chrome fighting the system material; no manual overrides that defeat the system's automatic contrast adaptation | `SOURCE` |
| Liquid Glass legibility | Text/icons on translucent surfaces stay legible over busy or content-heavy backgrounds (a known accessibility risk of the material system) | `RUNTIME` |
| App icon | Single asset, no transparency, no drawn corners, no text; dark AND tinted variants (iOS 18+); layered Icon Composer icon for the Liquid Glass era (iOS 26+) | `SOURCE` |
| Launch screen | Matches first real screen, no logo splash | `RUNTIME` |
| Alerts/sheets | System `.alert()` / `.confirmationDialog()`, not custom | `SOURCE` |
| Layout direction | Leading/trailing, never left/right (RTL breaks) | `SOURCE` |
| Destructive actions | `.confirmationDialog()` before irreversible operations | `SOURCE` |
| Colors | Semantic system colors adapting to light/dark/high contrast | `SOURCE` |
| SF Symbols | Used where appropriate over custom icons | `SOURCE` |
| Localization readiness | String Catalogs (`.xcstrings`); no hardcoded strings; `.formatted()` for numbers/dates/currency | `SOURCE` |
