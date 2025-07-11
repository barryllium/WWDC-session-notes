# [**Make your UIKit app more flexible**](https://developer.apple.com/videos/play/wwdc2025/282)

---

### **Scenes**

* A scene is an instance of your app’s UI
* It contains your app’s view controllers and views
* Scenes provide hooks for handling external data, like a URL for deep linking to a section of your app’s UI
* Each scene independently saves and restores UI state. A scene determines the best opportunities to ask for the current state, before persisting it to disk
    * You can query the previous UI state when a scene reconnects
* Scenes also provide context on how your app is displayed, including details about the screen, and the window’s geometry
* You can have multiple scenes, each with their own lifecycle and state
* In iOS 26, you can now mix SwiftUI and UIKit scenes in a single app
* [What’s new in UIKit](./What's%20new%20in%20UIKit.md) Session
* `UIScene` life cycle will be required after iOS 26
* Supporting multiple scenes is encouraged
* [Migrating to the UIKit scene-based life cycle](https://developer.apple.com/documentation/technotes/tn3187-migrating-to-the-uikit-scene-based-life-cycle) Tech Document

* It is the responsibility of the app delegate to determine the scene configuration for a connecting session
    * In the `configurationForConnectingSceneSession` delegate method, you check the scene session’s role

```swift
// Specify the scene configuration

@main
class AppDelegate: UIResponder, UIApplicationDelegate {

    func application(_ application: UIApplication,
                     configurationForConnecting sceneSession: UISceneSession,
                     options: UIScene.ConnectionOptions) -> UISceneConfiguration {

        if sceneSession.role == .windowExternalDisplayNonInteractive {
            return UISceneConfiguration(name: "Timer Scene",
                                        sessionRole: sceneSession.role)
        } else {
            return UISceneConfiguration(name: "Main Scene",
                                        sessionRole: sceneSession.role)
        }
    }
}
```

* UISceneDelegate manages the life cycle of an individual scene
    * If your scene configuration specifies a storyboard, window creation happens automatically

```swift
// Configure the UI

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    var timerModel = TimerModel()

    func scene(_ scene: UIScene,
               willConnectTo session: UISceneSession,
               options connectionOptions: UIScene.ConnectionOptions) {

        let windowScene = scene as! UIWindowScene
        let window = UIWindow(windowScene: windowScene)
        window.rootViewController = TimerViewController(model: timerModel)
        window.makeKeyAndVisible()
        self.window = window
    }
}
```

* Can track app lifecycle states within the `SceneDelegate`

```swift
// Handle life cycle events

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    var timerModel = TimerModel()

    // ...

    func sceneDidEnterBackground(_ scene: UIScene) {
        timerModel.pause()
    }
}
```

* State restoration within `SceneDelegate` to restore scenes to a previous state

```swift
// Restore UI state

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    var timerModel = TimerModel()

    // ...

    func stateRestorationActivity(for scene: UIScene) -> NSUserActivity? {
        let userActivity = NSUserActivity(activityType: "com.example.timer.ui-state")
        userActivity.userInfo = ["selectedTimeFormat": timerModel.selectedTimeFormat]
        return userActivity
    }

    func scene(_ scene: UIScene restoreInteractionStateWith userActivity: NSUserActivity) {
        if let selectedTimeFormat = userActivity?["selectedTimeFormat"] as? String {
            timerModel.selectedTimeFormat = selectedTimeFormat
        }
    }
}
```

### **Container view controllers**

* A container view controller is responsible for managing the layout of one or more child view controllers

#### UISplitViewController

* `UISplitViewController` manages the display of multiple adjacent columns of content, supporting navigation throughout a hierarchy of information
* When horizontal space is limited, the split view controller adapts by collapsing its columns into a navigation stack
* `UISplitViewController` gains new features, like interactive column resizing
    * You can now resize columns by dragging the split view controller’s separators
    * When using the pointer, its shape will adapt to indicate the directions in which a column can be resized
    * `UISplitViewController` provides a default minimum, maximum, and preferred width for each column
    * You can customize the minimum, maximum, and preferred widths of each column using their associated split view controller properties
        * Be careful not to require a width that limits the number of columns that can be displayed

```swift
// Customize the minimum, maximum, and preferred column widths

let splitViewController = // ...

splitViewController.minimumPrimaryColumnWidth = 200.0
splitViewController.maximumPrimaryColumnWidth = 400.0
splitViewController.preferredSupplementaryColumnWidth = 500.0
```

* In Mail, disclosure indicators are shown when the split view controller is collapsed, to convey additional content can be revealed upon cell selection
* A new trait, `splitViewControllerLayoutEnvironment`, conveys whether an ancestor split view controller is expanded or collapsed

```swift
// Adapt for the split view controller layout environment

override func updateConfiguration(using state: UICellConfigurationState) {
   
    // ...
    
    if state.traitCollection.splitViewControllerLayoutEnvironment == .collapsed {
        accessories = [.disclosureIndicator()]
    } else {
        accessories = []
    }
}
```

* New first-class support for inspector columns
    * An inspector is a column within a split view controller that provides additional details of the selected content
* When the split view controller is expanded, the inspector column resides on the trailing edge, adjacent to the secondary column
* When collapsed, the split view controller adapts automatically, and presents the inspector column as a sheet

```swift
// Show an inspector column

let splitViewController = // ... 
splitViewController.setViewController(inspectorViewController, for: .inspector)

splitViewController.show(.inspector)
```

#### Tab bar controller

* `UITabBarController` categorizes app into sections
* Enables quick switching between tabs, while preserving the current state within each pane
* The appearance of the tab bar adapts for each platform
    * On iPhone, the tab bar is located at the bottom of the scene
    * On Mac, the tab bar can reside in the toolbar or can be displayed as a sidebar
    * On Apple Vision Pro, the tab bar is displayed in an ornament on the leading edge of the scene
    * On iPad, the tab bar resides at the top of the scene alongside navigation controls
        * The tab bar can also adapt into a sidebar, allowing quick access to collections of content
* Tab groups surface additional destinations in the sidebar
    * For example, in the Music app on iPad, the Library tab group includes Artists, Albums, and more
        * When the sidebar is not available, the Library group is a tab destination
* `UITabBarController` offers API to seamlessly manage this adaptation
    * Provide the tab group with a managing navigation controller
    * When a leaf tab of the tab group is selected, its view controller, along with the view controllers of its ancestor groups, are pushed onto this navigation stack
    * To customize the view controllers pushed onto this navigation stack, implement the `UITabBarController` delegate method, `displayedViewControllersFor tab`
* [Elevate your tab and sidebar experience in iPadOS](../2024/Elevate%20your%20tab%20and%20sidebar%20experience%20in%20iPadOS.md) Session from WWDC 2024

```swift
// Managing tab groups

let group = UITabGroup(title: "Library", ...)
group.managingNavigationController = UINavigationController()

// ...

// MARK: - UITabBarControllerDelegate

func tabBarController(
    _ tabBarController: UITabBarController,
    displayedViewControllersFor tab: UITab,
    proposedViewControllers: [UIViewController]) -> [UIViewController] {

    if tab.identifier == "Library" && !self.allowsSelectingLibraryTab {
        return []
    } else {
        return proposedViewControllers
    }
}
```

* While `UISplitViewController` and `UITabBarController` are designed to support a wide range of sizes, your app may require a minimum size to maintain core functionality
    * Use the `UISceneSizeRestrictions` API to express the preferred minimum size of your scene’s content
    * The best time to specify the minimum size is when the scene is about to connect

```swift
// Specify a preferred minimum size

class SceneDelegate: UIResponder, UIWindowSceneDelegate {

    func scene(_ scene: UIScene,
               willConnectTo session: UISceneSession,
               options connectionOptions: UIScene.ConnectionOptions) {

        let windowScene = scene as! UIWindowScene
        windowScene.sizeRestrictions?.minimumSize.width = 500.0
    }
}
```

### **Adaptivity**

* A crucial step in making your UI adaptable is to ensure that content remains within the safe area
    * The safe area is a region within a view that is appropriate for interactive or important content
    * Content placed outside of this region is vulnerable to getting covered, such as by a navigation bar or a toolbar
    * Content could also be occluded by system UI like the status bar, or even device features, like the Dynamic Island
* The sidebar adds a non-symmetrical safe area inset to the adjacent column in a split view controller
    * The background can freely extend outside of the safe area, underneath the sidebar
    * Content is positioned within the safe area to ensure that is remains visible
        * Content is inset from the edges of the safe area using layout margins
* Each view provides layout guides to apply standard margins around content
    * Layout margins are inset from the safe area by default

```swift
// Position content using the layout margins guide

let containerView = // ...
let contentView = // ...

let contentGuide = containerView.layoutMarginsGuide

NSLayoutConstraint.activate([
    contentView.topAnchor.constraint(equalTo: contentGuide.topAnchor),
    contentView.leadingAnchor.constraint(equalTo: contentGuide.leadingAnchor),
    contentView.bottomAnchor.constraint(equalTo: contentGuide.bottomAnchor)
    contentView.trailingAnchor.constraint(equalTo: contentGuide.trailingAnchor)
])
```

* In iPadOS 26, scenes gain a new control to close, minimize, and arrange the window, similar to macOS
* The window control appears alongside the content in your scene
* A scene can specify a preferred windowing control style to compliment its content
    * To specify a preference, implement the `UIWindowSceneDelegate` method `preferredWindowingControlStyle for scene`

```swift
// Specify the window control style

class SceneDelegate: UIResponder, UIWindowSceneDelegate {

    func preferredWindowingControlStyle(
        for scene: UIWindowScene) -> UIWindowScene.WindowingControlStyle {
        return .unified
    }
}
```

* System components, such as UINavigationBar, adapt automatically by arranging their subviews around the window control
    * Your UI should also adapt to the window control, regardless of its style
    * To ensure that your UI is not occluded, use a layout guide that accounts for the window control

```swift
// Respect the window control area

let containerView = // ...
let contentView = // ...

let contentGuide = containerView.layoutGuide(for: .margins(cornerAdaptation: .horizontal)

NSLayoutConstraint.activate([
    contentView.topAnchor.constraint(equalTo: contentGuide.topAnchor),
    contentView.leadingAnchor.constraint(equalTo: contentGuide.leadingAnchor),
    contentView.bottomAnchor.constraint(equalTo: contentGuide.bottomAnchor),
    contentView.trailingAnchor.constraint(equalTo: contentGuide.trailingAnchor)
])
```

* When your UI is adaptive, the interface orientation should be redundant
    * Scene resizing, device rotation, and changes to window layout, all ultimately result in a modification to your scene’s size
    * Certain categories of apps may benefit from temporarily locking the orientation
        * e.g. a driving game may want to lock the orientation when the device is expected to rotate for steering a vehicle
* When a view controller is visible, it can prefer a locked interface orientation
    * To specify a preference, override `prefersInterfaceOrientationLocked` in your view controller subclass
    * Whenever this preference changes, call `setNeedsUpdateOfPrefersInterfaceOrientationLocked`
    * To observe the interface orientation lock, implement the `UIWindowSceneDelegate` method `didUpdateEffectiveGeometry`, and compare whether the value of `isInterfaceOrientationLocked` has changed

```swift
// Request orientation lock
class RaceViewController: UIViewController {

    override var prefersInterfaceOrientationLocked: Bool {
        return isDriving
    }

    // ...

    var isDriving: Bool = false {
        didSet {
            if isDriving != oldValue {
                setNeedsUpdateOfPrefersInterfaceOrientationLocked()
            }
        }
    }
}

// Observe the interface orientation lock
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var game = Game()

    func windowScene(
        _ windowScene: UIWindowScene,
        didUpdateEffectiveGeometry previousGeometry: UIWindowScene.Geometry) {
        
        let wasLocked = previousGeometry.isInterfaceOrientationLocked
        let isLocked = windowScene.effectiveGeometry.isInterfaceOrientationLocked

        if wasLocked != isLocked {
    game.pauseIfNeeded(isInterfaceOrientationLocked: isLocked)
        }
    }
}
```

* Apps should respond quickly to being resized
    * There may be elements of your app’s UI that are computationally expensive to draw
    * Re-rendering assets for every size within a resize interaction is unnecessary
    * In the example below, `isInteractivelyResizing` is queried to only update assets for a new scene size after the interaction finishes

```swift
// Query whether the scene is resizing

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var gameAssetManager = GameAssetManager()
    var previousSceneSize = CGSize.zero

    func windowScene(
        _ windowScene: UIWindowScene,
        didUpdateEffectiveGeometry previousGeometry: UIWindowScene.Geometry) {

        let geometry = windowScene.effectiveGeometry
        let sceneSize = geometry.coordinateSpace.bounds.size

        if !geometry.isInteractivelyResizing && sceneSize != previousSceneSize {
            previousSceneSize = sceneSize
            gameAssetManager.updateAssets(sceneSize: sceneSize)
        }
    }
}
```

* The `UIRequiresFullscreen` Info.plist key is a compatibility mode from iOS 9 that prevents scene resizing
    * `UIRequiresFullscreen` is deprecated and will be ignored in a future release

#### Future hardware compatibility

* Previously, when new hardware was released with a different screen size, the system would scale or letterbox your app’s UI
    * The scaling would stay in place until you built with a newer SDK and resubmitted your app
* Once you build and submit with the iOS 26 SDK, the system will no longer scale or letterbox your app’s UI for a new screen size
