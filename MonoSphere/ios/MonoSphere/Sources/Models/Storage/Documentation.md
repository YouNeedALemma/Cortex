# MonoSphere Storage Implementation

This document provides detailed information about the storage implementations in MonoSphere, focusing on how events are persisted and retrieved in the event-sourced architecture.

## Overview

The MonoSphere storage system is built around the `EventStore` protocol, which defines a common interface for storing and retrieving events. The system provides multiple implementations to support different use cases:

1. **InMemoryEventStore**: A volatile, in-memory implementation for development and testing
2. **CoreDataEventStore**: A persistent implementation using Core Data for iOS applications

Each implementation adheres to the same interface, allowing the application to switch between storage mechanisms without changing the application logic.

## EventStore Protocol

The `EventStore` protocol defines the fundamental operations for event storage:

```swift
public protocol EventStore {
    /// Add an event to the store
    func append(_ event: Event) async throws
    
    /// Get events with optional pagination
    func getEvents(startId: String?, limit: Int?) async throws -> [Event]
    
    /// Get events of a specific type with optional pagination
    func getEventsByType(type: String, startId: String?, limit: Int?) async throws -> [Event]
    
    /// Get events with a specific correlation ID
    func getEventsByCorrelationId(correlationId: String) async throws -> [Event]
    
    /// Migration registry for schema evolution
    var migrationRegistry: EventMigrationRegistry { get }
}
```

This protocol establishes a clear contract for how events are stored and retrieved:

- **append**: Add a new event to the store
- **getEvents**: Retrieve events with optional pagination
- **getEventsByType**: Retrieve events of a specific type
- **getEventsByCorrelationId**: Retrieve events with a specific correlation ID
- **migrationRegistry**: Access to the event migration system

## In-Memory Event Store

The `InMemoryEventStore` provides a simple implementation that keeps events in memory:

```swift
public class InMemoryEventStore: EventStore {
    private var events: [Event] = []
    public let migrationRegistry: EventMigrationRegistry
    
    public init(migrationRegistry: EventMigrationRegistry = .shared) {
        self.migrationRegistry = migrationRegistry
    }
    
    public func append(_ event: Event) async throws {
        events.append(event)
    }
    
    public func getEvents(startId: String? = nil, limit: Int? = nil) async throws -> [Event] {
        var result = events
        
        // Apply pagination
        if let startId = startId, let startIndex = events.firstIndex(where: { $0.id == startId }) {
            result = Array(events[startIndex...])
        }
        
        if let limit = limit {
            result = Array(result.prefix(limit))
        }
        
        // Apply migrations
        return result.map { migrationRegistry.migrateEvent($0) }
    }
    
    // Other methods...
}
```

**Key Characteristics**:

- Events are stored in an in-memory array
- No persistence between application launches
- Fast access and simple implementation
- Automatic migration of events via the migration registry
- Suitable for development, testing, and prototyping

## Core Data Event Store

The `CoreDataEventStore` provides a persistent implementation using Core Data:

```swift
public class CoreDataEventStore: EventStore {
    private let dataModel: EventDataModel
    public let migrationRegistry: EventMigrationRegistry
    
    public init(
        dataModel: EventDataModel = .shared,
        migrationRegistry: EventMigrationRegistry = .shared
    ) {
        self.dataModel = dataModel
        self.migrationRegistry = migrationRegistry
    }
    
    public func append(_ event: Event) async throws {
        let context = dataModel.newBackgroundContext()
        
        try await context.perform {
            // Create event entity
            let _ = try CoreDataSerialization.eventToEntity(event, context: context)
            
            // Save context
            try self.dataModel.saveContext(context)
        }
    }
    
    public func getEvents(startId: String? = nil, limit: Int? = nil) async throws -> [Event] {
        let context = dataModel.newBackgroundContext()
        
        return try await context.perform {
            // Create fetch request
            let fetchRequest = NSFetchRequest<EventEntity>(entityName: "EventEntity")
            
            // Apply sorting and filtering
            fetchRequest.sortDescriptors = [NSSortDescriptor(key: "timestamp", ascending: true)]
            
            if let startId = startId {
                let startEvent = try self.fetchEventById(startId, context: context)
                if let startEvent = startEvent {
                    fetchRequest.predicate = NSPredicate(format: "timestamp > %@", startEvent.timestamp as NSDate)
                }
            }
            
            if let limit = limit {
                fetchRequest.fetchLimit = limit
            }
            
            // Execute fetch
            let eventEntities = try context.fetch(fetchRequest)
            
            // Convert entities to events
            let events = try eventEntities.map { try self.convertToEvent($0) }
            
            // Apply migrations
            return events.map { self.migrationRegistry.migrateEvent($0) }
        }
    }
    
    // Other methods...
}
```

