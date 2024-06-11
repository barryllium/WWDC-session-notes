# [**Meet SwiftData**](https://developer.apple.com/videos/play/wwdc2023/10187/)

---

SwiftData is a framework for data modeling and management and enhances your modern Swift app. Like SwiftUI, it focuses entirely on code with no external file formats and uses Swift's new macro system to create a seamless API experience.

* Relies on Swift macros
* Integrated with SwiftUI
* Works with other platform features, like CloudKit and Widgets

### **Using the model macro**

#### `@Model`

* New Swift macro
* Define your schema with code
    * when needed, you can annotate your properties with additional metadata
* Add SwiftData functionality to model types

```swift
import SwiftData

@Model
class Trip {
    var name: String
    var destination: String
    var endDate: Date
    var startDate: Date
 
    var bucketList: [BucketListItem]? = []
    var livingAccommodation: LivingAccommodation?
}
```

#### Attributes

* Attributes inferred from properties
* Support basic value types (string, int, float)
* Complex value types
    * Struct
    * Enum
    * Codable
    * Collections of value types

#### Relationships

* Relationships are inferred from reference types
    * Other model types
    * Collections of model types

#### Additional metadata

* `@Model` modifies all stored properties
* Control how properties are inferred
    * `@Attribute` - can add unique constraints
        * In below code, the `.unique` attribute is used to identify `name` as unique
    * `@Relationship` - control the choice of inverses and specify delete propagation rules
        * Below, `.cascade` instructs SwiftData to delete all the related bucket list items whenever this trip is deleted
* Exclude properties with `@Transient`
* To learn more, check out the [**Model your schema with SwiftData**](./Model%20your%20schema%20with%20SwiftData.md) session

```swift
@Model
class Trip {
    @Attribute(.unique) var name: String
    var destination: String
    var endDate: Date
    var startDate: Date
 
    @Relationship(.cascade) var bucketList: [BucketListItem]? = []
    var livingAccommodation: LivingAccommodation?
}
```

### **Working with your data**

#### Model container

* Persistence backend
* Customized with configurations
* Provides schema migration options
* You can create a model container just by specifying the list of model types you want stored
    * If you want to customize your container further, you can use a configuration to change your URL, CloudKit and group container identifiers, and migration options

```swift
// Initialize with only a schema
let container = try ModelContainer([Trip.self, LivingAccommodation.self])

// Initialize with configurations
let container = try ModelContainer(
    for: [Trip.self, LivingAccommodation.self],
    configurations: ModelConfiguration(url: URL("path"))
)
```

* You can also use SwiftUI's view and scene modifiers to set up container and have it automatically established in the view's environment

```swift
import SwiftUI

@main
struct TripsApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(
            for: [Trip.self, LivingAccommodation.self]
        )
    }
}
```

#### ModelContext

* Observe all the changes to your models and provide many of the actions ot operate on them
    * Tracking updates
    * Fetching models
    * Saving changes
    * Undoing changes
* Generally get the modelContext from your view's environment after you create your model container
* Outside the view hierarchy, you can ask the model container to give you a shared main actor bound context, or you can simply instantiate new contexts for a given model container.

```swift
import SwiftUI

struct ContextView : View {
    @Environment(\.modelContext) private var context
}


import SwiftData

// let context = container.mainContext
let context = ModelContext(container)
```

#### Fetching your data

* New Swift native types
    * `Predicate`
    * `FetchDescriptor`
* Improvements to `SortDescriptor`

#### Predicate

* Works with native Swift types and uses Swift macros for strongly typed construction
    * Fully type checked modern replacement for NSPredicate
* `#Predicate` construction instead of text parsing
* Autocompleted keypaths
* Example to fetch all trips to New York that are about birthdays, that are in the future

```swift
let today = Date()
let tripPredicate = #Predicate<Trip> { 
    $0.destination == "New York" &&
    $0.name.contains("birthday") &&
    $0.startDate > today
}

let descriptor = FetchDescriptor<Trip>(predicate: tripPredicate)

let trips = try context.fetch(descriptor)
```

#### SortDescriptor

* Updated to support all `Comparable` types
* Supports native Swift types and keypaths
* Can use it to specify the order our results should be organized

```swift
let descriptor = FetchDescriptor<Trip>(
    sortBy: SortDescriptor(\Trip.name),
    predicate: tripPredicate
)

let trips = try context.fetch(descriptor)
```

#### More FetchDescriptor options

* Relationships to prefetch
* Result limits
* Exclude unsaved changes

#### Modifying your data

* Basic operations
    * Inserting
    * Deleting
    * Saving
    * Changing

```swift
var myTrip = Trip(name: "Birthday Trip", destination: "New York")

// Insert a new trip
context.insert(myTrip)

// Delete an existing trip
context.delete(myTrip)

// Manually save changes to the context
try context.save()
```

* `@Model` macro modifiers setters for change tracking and observation
* Updates automatically by the `Model Context`
* [**Dive deeper into SwiftData**](./Dive%20deeper%20into%20SwiftData.md) session

### **Use SwiftData with SwiftUI**

* Seamless integration with SwiftUI
* Easy configuration
* Automatically fetch data and update views

#### View modifiers

* Leverage scene and view modifiers
* Configure data store with `.modelContainer`
    * enable undo, toggle autosaving, etc.
* Propagated throughout SwiftUI environment
* `@Query` property wrapper
    * Load, filter anything in your database with a single line of code

```swift
import SwiftUI

struct ContentView: View  {
    @Query(sort: \.startDate, order: .reverse) var trips: [Trip]
    @Environment(\.modelContext) var modelContext
    
    var body: some View {
       NavigationStack() {
          List {
             ForEach(trips) { trip in 
                 // ...
             }
          }
       }
    }
}
```

#### Observing changes

* No need for `@Published`
* SwiftUI automatically refreshes

Learn more in the [**Build an app with SwiftData**](./Build%20an%20app%20with%20SwiftData.md) session
