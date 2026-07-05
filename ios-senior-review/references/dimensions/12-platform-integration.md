# Dimension 12: Platform Integration Depth

Tier 4 — Platform Opportunity (advisory). Never produces `[R]` or `[W]` — only `[~]` recommendations or `[+]` praise. Flag only natural fits.

| Check | When applicable |
|---|---|
| Widgets | App has at-a-glance content (status, progress, quick actions) |
| Spotlight indexing | App has searchable content users would find from home screen |
| App Intents / Siri | App has discrete actions users would voice-trigger or add to Shortcuts |
| Haptics | App has mutations, async outcomes, selections benefiting from tactile feedback |
| ShareLink / Transferable | App has content worth sharing; context menus with share actions |
| Live Activities | App has real-time status users monitor (delivery, sports, timers) — anti-spam rules in Dimension 1 (4.5.3) still apply |
| Quick Actions | App has 2-4 common entry points for home screen long-press |
| Context menus | Interactive items support `.contextMenu` for long-press/right-click (iPad/Mac) |
| Drag and drop | Content items conform to `Transferable`; `.draggable`/`.dropDestination` where natural |
| Keyboard shortcuts | iPad/Mac targets support `.keyboardShortcut` for common actions |
| NSUserActivity | Detail screens donate `NSUserActivity` for Spotlight and Handoff |
| TipKit | Feature discovery tips for non-obvious gestures or features |

**Platform-specific targets (advisory, conditional):** when the scope includes a watchOS companion, tvOS, or visionOS target, add the platform basics — watchOS: complications/widgets present for glanceable data, workout/background session hygiene; tvOS: focus-engine behavior, top-shelf content; visionOS: ornaments over floating chrome, immersive-space lifecycle. Same `[~]`/`[+]`-only rule as the rest of Tier 4.
