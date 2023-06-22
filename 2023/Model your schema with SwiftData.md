# [**Model your schema with SwiftData**](https://developer.apple.com/videos/play/wwdc2023/10195/)

---

Watch [**Meet SwiftData**](./Meet%20SwiftData.md) session first

* Original Trip model using `@Model` to utilize SwiftData

```swift
import SwiftUI
import SwiftData

@Model
final class Trip {
    var name: String
    var destination: String
    var start_date: Date
    var end_date: Date
    
    var bucketList: [BucketListItem]? = []
    var livingAccommodation: LivingAccommodation?
}
```

### **Utilizing schema macros**

* With the original model, trip names were not assured to be unique, causing conflicts between trips with the same name
    * Resolved by adding the `@Attribute` schema macro with the `.unique` option to the `name` var
    * If a trip already exists with that name, then the persistent back end will update to the latest values (called an `upsert`)
        * Starts as an insert, but becomes an update with a data collision
    * Can apply unique constraints on other properties as long as they are primitive value types such as numerics, string, or UUID
    * Can also decorate a `to-one` relationship

```swift
@Model 
final class Trip {
    @Attribute(.unique) var name: String
    var destination: String
    var start_date: Date
    var end_date: Date
    
    var bucketList: [BucketListItem]? = []
    var livingAccommodation: LivingAccommodation?
}
```

* We also want to change the parameter names of the `start_date` and `end_date` params to remove the underscores
    * Simply changing the names would generate a new property in the schema, and we would lose existing data
    * Instead, we preserve the existing data using the `originalName` parameter on `@Attribute`
    * Ensures the schema update will be a simple migration for the next release of the app.

```swift
@Model 
final class Trip {
    @Attribute(.unique) var name: String
    var destination: String
    @Attribute(originalName: "start_date") var startDate: Date
    @Attribute(originalName: "end_date") var endDate: Date
    
    var bucketList: [BucketListItem]? = []
    var livingAccommodation: LivingAccommodation?
}
```

* `@Attribute` can also:
    * Store large data externally
    * Provide support for transformables
    * Spotlight integration
    * Hash modifier

#### Relationships

* SwiftData will implicitly discover the inverses between my models and set them
* Implicit inverses do not require any annotations
    * Implicit inverses use a default delete rule that will nullify the child items when a parent is deleted (e.g. `bucketList` items and `livingAccomodation` will nullify when a `Trip` is deleted)
* We can change the behavior to a delete (instead of nullify) using the `.cascade` delete rule on the `@Relationship` macro

```swift
@Model 
final class Trip {
    @Attribute(.unique) var name: String
    var destination: String
    @Attribute(originalName: "start_date") var startDate: Date
    @Attribute(originalName: "end_date") var endDate: Date
    
    @Relationship(.cascade)
    var bucketList: [BucketListItem]? = []
  
    @Relationship(.cascade)
    var livingAccommodation: LivingAccommodation?
}
```

* `@Relationship` macro also:
    * `originalName` modifier
    * Ability to specify the minimum and maximum count on a to-many relationship
    * Hash modifier

#### Transient data

* For data we do not want to be persisted by SwiftData, we use the `@Transient` macro
* This hides properties from SwiftData
* We can specify which properties do not persist
* Be sure to provide a default value so we have logical values when fetched from SwiftData

```swift
@Model 
final class Trip {
    @Attribute(.unique) var name: String
    var destination: String
    @Attribute(originalName: "start_date") var startDate: Date
    @Attribute(originalName: "end_date") var endDate: Date
    
    @Relationship(.cascade)
    var bucketList: [BucketListItem]? = []
  
    @Relationship(.cascade)
    var livingAccommodation: LivingAccommodation?

    @Transient
    var tripViews: Int = 0
}
```

### **Evolving schemas**

* Whenever you prepare to release a new version of your app with changes to your SwiftData models, define a `VersionedSchema` that encapsulates your previously released schema
    * Each distinct version of your schema should be defined as a VersionedSchema so that SwiftData knows what changes occurred between them
* Use your total ordering of `VersionedSchemas` to create a `SchemaMigrationPlan`
    * Allows SwiftData to perform the needed migrations in order
* Define each migration stage
    * Lightweight migration stages do not require any additional code to migrate existing data
        * e.g. `originalName` or specifying delete rules
        * Making a parameter unique is not eligible for a lightweight migration
    * Custom migrations do require additional code - making the name parameter unique requires that we de-duplicate trips before making their names unique
* In the code below version schemas
    * The original schema is encapsulated as `SampleTripsSchemaV1`
    * `SampleTripsSchemaV2` adds the uniqueness constraint
    * `SampleTripsSchemaV3` adds the name changes made to the start and end date

```swift
enum SampleTripsSchemaV1: VersionedSchema {
    static var models: [any PersistentModel.Type] {
        [Trip.self, BucketListItem.self, LivingAccommodation.self]
    }

    @Model
    final class Trip {
        var name: String
        var destination: String
        var start_date: Date
        var end_date: Date
    
        var bucketList: [BucketListItem]? = []
        var livingAccommodation: LivingAccommodation?
    }

    // Define the other models in this version...
}

enum SampleTripsSchemaV2: VersionedSchema {
    static var models: [any PersistentModel.Type] {
        [Trip.self, BucketListItem.self, LivingAccommodation.self]
    }

    @Model
    final class Trip {
        @Attribute(.unique) var name: String
        var destination: String
        var start_date: Date
        var end_date: Date
    
        var bucketList: [BucketListItem]? = []
        var livingAccommodation: LivingAccommodation?
    }

    // Define the other models in this version...
}

enum SampleTripsSchemaV3: VersionedSchema {
    static var models: [any PersistentModel.Type] {
        [Trip.self, BucketListItem.self, LivingAccommodation.self]
    }

    @Model
    final class Trip {
        @Attribute(.unique) var name: String
        var destination: String
        @Attribute(originalName: "start_date") var startDate: Date
        @Attribute(originalName: "end_date") var endDate: Date
    
        var bucketList: [BucketListItem]? = []
        var livingAccommodation: LivingAccommodation?
    }

    // Define the other models in this version...
}
```

* We then construct a schema migration plan, to describe how to handle the migrations from release to release
    * Provide the total ordering of my application's schemas
    * Annotate which migrations are lightweight or custom

```swift
enum SampleTripsMigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] {
        [SampleTripsSchemaV1.self, SampleTripsSchemaV2.self, SampleTripsSchemaV3.self]
    }
    
    static var stages: [MigrationStage] {
        [migrateV1toV2, migrateV2toV3]
    }

    static let migrateV1toV2 = MigrationStage.custom(
        fromVersion: SampleTripsSchemaV1.self,
        toVersion: SampleTripsSchemaV2.self,
        willMigrate: { context in
            let trips = try? context.fetch(FetchDescriptor<SampleTripsSchemaV1.Trip>())
                      
            // De-duplicate Trip instances here...
                      
            try? context.save() 
        }, didMigrate: nil
    )
  
    static let migrateV2toV3 = MigrationStage.lightweight(
        fromVersion: SampleTripsSchemaV2.self,
        toVersion: SampleTripsSchemaV3.self
    )
}
```

* Finally, perform the migration by setting up the `ModelContainer` with the current schema and the migration plan

```swift
struct TripsApp: App {
    let container = ModelContainer(
        for: Trip.self, 
        migrationPlan: SampleTripsMigrationPlan.self
    )
    
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(container)
    }
}
```
