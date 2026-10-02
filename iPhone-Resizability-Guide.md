# iPhone Resizability Guide

On iOS 27, iPhone apps are fully resizable - in iPhone Mirroring on Mac, and when an iPhone-only app runs on iPad. Your app is expected to adjust dynamically to different scene sizes at runtime.

Please work through the four questions below. Most issues are resolved by questions 1 and 2 alone.

## 1. Have you migrated from the AppDelegate lifecycle to the SceneDelegate lifecycle?

If not, your app will not launch when built against the iOS 27 SDK. SwiftUI App lifecycle apps are already done - move to question 2. UIKit apps need `UIApplicationSceneManifest` in Info.plist, and `var window: UIWindow?` moved from AppDelegate to a UIWindowSceneDelegate.

## 2. Have you removed these APIs?

Each one asks what device you are on, or how big the screen is. The only reliable question is how much space your view has right now.

| Remove | Replace with |
|--------|--------------|
| `UIDevice.current.userInterfaceIdiom` | Size classes |
| `UIScreen.main.bounds` | `windowScene.effectiveGeometry`, or the view's own size |
| `UIScreen.main.scale` | `traitCollection.displayScale` |
| `UIScreen.main` (any other use) | `window?.windowScene?.screen` |
| `UIApplication.shared.windows` / `keyWindow` | The scene's own window |

### Best Practices

- **Prefer your view's own size over scene geometry.** Using the size of your view controller's view, or your view's superview, makes your UI less dependent on how it's presented - it adapts better when the same view controller appears in another context, such as inside a split view controller.

- **Monitor available space** with `windowScene(_:didUpdateEffectiveGeometry:)` in your scene delegate when you need to react to changes.

- **Common override points are tracked for you.** The system tracks which trait collection properties you read inside methods such as `layoutSubviews`, or `drawRect`, and calls them again when a tracked trait changes. Use `registerForTraitChanges` where automatic tracking isn't available, to invalidate caches or update related data.

- **Also worth removing:**
  - Fixed frames such as `.frame(width: 390, height: 844)`
  - Hardcoded safe-area insets

## 3. Interface Orientation

Interface orientation is no longer useful for layout decisions. In iOS 27, your app's supported interface orientation is a preference you provide to the system, and it's ignored in resizable environments. In iPhone Mirroring, your app always runs in the portrait interface orientation, regardless of the aspect ratio of its scene.

- Update any orientation checks to use size classes instead
- An orientation lock prevents rotation, not resizing
- Your app can be wider than it is tall even if it only supports portrait

### UIRequiresFullScreen Deprecation

`UIRequiresFullScreen` is deprecated for apps in iOS 27. If you are having trouble freely resizing your window to any size, remove this Info.plist key and stop using it.

**Exception:** Games are still supported. Full resizing is still preferred, but where that isn't possible, `UIRequiresFullScreen` remains valid for games. It honours your supported interface orientations, and resizing happens discretely - snapping between sizes rather than interpolating smoothly.

## 4. Have you resized your app, and does it hold up?

Resize your app to its extremes and check each one:
- Narrowest
- Widest
- Shortest
- Tallest
- Roughly square

Look for:
- Clipped content
- Overlapping bars
- Views that outgrow their container
- State that doesn't survive a resize

### Testing Tools

- **Xcode 27 Live Previews:** Have resize handles, so you can iterate on layout quickly
- **Device Hub:** Offers a resize mode - enter it, then drag the edges of the device to resize freely. Run your iPhone app here, not your iPad app - it's your iPhone app's resizing behaviour you're testing.
- **Real devices:** Test on iPhone Mirroring on Mac, and your app on iPad

## 5. Do your layouts adapt as the window grows and shrinks?

If not, adapt to available space rather than to a device:

- **Size classes** decide what to show - tabs or sidebar, one column or multiple
- **ViewThatFits** picks the first of several layouts that fits the available space. It measures without rendering, so no size class logic is needed
- **GeometryReader, containerRelativeFrame** decide how big to draw it
- **NavigationSplitView / UISplitViewController** handle compact and regular for free
- **Toolbar item priorities and groups** let bars shed content gracefully as width shrinks
- **readableContentGuide or .frame(maxWidth:)** cap line length so text doesn't sprawl at wide sizes

### Tab Bars and Sidebars

Tab bars can expand into a sidebar on iPhone, new in iOS 27. Unlike iPad, this is an app choice:
- Set `sidebar.preferredPlacement` to `.sidebar`
- Use `sidebar.isAvailable` to check whether it can currently be shown
- There's no way to toggle between the two in the UI - the system decides based on available space, so surface the content behind nested tabs elsewhere in your app when a sidebar isn't available

### Size Class Change Handling

- **React to size class changes** rather than to every size change
- Prefer `.onChange(of: horizontalSizeClass)` over `.onChange(of: geometry.size)` when a layout decision only changes at a breakpoint
- Reach for `horizontalSizeClass` when the two layouts are genuinely different - a TabView in compact versus a sidebar in regular, for example
- Avoid branching between two container types that already adapt, since swapping view identity mid-resize discards navigation state and the collapse animation

## Xcode Skill

Xcode 27 includes an app modernization skill that understands these adaptivity tasks. Ask an agent to make your app more adaptable and it handles much of questions 1 and 2 for you, replacing screen and orientation references with size classes and scene geometry, and converting your app to scene lifecycle. To use the skill outside Xcode, export it with `xcrun agent skills export`. Try this before working through the list by hand.
