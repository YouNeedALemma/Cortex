# Event Sourcing Architecture

MonoSphere implements a robust event sourcing architecture for managing application state. This document explains the event sourcing principles, components, and implementation details in the MonoSphere application.

## What is Event Sourcing?

Event Sourcing is an architectural pattern that involves:

1. **Recording Changes as Events**: Instead of storing the current state, all changes to an application are stored as a sequence of events
2. **Immutable Event Log**: Events are never modified once stored
3. **State Reconstruction**: Application state is derived by replaying events
4. **Complete History**: The event log provides a full audit trail of all changes

## Core Components

### Event

The fundamental unit in event sourcing is the `Event`:

```swift
public struct Event: Equatable, Identifiable, Codable {
    /// Unique identifier for the event
    public let id: String
    
    /// Type of the event (used for routing and processing)
    public let type: String
    
    /// When the event occurred
    public let timestamp: Date
    
    /// Additional information about the event
    public let metadata: EventMetadata
    
    /// Event data payload
    public let payload: [String: AnyCodable]
}

public struct EventMetadata: Equatable, Codable {
    /// User who initiated the event (if applicable)
    public let userId: String?
    
    /// Device from which the event originated
    public let deviceId: String?
    
    /// Schema version of the event
    public let version: String
    
    /// Correlation ID for grouping related events
    public let correlationId: String?
    
    /// Causation ID for tracking event chains
    public let causationId: String?
}
```

### Event Store

The `EventStore` protocol defines how events are stored and retrieved:

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
    
    /// Migration registry for evolving event schemas
    var migrationRegistry: EventMigrationRegistry { get }
}
```

MonoSphere provides these implementations:

1. **InMemoryEventStore**: For development and testing
2. **CoreDataEventStore**: For persistent storage on iOS

### Projections

Projections transform event streams into application state:

```swift
public struct StateContainer: Equatable {
    /// All projects indexed by ID
    public var projects: [String: Project]
    
    /// All tasks indexed by ID
    public var tasks: [String: Task]
}

/// Project events to application state
public func projectEvents(_ events: [Event], initialState: StateContainer = createEmptyState()) -> StateContainer {
    // Apply each event to the state
    return events.reduce(initialState) { state, event in
        applyEvent(event, to: state)
    }
}
```

### Event Handlers

Event handlers process specific event types:

```swift
/// Function to apply an event to the current state
private func applyEvent(_ event: Event, to state: StateContainer) -> StateContainer {
    var newState = state
    
    switch event.type {
    case TaskEventType.created:
        newState = handleTaskCreated(event, state: newState)
    case TaskEventType.updated:
        newState = handleTaskUpdated(event, state: newState)
    case TaskEventType.deleted:
        newState = handleTaskDeleted(event, state: newState)
    case ProjectEventType.created:
        newState = handleProjectCreated(event, state: newState)
    // Other event types...
    default:
        // Unknown event type, state unchanged
        break
    }
    
    return newState
}
```

### Event Types

MonoSphere defines specific event types for domain changes:

```swift
public enum TaskEventType {
    public static let created = "task.created"
    public static let updated = "task.updated"
    public static let deleted = "task.deleted"
    public static let completed = "task.completed"
    public static let uncompleted = "task.uncompleted"
    public static let priorityChanged = "task.priority.changed"
    public static let assignedToProject = "task.assigned.to.project"
    public static let removedFromProject = "task.removed.from.project"
}

