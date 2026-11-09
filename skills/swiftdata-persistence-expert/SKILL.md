---
name: swiftdata-persistence-expert
description: SwiftData and Core Data persistence architecture, schema design, model relationships, CloudKit sync, and safe multi-version database migrations (VersionedSchema & SchemaMigrationPlan). Use when designing data persistence, fixing store corruption, or upgrading app schema between releases.
---

# SwiftData & Core Data Persistence Specialist

This skill provides authoritative guidance on implementing, optimizing, and migrating data persistence using **SwiftData** and **Core Data**, based on Apple's official documentation and WWDC architectural standards.

---

## 1. Schema Design Standards (`@Model`)

### A. Model Definition Rules
1. **Always provide default values:** If CloudKit sync is enabled, every property must either be optional or have an explicit default value.
2. **Define explicit delete rules:** Never leave `@Relationship` delete rules implicit. Always choose `.cascade`, `.nullify`, or `.deny`.
3. **Use `@Transient` for computed or runtime-only state:** Properties marked `@Transient` are never written to SQLite disk.

```swift
import SwiftData
import Foundation

@Model
public final class Project {
    public var id: UUID = UUID()
    public var title: String = ""
    public var createdAt: Date = Date.now
    public var isArchived: Bool = false

    // Explicit relationship with delete rule
    @Relationship(deleteRule: .cascade, inverse: \TaskItem.project)
    public var tasks: [TaskItem] = []

    // Transient property: Not saved to SQLite
    @Transient
    public var isSelectedForBatchAction: Bool = false

    public init(title: String) {
        self.title = title
    }
}
```

---

## 2. Safe Schema Migrations (`VersionedSchema` & `SchemaMigrationPlan`)

When updating an application from Version 1 to Version 2, unversioned schema modifications can cause **fatal startup crashes** (`Store could not be loaded`). Always define explicit `VersionedSchema` namespaces:

### Step 1: Define Schemas per Version
```swift
// Version 1 Schema
public enum AppSchemaV1: VersionedSchema {
    public static var versionIdentifier: Schema.Version = Schema.Version(1, 0, 0)
    public static var models: [any PersistentModel.Type] { [Project.self, TaskItem.self] }

    @Model
    public final class Project {
        public var title: String = ""
        public init(title: String) { self.title = title }
    }
}

// Version 2 Schema (Added 'priority' field, renamed 'title' to 'name')
public enum AppSchemaV2: VersionedSchema {
    public static var versionIdentifier: Schema.Version = Schema.Version(2, 0, 0)
    public static var models: [any PersistentModel.Type] { [Project.self, TaskItem.self] }

    @Model
    public final class Project {
        public var name: String = ""
        public var priority: Int = 1
        public init(name: String, priority: Int = 1) {
            self.name = name
            self.priority = priority
        }
    }
}
```

### Step 2: Define Migration Plan
```swift
public enum AppMigrationPlan: SchemaMigrationPlan {
    public static var schemas: [any VersionedSchema.Type] {
        [AppSchemaV1.self, AppSchemaV2.self]
    }

    public static var stages: [MigrationStage] {
        [migrateV1toV2]
    }

    public static let migrateV1toV2 = MigrationStage.custom(
        fromVersion: AppSchemaV1.self,
        toVersion: AppSchemaV2.self,
        willMigrate: { context in
            // Optional pre-migration validation
        },
        didMigrate: { context in
            // Data transformation: populate new 'name' from old 'title'
            let projects = try context.fetch(FetchDescriptor<AppSchemaV2.Project>())
            for project in projects {
                if project.priority == 0 { project.priority = 1 }
            }
            try context.save()
        }
    )
}
```

### Step 3: Initialize ModelContainer with Migration Plan
```swift
let container = try ModelContainer(
    for: AppSchemaV2.Project.self,
    migrationPlan: AppMigrationPlan.self,
    configurations: ModelConfiguration(isStoredInMemoryOnly: false)
)
```

---

## 3. Concurrency with SwiftData (`@ModelActor`)

**CRITICAL RULE:** `ModelContext` is **NOT thread-safe**. Never access or pass a `ModelContext` across different threads or actors.

To perform background imports, bulk syncs, or intensive calculations without freezing the UI, use the **`@ModelActor`** macro:

```swift
import SwiftData

@ModelActor
public actor BackgroundDataImporter {
    public func importRemoteRecords(_ records: [RemoteRecordDTO]) throws {
        // modelContext and modelExecutor are synthesized by @ModelActor
        for record in records {
            let entity = Project(title: record.name)
            modelContext.insert(entity)
        }
        try modelContext.save()
    }
}
```

---

## 4. SwiftData Verification Checklist

- [ ] **CloudKit Ready:** If using iCloud sync, verify no model uses `@Attribute(.unique)` and all fields have default values or are optional.
- [ ] **No Cross-Thread Context Access:** Ensure `@Query` and `modelContext` in views stay on the `@MainActor`. All background mutations must use `@ModelActor`.
- [ ] **Delete Rules Tested:** Verify what happens when a parent entity is deleted (do children cascade delete or nullify?).
- [ ] **Migration Plan Active:** Any change to model fields in an existing app must increment `VersionedSchema` and be tested with an old store database file.
