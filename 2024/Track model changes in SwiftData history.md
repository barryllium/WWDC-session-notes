# [**Track model changes in SwiftData history**](https://developer.apple.com/videos/play/wwdc2024/10075/)

---

### **Fundamentals**

* As an app is used, the content stored by SwiftData changes over time
    * Models are created, fetched from a remote server, changed deleted, etc.
    * Queries represent what is currently in the data store
    * SwiftData History provides a way to track changes to the data store over time
* Reveal history
    * Sync offline changes
    * Discover out-of-process changes (like changes from a Widget extension)
    * Reload data efficiently
* SwiftData history
    * Composed of Transactions and Changes
    * Transactions group together all changes that occurred in the data store on a boundary, such as a `ModelContext` save
    * Transactions are ordered by when they occurred
        * Within a transaction, the set of changes it contains also preserve the order in which each change occurred
        * Each change represents a model that has been inserted, updated, or deleted
            * Parameterized by a `PersistentModel`, allowing references to properties of the `PersistentModel` to use KeyPaths
* SwiftData history uses the concept of a token, which acts as a bookmark for transactions in History
    * Helps the app keep track of the last transaction it processed in a stream of history
    * Tokens are only valid for the dat stores they are associated with, and history info can only be deleted through the model context
        * When deleted, tokens become expired and cannot be used for fetches
    * SwiftData History operations involving an expired token will throw a `SwiftDataError.historyTokenExpired` error
        * If this happens, discard the token and fetch a new one
* When models are deleted, the data in the model is discarded
    * Essential data like identifiers might be lost and don't provide enough information to the app when processing history information
    * SwiftData History lets you preserve specific attributes on a model to address this
        * When the model is deleted, these attributes are preserved as tombstone values, and let you process the history information for the deleted models
        * Use `@Attribute(.preserveValueOnDeletion)`
        * Tombstones are also parameterized by a PersistentModel, so that their KeyPaths can be used to retrieve the tombstone values or iterated as desired

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

### **Transactions and changes**

#### Fetch and process history

* Build `HistoryDescriptor` and fetch history
    * Create a predicate that constrains the transaction to have occurred after the provided token and by a specific author
    * If no token is provided, just fetch all the available history
    * Call `fetchHistory` on the `modelContext` using the `HistoryDescriptor` to get an array of transactions

```swift
private func findTransactions(after token: DefaultHistoryToken?, author: String) -> [DefaultHistoryTransaction] {
    var historyDescriptor = HistoryDescriptor<DefaultHistoryTransaction>() 
    if let token {
        historyDescriptor.predicate = #Predicate { transaction in
            (transaction.token > token) && (transaction.author == author)
        }
    }
    
    var transactions: [DefaultHistoryTransaction] = []
    let taskContext = ModelContext(modelContainer)
    do {
        transactions = try taskContext.fetchHistory(historyDescriptor)
    } catch let error {
        print(error)
    }

    return transactions
}
```

* Process the changes
    * Accepts an array of transactions and returns a set of Trips that need to be badged as having unread changes and a token
    * For each transaction and a change in a transaction, the History API provides the persistent model ID
        * To get the model instance of each `Trip`, build a fetch descriptor for `LivingAccommodation` using that persistent model ID.
        * Fetch the model from the model context and store the `Trip` associated with the `LivingAccommodation`
    * Use a switch statement to determine if it's an insert, update, or delete
    * Return the last token (along with the set of trips) so that future calls to the function return only the changes that occurred after that transaction

```swift
private func findTrips(in transactions: [DefaultHistoryTransaction]) -> (Set<Trip>, DefaultHistoryToken?) {
    let taskContext = ModelContext(modelContainer)
    var resultTrips: Set<Trip> = []
    for transaction in transactions {
        for change in transaction.changes {
            let modelID = change.changedPersistentIdentifier
            let fetchDescriptor = FetchDescriptor<Trip>(predicate: #Predicate { trip in
                trip.livingAccommodation?.persistentModelID == modelID
            })
            let fetchResults = try? taskContext.fetch(fetchDescriptor)
            guard let matchedTrip = fetchResults?.first else {
                continue
            }
            switch change {
            case .insert(_ as DefaultHistoryInsert<LivingAccommodation>):
                resultTrips.insert(matchedTrip)
            case .update(_ as DefaultHistoryUpdate<LivingAccommodation>):
                resultTrips.update(with: matchedTrip)
            case .delete(_ as DefaultHistoryDelete<LivingAccommodation>):
                resultTrips.remove(matchedTrip)
            default: break
            }
        }
    }
    return (resultTrips, transactions.last?.token)
}
```

* Save and use a history token
    * Use UserDefaults to store the most recent function
    * Fetch this token to call `findTransactions`, and store it after calling `findTrips`

