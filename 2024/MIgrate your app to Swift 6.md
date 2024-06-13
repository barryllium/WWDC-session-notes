# [**Migrate your app to Swift 6**](https://developer.apple.com/videos/play/wwdc2024-10169)

---

* Swift 6 language mode introduces full enforcement of data isolation
* When to migrate to Swift 6
    * You're experiencing hard-to-reproduce crashes
    * You're integrating more concurrency into your app
    * You're maintaining a library that is used from concurrent code
* Track adoption of Swift 6 in popular packages on [swiftpackageindex.com](https://swiftpackageindex.com)
* Data race safety allows you to leverage concurrency in your app without fear of introducing new data races later
    * Helps with crashes that cannot be reproduced

### **Steps**

* Compile with Xcode 16to make sure the app compiles properly
* Enable `Complete Concurrency Checking`
    * Per-module setting that leaves your project in Swift 5 mode, but enables warnings for all the code that would fail with Swift 6's enforced data isolation
    * Resolve all warnings
* Enable Swift 6 mode
    * Locks in all changes, and prevents any future refactoring from regressing to an unsafe state
* Move onto the next target and repeat the process
* Audit unsafe opt-outs
* Try not to blend significant refactoring and blending data race safety

### Complete Concurrency Checking

```swift
//Define Recaffeinator class
class Recaffeinater: ObservableObject {
    @Published var recaffeinate: Bool = false
    var minimumCaffeine: Double = 0.0
}

//Add protocol to notify if caffeine level is dangerously low
extension Recaffeinater: CaffeineThresholdDelegate {
    public func caffeineLevel(at level: Double) {
        if level < minimumCaffeine {
            // TODO: alert user to drink more coffee!
        }
    }
}
```

* In the above code, we are going to publish a value back to a SwiftUI View, so we want it to be isolated to the Main Actor:

```swift
//Isolate the Recaffeinater class to the main actor
@MainActor
class Recaffeinater: ObservableObject {
    @Published var recaffeinate: Bool = false
    var minimumCaffeine: Double = 0.0
}
```

* Doing, so, though, gives a warning in the `caffeineLevel` function
    * `Main actor-isolated instance method 'caffeineLevel(at:)' cannot be used to satisfy nonisolated protocol requirement` on the function declaration
    * `CaffeineThresholdDelegate` protocol makes no guarantees about how it's going to be called
        * It's inside a module that has not been updated to Swift 6 yet
        * We are conforming `Recaffeinater` to this protocol
        * Because `Recaffeinater` is constrained to the Main Actor, it can't just conform to a protocol that isn't always guaranteed to be called on the main actor
    * Will come back to this later, but this is an example of an error that the swift compiler generates because of opting a type into checking that it was being called on the Main Actor.
* Next, turn on Complete Concurrency checking for the main target (not CoffeeKit) and build
    * This produces three more warnings, the first of which is on our `Logger` initialization

```swift
//var 'logger' is not concurrency-safe because it is non-isolated global shared mutable state; this is an error in the Swift 6 language mode
var logger = Logger(
    subsystem:
        "com.example.apple-samplecode.Coffee-Tracker.watchkitapp.watchkitextension.ContentView",
    category: "Root View")
```

* We can expand the issue to see possible fixes
    * The first option is to make `logger` read-only, by switching `var` to `let` (the correct solution here)
    * Could also have tied the variable to `@MainActor`
    * Could also use `nonisolated(unsafe)`
        * Puts the burden on the developer to ensure safety
        * Should be a last resort
        * A way to temporarily resolve a warning but to come back and fully resolve later
* Global variables are initialized lazily in Swift
    * Different than C and Obj-C, where they are initialized on startup
    * Lazy initialization avoids launch time issues
    * Also can introduce race conditions
        * Swift handles this by creating global variables atomically
* The next warning is where we call the `shared()` method on `WKApplication`

```swift
func scheduleBackgroundRefreshTasks() {

    scheduleLogger.debug("Scheduling a background task.")

    // Get the shared extension object.
    let watchExtension = WKApplication.shared() //warning: Call to main actor-isolated class method 'shared()' in a synchronous nonisolated context

    // If there is a complication on the watch face, the app should get at least four
    // updates an hour. So calculate a target date 15 minutes in the future.
    let targetDate = Date().addingTimeInterval(15.0 * 60.0)

    // Schedule the background refresh task.
    watchExtension.scheduleBackgroundRefresh(withPreferredDate: targetDate, userInfo: nil) { //warning: Call to main actor-isolated instance method 'scheduleBackgroundRefresh(withPreferredDate:userInfo:scheduledCompletion:)' in a synchronous nonisolated context
        error in

        // Check for errors.
        if let error {
            scheduleLogger.error(
                "An error occurred while scheduling a background refresh task: \(error.localizedDescription)"
            )
            return
        }

        scheduleLogger.debug("Task scheduled!")
    }
}
```

* The first possible fix would be to handle things asynchronously
    * Could use `await` if it were an async function
    * Could make the `scheduleBackgroundRefreshTasks()` function async
    * Could wrap the call in `Task { ... }`
* Instead, since this is a free function (not a method on our view), it's not defaulting to the Main Actor
    * We can apply `@MainActor` and the warning is resolved (also resolves the second warning in the code above)
    * This works because we can look at the callers, and both are known to be run on the Main Actor
        * Would have gotten a compiler error otherwise
        * One caller is `WKApplicationDelegate`, and if we option-click on it, we see that it's tied to the Main Actor
* The latest SDKs shipping with Xcode 16 have more delegates and protocols annotated with `@MainActor`, so it will show significantly less warnings

#### Callbacks

* Some callbacks have a guarantee that their callbacks will always be on the Main thread
    * A lot of UI frameworks give this guarantee
* Some delegates (like HealthKit APIs) do the opposite, making no guarantees
    * You need to re-dispatch onto the right queue or actor or do work in a thread-safe way in this scenario
* Swift Concurrency makes these guarantees (or lack of guarantees) explicit
    * If a callback doesn't specify how it's called back, it's considered non-isolated
        * Can't access data that requires a certain isolation
    * If a callback does provide an isolation guarantee, then it can annotate the delegate protocol or callback as always being on the main actor
        * The receiver can then rely on that guarantee
* Back to the first warning now

```swift
//This class is guaranteed on the main actor...
@MainActor
class Recaffeinater: ObservableObject {
    @Published var recaffeinate: Bool = false
    var minimumCaffeine: Double = 0.0
}

//...but this protocol is not
//warning: Main actor-isolated instance method 'caffeineLevel(at:)' cannot be used to satisfy nonisolated protocol requirement
extension Recaffeinater: CaffeineThresholdDelegate {
    public func caffeineLevel(at level: Double) {
        if level < minimumCaffeine {
            // TODO: alert user to drink more coffee!
        }
    }
}
```

* We are given a couple of options to fix the issue
    * The first is to declare `caffeineLevel` nonisolated
        * Means that despite this being a method on a main actor isolated type, this specific method will not be isolate to the main actor
        * This is the correct route for callbacks that make no promises about where they call back from
        * For this specific case, we would want to immediately do work on the main actor, otherwise we would get a compiler error
            * `Main actor-isolated property 'minimumCaffeine' can not be referenced from a non-isolated context`
        * Could fix this by wrapping the function content in a Task...

```swift
nonisolated public func caffeineLevel(at level: Double) {
    Task { @MainActor in
      if level < minimumCaffeine {
        // TODO: alert user to drink more coffee!
        }
    }
}
```

* What we *really* want, though, is to guarantee that the callback comes on the main actor
    * Because the code is inside CoffeeKit, we can update the protocol to return on `@MainActor`
    * But we *don't* own that codebase and cannot make that change, we can use `assumeIsolated`

```swift
nonisolated public func caffeineLevel(at level: Double) {
    MainActor.assumeIsolated {
        if level < minimumCaffeine {
            // TODO: alert user to drink more coffee!
        }
    }
}
```

* `assumeIsolated` is great for when we know for certain that the method is getting called on the main actor (by checking code or reading documentation)
    * This doesn't start a new task to async onto the main actor, it just tells Swift that this code is already running on the main actor
    * `assumeIsolated` adds an assertion that it is called on the main actor
        * If it's ever not called from the main actor, it will trap and the program will stop running
    * Can also use `@preconcurrency` on the extension as shorthand, no need to wrap the function content in `assumeIsolated`

```swift
extension Recaffeinater: @preconcurrency CaffeineThresholdDelegate {
    public func caffeineLevel(at level: Double) {
        if level < minimumCaffeine {
            // TODO: alert user to drink more coffee!
        }
    }
}
```

### Swift Language Mode

* Now that all the warnings are cleaned up, we can turn on the Swift 6 language mode
    * In Build Settings under `Swift Language Mode`
    * Set to Swift 6, and compile
    * From this point on, full data isolation checking is locked in, and any future changes will have full data isolation checking from the compiler
* At this point, move on to `CoffeeKit`
    * Start by adding `@MainActor` to `CaffeineThresholdDelegate`

```swift
@MainActor
public protocol CaffeineThresholdDelegate: AnyObject {
    func caffeineLevel(at level: Double)
}
```

* This puts a warning back in our extension, on the `Recaffeinater` extension

```swift
//warning: @preconcurrency attribute on conformance to 'CaffeineThresholdDelegate' has no effect
extension Recaffeinater: @preconcurrency CaffeineThresholdDelegate {
    public func caffeineLevel(at level: Double) {
        if level < minimumCaffeine {
            // TODO: alert user to drink more coffee!
        }
    }
}
```

* Now that the compiler can see that the protocol is guaranteed to be on the main actor, using `@preconcurrency` is no longer needed

```swift
extension Recaffeinater: CaffeineThresholdDelegate {
    public func caffeineLevel(at level: Double) {
        if level < minimumCaffeine {
            // TODO: alert user to drink more coffee!
        }
    }
}
```

* At this point, enable Complete Concurrency Checking for the `CoffeeKit` target
    * This generates 11 warnings
    * Most of them are resolved by changing `var` to `let`
* The next set of errors is due to data race issues

```swift
// warning: Sending 'self.currentDrinks' risks causing data races
// Sending main actor-isolated 'self.currentDrinks' to actor-isolated instance method 'save' risks causing data races between actor-isolated and main actor-isolated uses
await store.save(currentDrinks)
```

* We can look at the `Drink` model to see what it looks like:

```swift
// The record of a single drink.
public struct Drink: Hashable, Codable {

    // The amount of caffeine in the drink.
    public let mgCaffeine: Double

    // The date when the drink was consumed.
    public let date: Date

    // A globally unique identifier for the drink.
    public let uuid: UUID

    public let type: DrinkType?

    public var latitude, longitude: Double?

    // The drink initializer.
    public init(type: DrinkType, onDate date: Date, uuid: UUID = UUID()) {
        self.mgCaffeine = type.mgCaffeinePerServing
        self.date = date
        self.uuid = uuid
        self.type = type
    }

    internal init(from sample: HKQuantitySample) {
        self.mgCaffeine = sample.quantity.doubleValue(for: miligrams)
        self.date = sample.startDate
        self.uuid = sample.uuid
        self.type = nil
    }

    // Calculate the amount of caffeine remaining at the provided time,
    // based on a 5-hour half life.
    public func caffeineRemaining(at targetDate: Date) -> Double {
        // Calculate the number of half-life time periods (5-hour increments)
        let intervals = targetDate.timeIntervalSince(date) / (60.0 * 60.0 * 5.0)
        return mgCaffeine * pow(0.5, intervals)
    }
}
```

* `Drink` is a struct, with a few immutable properties
    * It can easily be made `Sendable`
    * If `Drink` had been internal, it would have automatically been considered `Sendable`
    * Because it's a public type, so that cannot be inferred, and much be added explicitly

```swift
// The record of a single drink.
public struct Drink: Hashable, Codable, Sendable {
  //...
}
```

* Upon recompiling, we see that the `DrinkType` property of `Drink` is not Sendable, giving us another warning there
    * If we can determine that this variable is safe, we could use the `nonisolated(unsafe)` keyword
    * Instead, though, we will mark `DrinkType` as `Sendable`

```swift
// Define the types of drinks supported by Coffee Tracker.
public enum DrinkType: Int, CaseIterable, Identifiable, Codable, Sendable {
  //...
}
```

* Now we can enable Swift 6 Language Mode on `CoffeeKit`
* After compiling, the app builds, and the entire app is protected by Swift concurrency
* Next we will try adding a new feature
    * Attempt to add location tracking, fetching location before adding a drink in CoffeeKit's `addDrink` method
    * Will use CoreLocation's async sequence that streams current location, and loop over results until we have the right level of accuracy
    * This raises an issue where we have to raise the minimum deployment target

```swift
//Create a new drink to add to the array.
var drink = Drink(type: type, onDate: date)

do {
  //error: 'CLLocationUpdate' is only available in watchOS 10.0 or newer
  for try await update in CLLocationUpdate.liveUpdates() {
    guard let coord = update.location else {
      logger.info( "Update received but no location, \(update.location)")
      break
    }
    drink.latitude = coord.coordinate.latitude
    drink.longitude = coord.coordinate.longitude
  } catch {
    
  }
```

* We don't want to raise our minimum deployment target, so we have to use the older CoreLocation APIs based on delegate callbacks
    * Predates Swift concurrency
    * We will remove the new code above using `CLLocationUpdate.liveUpdates`
    * Create a new `CLLocationManagerDelegate`

```swift
@MainActor
class CoffeeLocationDelegate: NSObject, CLLocationManagerDelegate {
  var location: CLLocation?
  var manager: CLLocationManager!
  
  var latitude: CLLocationDegrees? { location?.coordinate.latitude } 
  var longitude: CLLocationDegrees? { location?.coordinate.longitude }

  override init () {
    super.init()
    manager = CLLocationManager()
    manager.delegate = self
    manager.startUpdatingLocation()
  }
  
  func locationManager (
    _ manager: CLLocationManager, 
    didUpdateLocations locations: [CLLocation]
  ) {
      self.location = locations. last
  }
}
```

* We put this type entirely on the main actor
    * Means the location manager will be created on the main thread
    * This creates an error, though, as the `didUpdateLocations` function gives us an error
        * `Main actor-isolated instance method 'locationManager_:didUpdateLocations:)' cannot be used to satisfy nonisolated protocol requirement`
    * We resolve this by doing the following:
        * Mark the function as `nonisolated`
        * Wrap the code that must run on the main actor in `Mainactor.assumeIsolated { ... }`

```swift
nonisolated func locationManager (
  _ manager: CLLocationManager, 
  didUpdateLocations locations: [CLLocation]
) {
    MainActor.assumeIsolated { self.location = locations. last }
}
```

* [**Swift concurrency: Update a sample app**](https://developer.apple.com/videos/play/wwdc2021/10194/) session from WWDC 2021
* [**Swift 6 Migration Guide**](https://swift.org/migration)