public enum ProjectEventType {
    public static let created = "project.created"
    public static let updated = "project.updated"
    public static let deleted = "project.deleted"
    public static let completed = "project.completed"
    public static let statusChanged = "project.status.changed"
    public static let taskAdded = "project.task.added"
    public static let taskRemoved = "project.task.removed"
}
```

## Event Creation

Events are created through factory functions that ensure proper structure:

```swift
public static func createTaskCreatedEvent(
    taskId: String,
    title: String,
    description: String,
    priority: TaskPriority,
    dueDate: Date? = nil,
    tags: [String] = [],
    userId: String? = nil,
    deviceId: String? = nil,
    correlationId: String? = nil,
    causationId: String? = nil
) -> Event {
    let payload: [String: AnyCodable] = [
        "id": AnyCodable(taskId),
        "title": AnyCodable(title),
        "description": AnyCodable(description),
        "priority": AnyCodable(priority.rawValue),
        "dueDate": AnyCodable(dueDate),
        "tags": AnyCodable(tags),
        "isCompleted": AnyCodable(false),
        "createdAt": AnyCodable(Date()),
        "updatedAt": AnyCodable(Date())
    ]
    
    let metadata = EventMetadata(
        userId: userId,
        deviceId: deviceId,
        correlationId: correlationId,
        causationId: causationId
    )
    
    return Event.createWithLatestVersion(
        type: TaskEventType.created,
        metadata: metadata,
        payload: payload
    )
}
```

## Command Flow

The typical flow in event sourcing follows this pattern:

1. **Command Creation**: User initiates an action (e.g., create a task)
2. **Command Validation**: Business rules are checked
3. **Event Generation**: Valid commands produce events
4. **Event Storage**: Events are appended to the event store
5. **State Projection**: Events are projected to build new state
6. **View Update**: UI reflects the new state

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│             │     │             │     │             │     │             │
│  Command    │────►│  Events     │────►│  Event      │────►│  Projected  │
│             │     │             │     │  Store      │     │  State      │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                                                                   │
                                                                   ▼
                                                            ┌─────────────┐
                                                            │             │
                                                            │  User       │
                                                            │  Interface  │
                                                            │             │
                                                            └─────────────┘
```

## Event Sourcing Benefits

MonoSphere leverages these event sourcing benefits:

1. **Complete Audit Trail**: Every change is recorded and can be reviewed
2. **Temporal Queries**: Application state can be reconstructed for any point in time
3. **Domain Focus**: Business events are explicitly modeled and named
4. **Immutability**: Events are immutable, simplifying concurrency management
5. **Extensibility**: New projections can be added to derive different views of the same data
6. **Debuggability**: System behavior can be reproduced by replaying events

## Event Schema Evolution

As the application evolves, event schemas may need to change. MonoSphere handles this through:

1. **Versioned Events**: Each event has a version in its metadata
2. **Migration Registry**: Central registry for event migrations
3. **Migration Path**: Clear path from old to new schemas
4. **Automatic Migration**: Events are migrated when retrieved

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

## Event Serialization

Events are serialized for storage using the `AnyCodable` type:

```swift
public struct AnyCodable: Codable, Equatable {
    /// The wrapped value
    public var value: Any
    
    /// Initialize with a Codable value
    public init<T: Codable & Equatable>(_ value: T)
    
    /// Encoding logic
    public func encode(to encoder: Encoder) throws
    
    /// Decoding logic
    public init(from decoder: Decoder) throws
}
```

This allows type-safe encoding and decoding of heterogeneous event payloads.

## Implementation Patterns

MonoSphere implements these event sourcing patterns:

1. **Event Factory Functions**: Helper methods to create well-formed events
2. **Event Handler Functions**: Pure functions that apply events to state
3. **Snapshot Projections**: Build complete state from event streams
4. **Event Versioning**: Support for event schema evolution
5. **Correlation and Causation**: Track related events and event chains

## Example Flow

```swift
// 1. Command is created
let createTaskCommand = CreateTaskCommand(
    title: "Implement event sourcing",
    description: "Set up event sourcing architecture",
    priority: .high,
    tags: ["architecture", "development"]
)

// 2. Command is executed
let result = await commandHandler.execute(createTaskCommand)

// 3. Command handler validates and creates event
let event = TaskEvents.createTaskCreatedEvent(
    taskId: UUID().uuidString,
    title: command.title,
    description: command.description,
    priority: command.priority,
    tags: command.tags
)

// 4. Event is persisted
try await eventStore.append(event)

// 5. State is updated by projecting events
let updatedState = projectEvents([event], initialState: currentState)

// 6. UI is updated with new state
updateView(with: updatedState)
```

## Advanced Techniques

MonoSphere employs these advanced event sourcing techniques:

1. **Pure Projections**: Projections are pure functions without side effects
2. **Domain Invariants**: Business rules enforced before event creation
3. **Event Enrichment**: Common metadata added to events (user, device, timestamps)
4. **Event Streaming**: Events can be streamed with pagination for large histories
5. **Correlation Groups**: Related events grouped by correlation ID

## Design Considerations

When working with MonoSphere's event sourcing system, keep in mind:

1. **Event Granularity**: Events should represent meaningful business changes
2. **Event Naming**: Use past tense, domain-specific terms (TaskCreated, ProjectCompleted)
3. **Schema Evolution**: Plan for event schema changes with proper versioning
4. **Idempotency**: Event handlers should be idempotent (can be applied multiple times)
5. **Performance**: Consider snapshot projections for large event histories