```swift
private func findUnreadTrips() -> Set<Trip> {
    let tokenData = UserDefaults.standard.data(forKey: UserDefaultsKey.historyToken)
    
    var historyToken: DefaultHistoryToken? = nil
    if let tokenData {
        historyToken = try? JSONDecoder().decode(DefaultHistoryToken.self, from: tokenData)
    }
    let transactions = findTransactions(after: historyToken, author: TransactionAuthor.widget)
    let (unreadTrips, newToken) = findTrips(in: transactions)
    
    if let newToken {
        let newTokenData = try? JSONEncoder().encode(newToken)
        UserDefaults.standard.set(newTokenData, forKey: UserDefaultsKey.historyToken)
    }
    return unreadTrips
}
```

* Update user interface
    * When the app is opened, check for unread trips
    * When a trip is tapped on, remove it from the set of unread trips so the badge disappears
    * Add the badge to any Trip that is contained within the `unreadTripIdentifiers` set

```swift
struct ContentView: View {
    @Environment(\.scenePhase) private var scenePhase
    @State private var showAddTrip = false
    @State private var selection: Trip?
    @State private var searchText: String = ""
    @State private var tripCount = 0
    @State private var unreadTripIdentifiers: [PersistentIdentifier] = []

    var body: some View {
        NavigationSplitView {
            TripListView(selection: $selection, tripCount: $tripCount,
                         unreadTripIdentifiers: $unreadTripIdentifiers,
                         searchText: searchText)
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    EditButton()
                        .disabled(tripCount == 0)
                }
                ToolbarItemGroup(placement: .topBarTrailing) {
                    Spacer()
                    Button {
                        showAddTrip = true
                    } label: {
                        Label("Add trip", systemImage: "plus")
                    }
                }
            }
        } detail: {
            if let selection = selection {
                NavigationStack {
                    TripDetailView(trip: selection)
                }
            }
        }
        .task {
            unreadTripIdentifiers = await DataModel.shared.unreadTripIdentifiersInUserDefaults
        }
        .searchable(text: $searchText, placement: .sidebar)
        .sheet(isPresented: $showAddTrip) {
            NavigationStack {
                AddTripView()
            }
            .presentationDetents([.medium, .large])
        }
        .onChange(of: selection) { _, newValue in
            if let newSelection = newValue {
                if let index = unreadTripIdentifiers.firstIndex(where: {
                    $0 == newSelection.persistentModelID
                }) {
                    unreadTripIdentifiers.remove(at: index)
                }
            }
        }
        .onChange(of: scenePhase) { _, newValue in
            Task {
                if newValue == .active {
                    unreadTripIdentifiers += await DataModel.shared.findUnreadTripIdentifiers()
                } else {
                    // Persist the unread trip names for the next launch session.
                    await DataModel.shared.setUnreadTripIdentifiersInUserDefaults(unreadTripIdentifiers)
                }
            }
        }
        #if os(macOS)
        .onReceive(NotificationCenter.default.publisher(for: NSApplication.didBecomeActiveNotification)) { _ in
            Task {
                unreadTripIdentifiers += await DataModel.shared.findUnreadTripIdentifiers()
            }
        }
        .onReceive(NotificationCenter.default.publisher(for: NSApplication.willTerminateNotification)) { _ in
            Task {
                await DataModel.shared.setUnreadTripIdentifiersInUserDefaults(unreadTripIdentifiers)
            }
        }
        #endif
    }
}
```

### **Custom stores**

* Add history to custom data stores
    * You need to implement your own types to represent the fundamental elements of the SwiftData History API for your data store
        * `HistoryTransaction`
        * `HistoryChange`
        * `HistoryToken`
    * The custom data store also needs to conform to `HistoryProviding`
* Custom transaction
    * The boundaries of a transaction need to to be well-defined because write operations in the data store need to be coalesced and ordered
        * In the default store all of the changes to model instances on the `ModelContext` at save time are grouped as a single transaction
    * Need to define a way to uniquely identify a transaction within your persistence backend
* Custom change
    * What defines the boundaries of a change must be well-defined
        * In the default store, the boundaries of a change are scoped to an individual model instance
    * Pick a unique identifier to track the granular nature of these changes
    * Consider change types - do you need update and delete, or just insert change types? s
    * Consider if your app will need to support preserving values on deletion and how the deleted values will be stored
* Custom history providing
    * The custom store will need to implement the `HistoryProviding` protocol to vend history
    * After identifying which rows are part of a transaction, you need to build the specific sets of models
    * The default data store manages the TTL of history records
        * Custom providers need to decide when to delete history
* Custom token
    * Need to create a custom type of token
        * `HistoryToken` is the base protocol for a token
        * Needed to uniquely identify your position in the stream of transactions
    * Consider if your app uses multiple, related stores
        * Your custom token should include the state of all stores used in the transaction
* If you were using co-existence with Core Data to benefit from persistent history, you can now migrate to SwiftData History instead