**Key Characteristics**:

- Events are stored in a persistent Core Data database
- Data survives application restarts
- Supports complex queries through NSPredicate
- Uses NSManagedObjectContext for thread safety
- Automatic migration of events via the migration registry
- Suitable for production use in iOS applications

## Core Data Model

The Core Data model for events is defined programmatically:

```swift
public class EventDataModel {
    private lazy var persistentContainer: NSPersistentContainer = {
        // Create Core Data model programmatically
        let model = createEventModel()
        let container = NSPersistentContainer(name: modelName, managedObjectModel: model)
        
        // Load persistent stores
        container.loadPersistentStores { (storeDescription, error) in
            if let error = error as NSError? {
                fatalError("Failed to load Core Data stack: \(error)")
            }
        }
        
        return container
    }()
    
    private func createEventModel() -> NSManagedObjectModel {
        let model = NSManagedObjectModel()
        
        // Define Event entity
        let eventEntity = NSEntityDescription()
        eventEntity.name = "EventEntity"
        eventEntity.managedObjectClassName = NSStringFromClass(EventEntity.self)
        
        // Define EventMetadata entity
        let metadataEntity = NSEntityDescription()
        metadataEntity.name = "EventMetadataEntity"
        metadataEntity.managedObjectClassName = NSStringFromClass(EventMetadataEntity.self)
        
        // Define attributes and relationships
        // ...
        
        return model
    }
}
```

**Entity Structure**:

1. **EventEntity**:
   - id (String): Unique identifier
   - type (String): Event type
   - timestamp (Date): When the event occurred
   - payload (Binary Data): Serialized event payload
   - relationship to EventMetadataEntity

2. **EventMetadataEntity**:
   - userId (String): User who initiated the event
   - deviceId (String): Device from which the event originated
   - version (String): Schema version of the event
   - correlationId (String): Correlation ID for grouping related events
   - causationId (String): Causation ID for tracking event chains
   - relationship to EventEntity

## Serialization

Events are serialized to and from Core Data using the `CoreDataSerialization` utility:

```swift
public enum CoreDataSerialization {
    /// Serialize AnyCodable dictionary to Data
    public static func serializePayload(_ payload: [String: AnyCodable]) throws -> Data
    
    /// Deserialize Data to AnyCodable dictionary
    public static func deserializePayload(_ data: Data) throws -> [String: AnyCodable]
    
    /// Convert Event to CoreData entity
    public static func eventToEntity(_ event: Event, context: NSManagedObjectContext) throws -> EventEntity
    
    /// Convert CoreData entity to Event
    public static func entityToEvent(_ entity: EventEntity) throws -> Event
}
```

This utility handles the conversion between Swift domain models and Core Data entities, including serialization of the `AnyCodable` payload to binary data.

## Factory Function

The storage system provides a factory function to create event stores:

```swift
public enum EventStoreType {
    case inMemory
    case coreData
    case custom(EventStore)
}

public func createEventStore(
    type: EventStoreType,
    migrationRegistry: EventMigrationRegistry = .shared
) -> EventStore {
    switch type {
    case .inMemory:
        return InMemoryEventStore(migrationRegistry: migrationRegistry)
    case .coreData:
        return CoreDataEventStore(migrationRegistry: migrationRegistry)
    case .custom(let store):
        return store
    }
}
```

This factory function allows the application to easily switch between storage implementations or provide custom implementations.

## Migration Support

Both storage implementations include support for event schema migration:

```swift
// When retrieving events
let events = try await eventStore.getEvents()

// Events are automatically migrated
return events.map { migrationRegistry.migrateEvent($0) }
```

