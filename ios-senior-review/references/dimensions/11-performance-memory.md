# Dimension 11: Performance & Memory

Tier 3 — Engineering Quality. Senior-engineer concerns. NEVER affects the submission verdict; feeds the engineering verdict only.

| Check | Expected | Evidence |
|---|---|---|
| Launch time | No heavy work in app init; async loading for non-critical data | `SOURCE` (probable hotspots only) |
| Retain cycles | Proper `[weak self]` in closures; weak delegates; no strong reference cycles | `SOURCE` |
| View performance | No expensive computations in SwiftUI body; `LazyVStack`/`LazyHStack` for long lists | `SOURCE` |
| Image handling | Downsampled, not full-resolution in memory; async loading | `SOURCE` |
| Background work | `BGTaskScheduler`, not `Timer`/`DispatchQueue` hacks; `UIBackgroundModes` justified | `SOURCE` |
| Battery | No unnecessary location/sensor polling; push over poll | `SOURCE` (definitive needs `RUNTIME` profiling) |
| App size | Asset catalog with slicing/thinning; on-demand resources for large assets | `BUILD` |
