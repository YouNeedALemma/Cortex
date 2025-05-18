# Event Versioning and Migration System

In an event-sourced system like MonoSphere, events are immutable records of state changes that are persisted over time. As the application evolves, the structure of these events may need to change. The event versioning and migration system allows MonoSphere to handle schema evolution while maintaining compatibility with historical events.

## Overview

The event versioning system in MonoSphere provides:

1. **Schema Versioning**: Explicit versioning of event structures
2. **Migration Path**: Clear path for evolving events between versions
3. **Automatic Migration**: Transparent upgrading of events when retrieved
4. **Backward Compatibility**: Older events remain usable with newer code

## Core Components

### Event Version

The `EventVersion` struct represents a semantic version:

```swift
public struct EventVersion: Comparable, Equatable, Codable {
    /// Major version component (breaking changes)
    public let major: Int
    
    /// Minor version component (backwards-compatible additions)
    public let minor: Int
    
    /// Patch version component (backwards-compatible fixes)
    public let patch: Int
    
    /// Initialize with version components
    public init(major: Int = 1, minor: Int = 0, patch: Int = 0)
    
    /// Initialize from a string like "1.2.3"
    public init?(string: String)
    
    /// Convert to string (e.g., "1.2.3")
    public var stringValue: String
    
    /// Check compatibility between versions
    public func isCompatibleWith(_ other: EventVersion) -> Bool
}
```

The `EventVersion` follows semantic versioning principles:

- **Major**: Incompatible API changes
- **Minor**: Functionality added in a backwards-compatible manner
- **Patch**: Backwards-compatible bug fixes

### Event Migrator

The `EventMigrator` protocol defines how to transform events between versions:

```swift
public protocol EventMigrator {
    /// The event type this migrator handles
    var eventType: String { get }
    
    /// The source version this migrator handles
    var sourceVersion: EventVersion { get }
    
    /// The target version this migrator produces
    var targetVersion: EventVersion { get }
    
    /// Migrate event payload from source version to target version
    func migrate(payload: [String: AnyCodable]) -> [String: AnyCodable]
}
```

Migrators are specific to event types and version pairs, focusing on a single migration step.

### Migration Registry

The `EventMigrationRegistry` manages event migrators and migration paths:

```swift
public class EventMigrationRegistry {
    /// Shared instance for application-wide use
    public static let shared = EventMigrationRegistry()
    
    /// Register a migrator
    public func register(migrator: EventMigrator)
    
    /// Get migration path from source version to latest version
    public func getMigrationPath(eventType: String, sourceVersion: EventVersion) -> [EventMigrator]
    
    /// Get the latest version for an event type
    public func getLatestVersion(eventType: String) -> EventVersion
    
    /// Apply migrations to an event
    public func migrateEvent(_ event: Event) -> Event
}
```

The registry serves as the central point for event migration, determining:

1. What migrations are available
2. What migration path to take for a given event
3. What the latest version is for each event type
4. How to apply migrations to events

## How It Works

### Event Creation

When events are created, they include version information:

```swift
// Create an event with the latest version
public static func createWithLatestVersion(
    id: String = UUID().uuidString,
    type: String,
    timestamp: Date = Date(),
    metadata: EventMetadata,
    payload: [String: AnyCodable],
    registry: EventMigrationRegistry = .shared
) -> Event {
    let latestVersion = registry.getLatestVersion(eventType: type)
    
    let updatedMetadata = EventMetadata(
        userId: metadata.userId,
        deviceId: metadata.deviceId,
        version: latestVersion.stringValue,
        correlationId: metadata.correlationId,
        causationId: metadata.causationId
    )
    
    return Event(
        id: id,
        type: type,
        timestamp: timestamp,
        metadata: updatedMetadata,
        payload: payload
    )
}
```

### Event Retrieval and Migration

When events are retrieved from storage, they are automatically migrated:

```swift
// In EventStore implementations
public func getEvents(startId: String? = nil, limit: Int? = nil) async throws -> [Event] {
    var result = events
    
    // Apply filtering and pagination...
    
    // Apply migrations if needed
    return result.map { migrationRegistry.migrateEvent($0) }
}
```

### Migration Process

The migration process follows these steps:

1. **Version Parsing**: Extract and parse the event's version
2. **Latest Version Check**: Determine the latest version for the event type
3. **Path Determination**: Find the migration path from current to latest version
4. **Sequential Migration**: Apply each migrator in the path
5. **Event Reconstruction**: Create a new event with migrated payload and updated version

```swift
public func migrateEvent(_ event: Event) -> Event {
    guard let version = EventVersion(string: event.metadata.version) else {
        return event
    }
    
    let latestVersion = getLatestVersion(eventType: event.type)
    
    if version == latestVersion {
        return event // Already at latest version
    }
    
    let migrationPath = getMigrationPath(eventType: event.type, sourceVersion: version)
    
    if migrationPath.isEmpty {
        return event // No migration path found
    }
    
    var payload = event.payload
    
    for migrator in migrationPath {
        payload = migrator.migrate(payload: payload)
    }
    
    // Create a new event with migrated payload and updated version
    let updatedMetadata = EventMetadata(
        userId: event.metadata.userId,
        deviceId: event.metadata.deviceId,
        version: latestVersion.stringValue,
        correlationId: event.metadata.correlationId,
        causationId: event.metadata.causationId
    )
    
    return Event(
        id: event.id,
        type: event.type,
        timestamp: event.timestamp,
        metadata: updatedMetadata,
        payload: payload
    )
}
```