This ensures that events are migrated to the latest schema version when they are retrieved, regardless of the storage mechanism used.

## Thread Safety

The storage implementations address thread safety in different ways:

1. **InMemoryEventStore**:
   - Uses async/await to handle concurrent access
   - Operations are serialized through the Swift concurrency system

2. **CoreDataEventStore**:
   - Uses Core Data's concurrency model with NSManagedObjectContext
   - Creates a new background context for each operation
   - Uses performAndWait for transactional safety

```swift
public func append(_ event: Event) async throws {
    let context = dataModel.newBackgroundContext()
    
    try await context.perform {
        // Create and save entity...
        try self.dataModel.saveContext(context)
    }
}
```

## Error Handling

Storage operations may fail for various reasons, and errors are propagated through the Swift error handling system:

```swift
do {
    try await eventStore.append(event)
} catch let error as DomainError {
    // Handle domain-specific error
    switch error {
    case .eventStoreError(let reason):
        print("Event store error: \(reason)")
    default:
        print("Other domain error: \(error.localizedDescription)")
    }
} catch {
    // Handle other errors
    print("Unexpected error: \(error.localizedDescription)")
}
```

Common error types include:

- **eventStoreError**: General event store operation failure
- **serializationError**: Failure to serialize or deserialize data
- **migrationError**: Failure to migrate an event

## Performance Considerations

### InMemoryEventStore

The in-memory implementation provides optimal performance for small to medium event streams but has limitations:

- **Memory Usage**: All events are stored in memory, which can be problematic for large event streams
- **Query Performance**: Linear search time for finding events by ID
- **Scalability**: Not suitable for applications with very large numbers of events

### CoreDataEventStore

The Core Data implementation offers better scalability and persistence at the cost of some performance overhead:

- **Disk I/O**: Persistence requires disk operations which are slower than memory access
- **Indexed Queries**: Core Data provides indexed access to improve query performance
- **Memory Management**: Core Data manages memory usage with faulting and caching
- **Batch Operations**: Supports batch operations for improved performance with large datasets

## Usage Examples

### Creating and Using an Event Store

```swift
// Create an in-memory store
let inMemoryStore = createEventStore(type: .inMemory)

// Create a CoreData store
let coreDataStore = createEventStore(type: .coreData)

// Create a custom store
let customStore = createEventStore(type: .custom(MyCustomEventStore()))

// Store an event
let event = Event.createWithLatestVersion(
    type: "task.created",
    metadata: EventMetadata(userId: "user123"),
    payload: ["taskId": AnyCodable("task123"), "title": AnyCodable("New Task")]
)
try await eventStore.append(event)

// Retrieve all events
let events = try await eventStore.getEvents()

// Retrieve events by type
let taskEvents = try await eventStore.getEventsByType(type: "task.created")

// Retrieve events by correlation ID
let correlatedEvents = try await eventStore.getEventsByCorrelationId(correlationId: "correlation123")
```

### Pagination

Both storage implementations support pagination for efficient retrieval of large event streams:

```swift
// Get the first 10 events
let firstPage = try await eventStore.getEvents(limit: 10)

// Get the next 10 events
if let lastEventId = firstPage.last?.id {
    let nextPage = try await eventStore.getEvents(startId: lastEventId, limit: 10)
}
```

## Best Practices

When working with the storage system, follow these best practices:

1. **Use the Factory Function**: Always use the `createEventStore` factory function for consistent creation of event stores
2. **Handle Errors**: Properly handle errors from storage operations
3. **Use Pagination**: For large event streams, use pagination to limit memory usage
4. **Register Migrators**: Always register all necessary event migrators before retrieving events
5. **Test with InMemoryEventStore**: Use the in-memory implementation for tests
6. **Transaction Boundaries**: Keep related events in the same transaction

## Storage Extension

The storage system is designed to be extensible. To create a custom event store implementation:

1. **Implement the EventStore Protocol**:

```swift
public class CustomEventStore: EventStore {
    public let migrationRegistry: EventMigrationRegistry
    
    // Implement protocol methods
}
```

2. **Register with the Factory**:

```swift
let customStore = createEventStore(type: .custom(CustomEventStore()))
```

This allows the application to integrate with different storage technologies while maintaining a consistent interface.