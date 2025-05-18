# MonoSphere Models

This directory contains the core domain models and the event sourcing infrastructure for the MonoSphere iOS application.

## Overview

The Models module provides the foundation for MonoSphere's domain logic through:

1. **Domain Models**: Immutable value types representing core concepts
2. **Event Sourcing**: Infrastructure for recording and replaying state changes
3. **Command Handling**: Processing user actions according to business rules
4. **Validation**: Ensuring data integrity and business rule compliance

All models follow functional programming principles with:
- Immutable value types
- Pure functions
- Explicit error handling
- Type safety

## Domain Models

### Project

The `Project` model represents a collection of related tasks and associated metadata:

```swift
public struct Project: Equatable, Identifiable, Codable {
    public let id: String
    public let name: String
    public let description: String
    public let status: ProjectStatus
    public let tags: [String]
    public let createdAt: Date
    public let updatedAt: Date
    public let completedAt: Date?
    public let taskIds: [String]
}

public enum ProjectStatus: String, Equatable, Codable {
    case active
    case onHold
    case completed
    case cancelled
}
```

**Usage**:
- Organizing related tasks
- Tracking project state and progress
- Managing project metadata (tags, status)
- Maintaining relationship with tasks

### Task

The `Task` model represents an individual work item with associated metadata:

```swift
public struct Task: Equatable, Identifiable, Codable {
    public let id: String
    public let title: String
    public let description: String
    public let priority: TaskPriority
    public let dueDate: Date?
    public let isCompleted: Bool
    public let completedAt: Date?
    public let createdAt: Date
    public let updatedAt: Date
    public let projectId: String?
    public let tags: [String]
}

public enum TaskPriority: String, Equatable, Codable, CaseIterable {
    case low
    case medium
    case high
    case urgent
}
```

**Usage**:
- Tracking individual work items
- Managing task state (completion, priority)
- Organizing tasks within projects
- Setting deadlines and priorities

### Event

The `Event` model is the cornerstone of the event sourcing system:

```swift
public struct Event: Equatable, Identifiable, Codable {
    public let id: String
    public let type: String
    public let timestamp: Date
    public let metadata: EventMetadata
    public let payload: [String: AnyCodable]
}

public struct EventMetadata: Equatable, Codable {
    public let userId: String?
    public let deviceId: String?
    public let version: String
    public let correlationId: String?
    public let causationId: String?
}
```

**Usage**:
- Recording all state changes in the system
- Storing event payloads in a type-safe manner
- Tracking metadata for events (user, device, version)
- Supporting event versioning and migration

## Event Types

### Project Events

The `ProjectEvents` module defines events related to project lifecycle:

- `ProjectCreated`: A new project has been created
- `ProjectUpdated`: Project details have been modified
- `ProjectDeleted`: A project has been removed
- `ProjectCompleted`: A project has been marked as completed
- `ProjectTaskAdded`: A task has been associated with a project
- `ProjectTaskRemoved`: A task has been disassociated from a project

### Task Events

The `TaskEvents` module defines events related to task lifecycle:

- `TaskCreated`: A new task has been created
- `TaskUpdated`: Task details have been modified
- `TaskDeleted`: A task has been removed
- `TaskCompleted`: A task has been marked as completed
- `TaskUncompleted`: A task has been marked as incomplete
- `TaskPriorityChanged`: Task priority has been changed
- `TaskAssignedToProject`: Task has been assigned to a project
- `TaskRemovedFromProject`: Task has been removed from a project

## Event Infrastructure

### Event Store

The `EventStore` protocol defines how events are stored and retrieved:

```swift
public protocol EventStore {
    func append(_ event: Event) async throws
    func getEvents(startId: String?, limit: Int?) async throws -> [Event]
    func getEventsByType(type: String, startId: String?, limit: Int?) async throws -> [Event]
    func getEventsByCorrelationId(correlationId: String) async throws -> [Event]
    var migrationRegistry: EventMigrationRegistry { get }
}
```

**Implementations**:
- `InMemoryEventStore`: Volatile storage for development and testing
- `CoreDataEventStore`: Persistent storage using CoreData

### Projections

The `Projections` module is responsible for building application state from events:

```swift
public struct StateContainer: Equatable {
    public var projects: [String: Project]
    public var tasks: [String: Task]
}

// Function to project events to state
public func projectEvents(_ events: [Event], initialState: StateContainer = createEmptyState()) -> StateContainer { ... }
```

**Functionality**:
- Applying events to build the current application state
- Maintaining referential integrity between models
- Supporting different projection types (snapshot, specific entity)

### Command Handler

The `CommandHandler` class processes user commands and generates events:

```swift
public class CommandHandler {
    private let eventStore: EventStore
    private let state: StateContainer
    
    // Process commands and emit events
    public func execute<T>(_ command: Command) async -> CommandResult<T> { ... }
}
```

**Responsibilities**:
- Validating user input against business rules
- Applying domain logic
- Generating appropriate events
- Handling errors and validation failures

## Error Handling

The `DomainError` enum provides a comprehensive type system for domain errors:

```swift
public enum DomainError: Error, Equatable {
    // Task errors
    case taskNotFound(id: String)
    case invalidTaskTitle(reason: String)
    case invalidTaskDescription(reason: String)
    
    // Project errors
    case projectNotFound(id: String)
    case invalidProjectName(reason: String)
    case projectAlreadyCompleted(id: String)
    
    // Validation errors
    case validationFailed(reason: String)
    
    // Event errors
    case eventStoreError(reason: String)
    case eventMigrationFailed(reason: String)
}
```

**Usage**:
- Providing specific, actionable error information
- Supporting error recovery strategies
- Maintaining type safety in error handling

## Event Versioning

The `EventVersioning` module provides schema evolution capabilities:

```swift
public struct EventVersion: Comparable, Equatable, Codable {
    public let major: Int
    public let minor: Int
    public let patch: Int
}

public protocol EventMigrator {
    var eventType: String { get }
    var sourceVersion: EventVersion { get }
    var targetVersion: EventVersion { get }
    func migrate(payload: [String: AnyCodable]) -> [String: AnyCodable]
}

public class EventMigrationRegistry {
    public func register(migrator: EventMigrator)
    public func migrateEvent(_ event: Event) -> Event
}
```

**Capabilities**:
- Semantic versioning for events
- Migration path for schema evolution
- Automatic event migration during retrieval

## Usage Example

```swift
// Create a command handler
let eventStore = createEventStore(type: .coreData)
let state = await projectEvents(try await eventStore.getEvents())
let commandHandler = CommandHandler(eventStore: eventStore, state: state)

// Execute a command
let createTaskCommand = CreateTaskCommand(
    title: "Implement feature X",
    description: "Add support for feature X in the application",
    priority: .high,
    dueDate: Date().addingTimeInterval(86400), // Tomorrow
    tags: ["development", "feature"]
)

let result = await commandHandler.execute(createTaskCommand)

switch result {
case .success(let task):
    print("Task created with ID: \(task.id)")
case .failure(let error):
    print("Failed to create task: \(error)")
}
```

## Design Principles

1. **Immutability**: All domain models are immutable value types
2. **Separation of Concerns**: Clear boundaries between components
3. **Function Purity**: Pure functions with no side effects
4. **Error Explicitness**: Comprehensive error types and handling
5. **Type Safety**: Leverage Swift's type system for compile-time checks

## Additional Resources

- See [Event Sourcing Architecture](./Events/README.md) for more details on event sourcing
- See [Storage Implementations](./Storage/README.md) for persistent storage details
- See [Command Handling](./CommandHandler.md) for details on command processing