## Migration Examples

### Adding a New Field

When a new field is added to an event schema:

```swift
public class TaskCreatedV1_0_0ToV1_1_0Migrator: EventMigrator {
    public let eventType = TaskEventType.created
    public let sourceVersion = EventVersion(major: 1, minor: 0, patch: 0)
    public let targetVersion = EventVersion(major: 1, minor: 1, patch: 0)
    
    public func migrate(payload: [String: AnyCodable]) -> [String: AnyCodable] {
        var newPayload = payload
        
        // Add a new field that wasn't in v1.0.0
        if newPayload["estimatedHours"] == nil {
            newPayload["estimatedHours"] = AnyCodable(0)
        }
        
        return newPayload
    }
}
```

### Renaming a Field

When a field is renamed:

```swift
public class TaskCreatedV1_1_0ToV1_2_0Migrator: EventMigrator {
    public let eventType = TaskEventType.created
    public let sourceVersion = EventVersion(major: 1, minor: 1, patch: 0)
    public let targetVersion = EventVersion(major: 1, minor: 2, patch: 0)
    
    public func migrate(payload: [String: AnyCodable]) -> [String: AnyCodable] {
        var newPayload = payload
        
        // Rename "isComplete" to "isCompleted"
        if let isComplete = newPayload["isComplete"] {
            newPayload["isCompleted"] = isComplete
            newPayload.removeValue(forKey: "isComplete")
        }
        
        return newPayload
    }
}
```

### Changing Value Types

When a field's type changes:

```swift
public class TaskCreatedV1_2_0ToV2_0_0Migrator: EventMigrator {
    public let eventType = TaskEventType.created
    public let sourceVersion = EventVersion(major: 1, minor: 2, patch: 0)
    public let targetVersion = EventVersion(major: 2, minor: 0, patch: 0)
    
    public func migrate(payload: [String: AnyCodable]) -> [String: AnyCodable] {
        var newPayload = payload
        
        // Convert priority from string to integer
        if let priorityString = newPayload["priority"]?.value as? String {
            switch priorityString {
            case "low":
                newPayload["priority"] = AnyCodable(1)
            case "medium":
                newPayload["priority"] = AnyCodable(2)
            case "high":
                newPayload["priority"] = AnyCodable(3)
            case "urgent":
                newPayload["priority"] = AnyCodable(4)
            default:
                newPayload["priority"] = AnyCodable(1)
            }
        }
        
        return newPayload
    }
}
```

## Registration and Setup

Migrators are registered at application startup:

```swift
func registerMigrators() {
    let registry = EventMigrationRegistry.shared
    
    // Register task migrators
    registry.register(migrator: TaskCreatedV1_0_0ToV1_1_0Migrator())
    registry.register(migrator: TaskUpdatedV1_0_0ToV1_1_0Migrator())
    
    // Register project migrators
    registry.register(migrator: ProjectCreatedV1_0_0ToV1_1_0Migrator())
}
```

## Version Compatibility

MonoSphere follows these version compatibility rules:

1. **Major Version**: Incompatible changes that require explicit migration
2. **Minor Version**: Backward-compatible additions (new fields)
3. **Patch Version**: Backward-compatible fixes (no schema changes)

```swift
public func isCompatibleWith(_ other: EventVersion) -> Bool {
    // Major versions must match for compatibility
    return self.major == other.major
}
```

## Migration Path Determination

The system finds an optimal migration path:

```swift
public func getMigrationPath(eventType: String, sourceVersion: EventVersion) -> [EventMigrator] {
    guard let typeMigrators = migrators[eventType] else {
        return []
    }
    
    let sortedMigrators = typeMigrators.sorted { $0.sourceVersion < $1.sourceVersion }
    
    var result: [EventMigrator] = []
    var currentVersion = sourceVersion
    
    while let nextMigrator = sortedMigrators.first(where: { 
        $0.sourceVersion == currentVersion && $0.targetVersion > currentVersion
    }) {
        result.append(nextMigrator)
        currentVersion = nextMigrator.targetVersion
    }
    
    return result
}
```

This approach:
1. Finds migrators specific to the event type
2. Sorts them by source version
3. Builds a path starting from the event's current version
4. Adds migrators that form a continuous path to the latest version

## Best Practices

When working with the event versioning system, follow these practices:

1. **Incremental Versions**: Create migrations between adjacent versions
2. **One-Way Migration**: Migrations should only go forward, not backward
3. **Test Migrations**: Write tests for all migrations
4. **Documentation**: Document schema changes carefully
5. **Default Values**: Provide reasonable defaults for new fields
6. **Backward Compatibility**: Maintain compatibility when possible

## Advanced Techniques

MonoSphere supports these advanced versioning techniques:

1. **Multi-Step Migrations**: Complex changes can use multiple migrators
2. **Fallback Values**: Migrations can provide fallback values for missing data
3. **Data Conversion**: Migrations can transform data formats
4. **Version Leapfrogging**: Events can skip intermediate versions if migrators exist
5. **Migration Validation**: Validate migrated events for integrity

## Design Considerations

The versioning system addresses these design challenges:

1. **Event Immutability**: Events must remain immutable, so migration creates new events
2. **Performance**: Migration happens on read to avoid updating stored events
3. **Migration Errors**: The system handles missing or invalid migrations gracefully
4. **Version Compatibility**: The system enforces semantic versioning rules
5. **Extensibility**: New migrators can be added as the system evolves