# [**What's New in SwiftData**](https://developer.apple.com/videos/play/wwdc2026/274/)

---

### **Sectioning your fetches**

* When using `@Query`, can now add `sectionBy` parameter to section the data
    * The var in the code below stays as `[Trip]`
    * Wrap your `ForEach` in a section, and add a second `ForEach` for the items
    * To get the sections, access the query from the property wrapper, using the underscore-prefixed name `_trips`
    * Each section has an ID property: the ID is the value of my model from the KeyPath I passed to the SectionBy parameter of Query

```swift
// Sectioned fetching

struct TripListView: View {   
    @Query(sort: \Trip.startDate,
           sectionBy: \.destination)
    var trips: [Trip]

    var body: some View: {
        List(selection: $selection) {
            ForEach(_trips.sections) {section in
                Section(section.id) {
                    ForEach(trips) {trip in
                        TripListItem(trip: trip)
                    }
               }
            }
        }
    }
}
```

### **Using custom types**

* `@Attribute(.codable)` allows adding custom types to an `@Model` class if it's `Codable`
    * No support for filtering or sorting
    * Migration is defined by the type
        * If the shape of the codable type changes, like adding or removing properties, they will not trigger a migration
        * The Codable implementation of the type must be able to encode and decode in a forward- and backward-compatible way
    * Avoid using codable for Types that you define
        * Modeling them as SwiftData models or supported value types, allows you to sort, filter, and index
        * For types that you don't own, codable lets you store them alongside your other attributes

```swift
// Using Codable

import SwiftData

@Model class Trip {

    struct Location: Codable {
        var latitude: Double
        var longitude: Double
    }

    var name: String
    var destination: String

    var startDate: Date
    var endDate: Date

    var location: Location?
    @Attribute(.codable) var mapItemIdentifier: MKMapItem.Identifier?
}
```

### **Observing data stores**

* ResultsObserver
    * Observes a data store outside of a SwiftUI view
    * Uses Swift Observation
    * Allows sorting, filtering, and sectioning
    * In the code below, the token defines the lifetime of the observation - it's stored on the class to receive updates for the entire lifetime of the MapCameraController

```swift
// Use observation to update map bounds

@Observable @MainActor final class MapCameraController {
    private let resultsObserver: ResultsObserver<Trip, Never>
    var bounds: MapCameraBounds?
    private var token: ObservationTracking.Token?

    init(modelContext: ModelContext) throws {
        resultsObserver = try ResultsObserver<Trip, Never>(modelContext: modelContext)

        token = withContinuousObservation(options: [.didSet]) {[weak self], event in
            self?.bounds = self?.calculateBounds(trips: resultsObserver.results)
       }
    }

    private func calculateBounds(trips: [Trip]) -> MapCameraBounds? { ... }
}
```

* Persistent history
    * When data in a store changes, SwiftData keeps a record of everything that changes
    * You can use this history to build features like syncing with external servers, or reacting to changes made outside of your app, like in an app extension
    * Every time your data store is saved, SwiftData records a `HistoryTransaction`
        * Contains information about what changed, and where the change was coming from - and a token that uniquely identifies the transaction in the history
        * Can be used with the `ModelContext.fetchHistory` API to fetch newer transactions
        * [Track model changes with SwiftData history](../2024/Track%20model%20changes%20in%20SwiftData%20history.md) Session from WWDC 2024
* `HistoryObserver`
    * Observes the persistent history and lets your code react when new transactions are added
    * Filter by model type or transaction author
    * Uses Swift Observation
    * `HistoryObserver` has a single observable property - eventCounter: when new transactions are available in the persistent history, the eventCounter increments
        * Your code can observe the eventCounter and when it increments, use `ModelContext.fetchHistory` API to fetch the latest changes

```swift
// Using HistoryObserver to sync with a server

@SyncActor final class ServerSync {
    private let observer: HistoryObserver
    private var token: ObservationTracking.Token?
    
    func start() throws {
        self.observer = try HistoryObserver(authors: ["App"], modelContainer: modelContainer)
        token = withContinuousObservation(options: .didSet) {[weak self] _ in
            _ = self?.observer.eventCounter
            self?.processChanges()
        }
    }

    private func processChanges() {
        // Fetch and process history transactions
    }
}
```
