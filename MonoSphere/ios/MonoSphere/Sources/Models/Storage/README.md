# MonoSphere Event Storage

This module provides event storage implementations for the MonoSphere application.

## Overview

The MonoSphere event storage system is responsible for persisting events and retrieving them for state reconstruction. The system follows the event sourcing pattern, where all changes to application state are recorded as a sequence of immutable events.

## Components

### EventStore Protocol

The `EventStore` protocol defines the interface for event storage:

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
    
    /// Optional: Apply migrations to events as they are read
    var migrationRegistry: EventMigrationRegistry { get }
}
```

### Implementations

#### InMemoryEventStore

The `InMemoryEventStore` provides a simple in-memory implementation for development and testing. Events are stored in memory and lost when the application is terminated.

#### CoreDataEventStore

The `CoreDataEventStore` provides a persistent implementation using Core Data. Events are stored in a local database and persisted across application launches.

### Supporting Classes

#### EventDataModel

Defines the Core Data model for storing events and provides methods for creating and managing Core Data contexts.

#### CoreDataSerialization

Provides utilities for converting between Event models and Core Data entities.

## Usage

To create an event store:

```swift
// Create an in-memory store
let inMemoryStore = createEventStore(type: .inMemory)

// Create a CoreData store
let coreDataStore = createEventStore(type: .coreData)

// Create a custom store
let customStore = createEventStore(type: .custom(MyCustomEventStore()))
```

To store and retrieve events:

```swift
// Store an event
try await eventStore.append(event)

// Retrieve all events
let events = try await eventStore.getEvents()

// Retrieve events by type
let taskEvents = try await eventStore.getEventsByType(type: "task.created")

// Retrieve events by correlation ID
let correlatedEvents = try await eventStore.getEventsByCorrelationId(correlationId: correlationId)
```

## Event Migration

The event storage system supports event schema migration through the `EventMigrationRegistry`. When events are retrieved, they are automatically migrated to the latest schema version if needed.

To register event migrators:

```swift
let registry = EventMigrationRegistry.shared
registry.register(migrator: TaskCreatedV1_0_0ToV1_1_0Migrator())
```

## Testing

The `CoreDataEventStoreTests` class provides tests for the Core Data implementation. It uses an in-memory Core Data store for testing.