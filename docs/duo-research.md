# iPhone Duo and Readium API research

Checked on 2026-10-25. Apple facts come from Apple's own documentation data (developer.apple.com, the JSON behind the doc pages). Readium facts come from the swift-toolkit source at tag `3.11.1`. Re-check the Apple items against the SDK headers in Xcode before you write code that uses them.

## Readium

- **Pin `3.11.1`** (released 2026-10-16). It's the newest stable tag; `4.0.0-alpha.2` exists but is an alpha.
- 3.11.1's changelog says: "Fixed reflowable EPUBs being displayed as a single column on the iPhone Duo unfolded in landscape, by porting the responsive columns fix of Readium CSS 1.1.2." Anything older will show the wrong layout on the Duo.
- **`ReadiumAdapterGCDWebServer` is not needed.** The `EPUBNavigatorViewController` initializers that take `httpServer:` are deprecated with the message "The HTTP server is no longer needed for the EPUB navigator."
- **Two-page layout for reflowable books is `EPUBPreferences.columnCount`** (`.auto`, `.one`, `.two`). `spread` only applies to fixed-layout books. In the source, `spread: .auto` means "landscape", based on the view's aspect ratio (`size.width > size.height`), not on its width.
  - Pages setting → Readium: auto → `columnCount: .auto`, one → `.one`, two → `.two`. Readium CSS chooses the column count from the viewport width, so it follows the fold without any code from us. We still have to confirm on the simulator that `.auto` gives one column when folded and two when unfolded. If it doesn't, set `.one` or `.two` ourselves from the view's width.
  - For fixed-layout books, map auto → `spread: .auto`, one → `.never`, two → `.always`.
- Readium's own minimum is iOS 15, so an iOS 27 target is fine.

## Fold and hinge (iOS **27.1**, not 27.0)

Every hinge API below says "iOS 27.1" in Apple's docs, so **Squire's minimum target needs to be iOS 27.1**, or the hinge code needs `#available(iOS 27.1, *)` checks.

- SwiftUI: `func onHingeChange(isEnabled: Bool = true, _ action: @escaping (DeviceHingeContext, DeviceHingeContext) -> Void) -> some View`, documented as "Adds an action to perform when the hinge context of the view hierarchy changes."
- `struct DeviceHinge`: `angle`, `status` (`DeviceHinge.Status`: `.closed`, `.partiallyOpen`, `.fullyOpen`). Reached through `DeviceHingeContext` (third-party samples use `context.hinge`, which may be optional; check the header).
- UIKit: `@MainActor class UIHingeInteraction`, `init(updateHandler:)`, `isEnabled`, `Update`. "When the interaction moves out of a hierarchy that provides hinge updates, the update's `hinge` is nil."
- Layout: `GeometryProxy.reservedRegions(kind: .division, ...)` returns the fold's frame. I took this from third-party samples and haven't checked it against Apple's docs yet.

**Recommendation:** don't drive page layout from the hinge. Use the available width (Readium's `columnCount: .auto`, or our own width check as a fallback). Width-based layout handles the Duo folded and unfolded, iPad, Split View and Stage Manager with one code path, and it matches Apple's guidance to use the hinge for interactions and effects, not layout. Use the hinge only if a half-open (`.partiallyOpen`) "book stand" posture turns out to be worth adding.

## Cover (outer) display when closed: no public API fits

- `ExternalNonInteractiveAccessory` (iOS 27.0): "presented when an external display is connected to the device, or when the device is connected to an external display via AirPlay." That covers external displays, not the cover screen.
- `CameraCaptureAccessory` (iOS 27.1): "may be presented while the app is in the foreground and has an active camera capture session." This is how an app shows content on the cover display, but only while the camera is running. It doesn't fit a reader.
- According to third-party simulator reports, folding the device moves the app itself to the outer display (compact width). So "closed mid-book" most likely means Squire keeps running there as a one-page reader. Feature 3 (keep the position across a fold) covers that case.

**Closest alternatives** (none built yet; noted in `IDEAS.md`):

1. Treat the folded outer display as a first-class one-page reader. This comes free if layout follows width.
2. A Lock Screen or Home Screen widget (WidgetKit) showing the current book's cover and progress. Doing this needs an app group and a widget extension target. That isn't a new package, but it's worth agreeing on first.

## Keeping the position across fold and unfold

Readium reloads its spreads when the viewport or column settings change. Plan: before a layout change, keep the navigator's `currentLocation`, then after the reload `go(to:)` that locator if the visible position has drifted. Test this on the simulator by folding and unfolding repeatedly in the middle of a long chapter.
