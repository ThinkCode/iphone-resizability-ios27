# iOS 27 Resizability Implementation Guide

**Adapted from:** Jacob Techtavern (@jacobtechtavern)
**Original source:** https://x.com/jacobtechtavern/status/2105918220733436369

> This guide restructures and expands the original iOS 27 resizability guidance into an agent-friendly implementation checklist for iOS codebases.

## 1) Scene lifecycle migration

Check whether the app already uses the SceneDelegate lifecycle.

- SwiftUI app lifecycle: already compliant
- UIKit app lifecycle:
  - Add `UIApplicationSceneManifest` to `Info.plist`
  - Move `var window: UIWindow?` from `AppDelegate` to `UIWindowSceneDelegate`

### Required migration pattern

```swift
// AppDelegate
// Remove: var window: UIWindow?

// SceneDelegate
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
}
```

If this has not been completed, the app will fail to launch against the iOS 27 SDK.

---

## 2) Remove screen/device assumptions

Use available space, not device identity.

### Replace these APIs

| Remove | Replace with |
|---|---|
| `UIDevice.current.userInterfaceIdiom` | `horizontalSizeClass` / `verticalSizeClass` |
| `UIScreen.main.bounds` | `windowScene.effectiveGeometry` or the view's own `bounds` |
| `UIScreen.main.scale` | `traitCollection.displayScale` |
| `UIScreen.main` for layout decisions | `window?.windowScene?.screen` or view geometry |
| `UIApplication.shared.windows` / `keyWindow` | The scene's own `window` |

### Preferred approach

Prefer the size of the current view or container over the screen size.

```swift
// Good: based on actual available space
let height = view.bounds.height
let width = view.bounds.width
```

Bad patterns:

```swift
let screenWidth = UIScreen.main.bounds.width
let device = UIDevice.current.userInterfaceIdiom
```

### React to geometry updates

```swift
func windowScene(_ windowScene: UIWindowScene,
                 didUpdate previousCoordinateSpace: UICoordinateSpace,
                 interfaceOrientationChanged previousInterfaceOrientation: UIInterfaceOrientation) {
    // update layout when effective geometry changes
}
```

### Trait tracking

```swift
registerForTraitChanges([UITraitHorizontalSizeClass.self]) {
    self.invalidateLayout()
}
```

Also remove:
- fixed sizes like `.frame(width: 390, height: 844)`
- hardcoded safe-area inset assumptions

---

## 3) Stop basing layout on orientation

Orientation is no longer a useful layout input in resizable environments.

- `supportedInterfaceOrientations` is mostly a preference
- In iPhone Mirroring, the app runs in portrait even if the scene is wide
- Orientation lock prevents rotation, not resizing

Use size classes instead:

```swift
if horizontalSizeClass == .compact {
    // single-column layout
} else {
    // wider layout
}
```

Remove code like:

```swift
if UIDevice.current.orientation == .landscape {
    // layout decision
}
```

### `UIRequiresFullScreen`

This is deprecated for app development in iOS 27.

- Remove from `Info.plist` for non-game apps
- Keep only for games when needed

---

## 4) Resize-test the app thoroughly

Resize to extremes and verify the app remains stable.

Check these cases:
- narrowest width
- widest width
- shortest height
- tallest height
- roughly square scene

Look for:
- clipped content
- overlapping bars
- views outgrowing parents
- state lost during resize

### Testing tools

- Xcode 27 Live Previews with resize handles
- Device Hub resize mode
- iPhone Mirroring on Mac
- Run the iPhone app on iPad

---

## 5) Make layouts adapt to available space

Instead of checking device type, adapt to the current container size.

### Good patterns

- `Size classes` for high-level decisions
- `ViewThatFits` for choosing the first layout that fits
- `GeometryReader` and `containerRelativeFrame` for proportional drawing
- `NavigationSplitView` / `UISplitViewController` for compact vs regular behavior
- toolbar item priorities and groups to gracefully drop content
- `readableContentGuide` and `.frame(maxWidth:)` for text width caps

### Example: layout selection by size class

```swift
if horizontalSizeClass == .compact {
    TabView { ... }
} else {
    HStack {
        Sidebar()
        Content()
    }
}
```

### Example: ViewThatFits

```swift
ViewThatFits {
    HStack { ... }  // preferred, if it fits
    VStack { ... }  // fallback
}
```

### Example: respond to size-class changes instead of every size change

```swift
.onChange(of: horizontalSizeClass) { newValue in
    // only react at breakpoints
}
```

Avoid:

```swift
.onChange(of: geometry.size) { _ in
    // too frequent and noisy
}
```

---

## 6) Sidebar behavior on iPhone (iOS 27)

Tab bars can expand into a sidebar on iPhone.

Use:

```swift
sidebar.preferredPlacement = .sidebar
```

Check whether it is available:

```swift
sidebar.isAvailable
```

If not available, surface the content through nested tabs or alternate navigation patterns.

---

## 7) Agent-friendly implementation checklist

Use this checklist in a code review or implementation pass.

```text
[ ] Moved UIKit app from AppDelegate to SceneDelegate lifecycle
[ ] Removed UIDevice screen checks and screen-size assumptions
[ ] Replaced UIScreen usage with view bounds / scene geometry
[ ] Replaced orientation checks with size classes
[ ] Removed UIRequiresFullScreen for non-game apps
[ ] Tested narrowest, widest, shortest, tallest, and square layouts
[ ] Verified no clipped content or overlapping bars
[ ] Confirmed state survives resize
[ ] Updated layouts to use size classes and adaptive containers
[ ] Replaced hardcoded frames and safe-area assumptions
[ ] Used ViewThatFits / GeometryReader / split views where appropriate
```

---

## 8) Recommended implementation strategy for agents

When working on an iOS codebase, agents should prioritize in this order:

1. Replace screen and device-based logic with size-class and view-bounds logic
2. Move UIKit apps to scene-based lifecycle
3. Remove orientation-dependent logic
4. Test resize extremes with device hub / simulator / mirroring
5. Convert fixed layouts to adaptive layouts using `ViewThatFits`, `NavigationSplitView`, and size classes

---

## 9) Xcode skill note

Xcode 27 includes an app modernization skill that can handle much of this automatically.

Try:

```bash
xcrun agent skills export
```

It can help with:
- scene lifecycle modernization
- replacing `UIScreen`/`orientation` logic
- adapting layouts to size classes and available space

---

## 10) Summary

The main rule for iOS 27 resizability is simple:

> Adapt to available space, not the device or screen size.

If an app follows the scene lifecycle and removes hardcoded device/screen assumptions, most resizability issues disappear.

---

## Credits & Attribution

This guide is adapted from original iOS 27 resizability guidance shared by Jacob Techtavern.

- Original author: Jacob Techtavern (@jacobtechtavern)
- Original post: https://x.com/jacobtechtavern/status/2105918220733436369
- Adaptation purpose: restructuring the guidance into a practical agent-friendly implementation checklist for iOS development
