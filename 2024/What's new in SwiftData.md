# [**What's new in SwiftData**](https://developer.apple.com/videos/play/wwdc2024-10137)

---

### **Adopt SwiftData**

![Features](images/new_swiftdata/features.png)

* To use SwiftData with the models in an app, just import the framework and decorate each model with the `@Model` macro

```swift
// Trip Models decorated with @Model
import Foundation
import SwiftData

@Model
class Trip {
    var name: String
    var destination: String
    var startDate: Date
    var endDate: Date
    
    var bucketList: [BucketListItem] = [BucketListItem]()
    var livingAccommodation: LivingAccommodation?
}

@Model
class BucketListItem {...}

@Model
class LivingAccommodation {...}
```

* In the app's definition, the `modelContainer` modifier on the WindowGroup tells the entire view hierarchy about the model `Trip`

```swift
// Trip App using modelContainer Scene modifier
import SwiftUI
import SwiftData

@main
struct TripsApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView
        }
        .modelContainer(for: Trip.self)
    }
}
```

* Views can then remove static data and populate the view using `@Query`
    * Fetches Trip models from the model container and return an array of Trips

```swift
// Trip App using @Query
import SwiftUI
import SwiftData

struct ContentView: View {
    @Query
    var trips: [Trip]
    var body: some View {
        NavigationSplitView {
            List(selection: $selection) {
                ForEach(trips) { trip in
                    TripListItem(trip: trip)
                }
            }
        }
    }
}
```

### **Customize the schema**

* Schema macros
    * Previously had:
        * `@Attribute`
        * `@Relationship`
        * `@Transient`
    * Now adding `#Unique`
        * Define key paths that represent the same data
        * Collisions become updates

```swift
// Add unique constraints to avoid duplication
import SwiftData

@Model 
class Trip {
    #Unique<Trip>([\.name, \.startDate, \.endDate])
    
    var name: String
    var destination: String
    var startDate: Date
    var endDate: Date
    
    var bucketList: [BucketListItem] = [BucketListItem]()
    var livingAccommodation: LivingAccommodation?
}
```

* `#Unique` properties help ensure models are not duplicated
    * Also represent the identity of the model
    * Can use the `@Attribute` macro to decorate these properties with `.preserveValueOnDeletion`
        * Ensure these identity values will be available when using the history API in SwiftData

```swift
// Add .preserveValueOnDeletion to capture unique columns
import SwiftData

@Model 
class Trip {
    #Unique<Trip>([\.name, \.startDate, \.endDate])
    
    @Attribute(.preserveValueOnDeletion)
    var name: String
    var destination: String

    @Attribute(.preserveValueOnDeletion)
    var startDate: Date

    @Attribute(.preserveValueOnDeletion)
    var endDate: Date
    
    var bucketList: [BucketListItem] = [BucketListItem]()
    var livingAccommodation: LivingAccommodation?
}
```

* Reveal history with SwiftData
    * Track inserted, updated, and deleted models
    * Opt in to preserve values on deletion
    * Works with custom data stores
    * [**Track model changes with SwiftData history**](./) session

### **Tailor a container**

* The `.modelContainer` modifier lets you customize some of the properties of the container
    * For example, customize to keep data in memory, enable/disable autosave, and enable/disable undo-redo support

```swift
// Customize a model container in the app
import SwiftUI
import SwiftData

@main
struct TripsApp: App {   
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: Trip.self,
                        inMemory: true,
                        isAutosaveEnabled: true,
                        isUndoEnabled: true)
   }
}
```

* A model container can be customized even more by building your own modelContainer instance
    * Create the container, specifying a configuration and location url
    * Pass in via the `.modelContainer` modifier

```swift
// Add a model container to the app
import SwiftUI
import SwiftData

@main
struct TripsApp: App {
    var container: ModelContainer = {
        do {
            let configuration = ModelConfiguration(schema: Schema([Trip.self]), url: fileURL)
            return try ModelContainer(for: Trip.self, configurations: configuration)
        }
        catch { ... }
    }()
    
   var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(container)
   }
}
```

* In iOS 18, you can customize a modelContainer even more with fully custom data stores
    * By default uses Core Data supporting all of SwiftData's features
    * Can create your own, which uses its own implementation

```swift
// Use your own custom data store
import SwiftUI
import SwiftData

@main
struct TripsApp: App {
    var container: ModelContainer = {
        do {
            let configuration = JSONStoreConfiguration(schema: Schema([Trip.self]), url: jsonFileURL)
            return try ModelContainer(for: Trip.self, configurations: configuration)
        }
        catch { ... }
    }()
    
   var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(container)
   }
}
```

#### Data Stores

* Use familiar SwiftData API like `@Model` and `@Query` macros, regardless of the format the data is persisted in
* Provides a way to adopt features incrementally
* [**Create a custom data store with SwiftData**](https://developer.apple.com/videos/play/wwdc2024-10138) session

#### Previews

* Can create custom containers for use with Xcode previews
    * Create a struct that conforms to `PreviewModifier`
    * Implement an extension on `PreviewTrait` to easily access sample data

```swift
// Make preview data using traits

struct SampleData: PreviewModifier {
    static func makeSharedContext() throws -> ModelContainer {
        let config = ModelConfiguration(isStoredInMemoryOnly: true)
        let container = try ModelContainer(for: Trip.self, configurations: config)
        Trip.makeSampleTrips(in: container)
        return container
    }
    
    func body(content: Content, context: ModelContainer) -> some View {
        content.modelContainer(context)
    }
}

extension PreviewTrait where T == Preview.ViewTraits {
    @MainActor static var sampleData: Self = .modifier(SampleData())
}
```

* To use the store in a Preview, pass your `sampleData` var created above as a trait to the `#Preview`

```swift
// Use sample data in a preview

import SwiftUI
import SwiftData

struct ContentView: View {
    @Query
    var trips: [Trip]

    var body: some View {
        ...
    }
}

#Preview(traits: .sampleData) {
    ContentView()
}
```

* For views that do not include queries, but instead rely on models being passed to them, use the `@Previewable` macro

```swift
// Create a preview query using @Previewable

import SwiftUI
import SwiftData

#Preview(traits: .sampleData) {
    @Previewable @Query var trips: [Trip]
    BucketListItemView(trip: trips.first)
}
```

### **Optimize queries**

* Predicates are used to sort and filter queries for a view
    * Automatically reacts to changes made to the ModelContainer
    * Can be evaluated during the data queries, rather than with large in-memory sets

```swift
// Create a Predicate to find a Trip based on Search Text
let predicate = #Predicate<Trip> {
    searchText.isEmpty ? true : $0.name.localizedStandardContains(searchText)
}

// Create a Compound Predicate to find a Trip based on Search Text
let predicate = #Predicate<Trip> {
    searchText.isEmpty ? true :
    $0.name.localizedStandardContains(searchText) ||
    $0.destination.localizedStandardContains(searchText)
}
```

* New in iOS 18 is the `#Expression` macro
    * Express complex queries that do not produce true or false, but instead allow for arbitrary types
    * Can be composed within predicates to tailor query results further

```swift
// Build a predicate to find Trips with BucketListItems that are not in the plan

let unplannedItemsExpression = #Expression<[BucketListItem], Int> { items in
    items.filter {
        !$0.isInPlan
    }.count
}

let today = Date.now
let tripsWithUnplannedItems = #Predicate<Trip>{ trip
    // The current date falls within the trip
    (trip.startDate ..< trip.endDate).contains(today) &&

    // The trip has at least one BucketListItem
    // where 'isInPlan' is false
    unplannedItemsExpression.evaluate(trip.bucketList) > 0
}
```

* `#Index` macro
    * Adds metadata to the model
    * Provides faster and more efficient queries
    * Specify properties used in queries

```swift
// Add Index for commonly used KeyPaths or combination of KeyPaths
import SwiftData

@Model 
class Trip {
    #Unique<Trip>([\.name, \.startDate, \.endDate])
    #Index<Trip>([\.name], [\.startDate], [\.endDate], [\.name, \.startDate, \.endDate])

    var name: String
    var destination: String
    var startDate: Date
    var endDate: Date
    
    var bucketList: [BucketListItem] = [BucketListItem]
    var livingAccommodation: LivingAccommodation
}
```
