# MonoSphere Swift API Reference

This document provides a comprehensive reference for the MonoSphere Swift API. It covers the key interfaces, classes, and types that make up the MonoSphere iOS application's architecture.

## Table of Contents

- [Core Domain Models](#core-domain-models)
  - [Task](#task)
  - [Project](#project)
- [Event Sourcing](#event-sourcing)
  - [Event](#event)
  - [EventStore](#eventstore)
  - [EventMetadata](#eventmetadata)
  - [Projections](#projections)
- [Command Handling](#command-handling)
  - [Command](#command)
  - [CommandHandler](#commandhandler)
  - [CommandResult](#commandresult)
- [Error Handling](#error-handling)
  - [DomainError](#domainerror)
  - [ValidationResult](#validationresult)
- [Validation](#validation)
  - [Validator](#validator)
- [Event Versioning](#event-versioning)
  - [EventVersion](#eventversion)
  - [EventMigrator](#eventmigrator)
  - [EventMigrationRegistry](#eventmigrationregistry)
- [Storage](#storage)
  - [InMemoryEventStore](#inmemoryeventstore)
  - [CoreDataEventStore](#coredataeventstore)
  - [EventDataModel](#eventdatamodel)

## Core Domain Models

### Task

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
```

**TaskPriority**
```swift
public enum TaskPriority: String, Equatable, Codable, CaseIterable {
    case low
    case medium
    case high
    case urgent
}
```

### Project

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
```

**ProjectStatus**
```swift
public enum ProjectStatus: String, Equatable, Codable {
    case active
    case onHold
    case completed
    case cancelled
}
```

## Event Sourcing

### Event

```swift
public struct Event: Equatable, Identifiable, Codable {
    public let id: String
    public let type: String
    public let timestamp: Date
    public let metadata: EventMetadata
    public let payload: [String: AnyCodable]
    
    public init(
        id: String,
        type: String,
        timestamp: Date,
        metadata: EventMetadata,
        payload: [String: AnyCodable]
    )
    
    public static func createWithLatestVersion(
        id: String = UUID().uuidString,
        type: String,
        timestamp: Date = Date(),
        metadata: EventMetadata,
        payload: [String: AnyCodable],
        registry: EventMigrationRegistry = .shared
    ) -> Event
}
```

### EventStore

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

### EventMetadata

```swift
public struct EventMetadata: Equatable, Codable {
    public let userId: String?
    public let deviceId: String?
    public let version: String
    public let correlationId: String?
    public let causationId: String?
    
    public init(
        userId: String? = nil,
        deviceId: String? = nil,
        version: String = "1.0.0",
        correlationId: String? = nil,
        causationId: String? = nil
    )
    
    public var versionAsEventVersion: EventVersion?
}
```

### Projections

```swift
public struct StateContainer: Equatable {
    public var projects: [String: Project]
    public var tasks: [String: Task]
    
    public init(projects: [String: Project] = [:], tasks: [String: Task] = [:])
}

/// Create an empty state container
public func createEmptyState() -> StateContainer

/// Project events to build application state
public func projectEvents(_ events: [Event], initialState: StateContainer = createEmptyState()) -> StateContainer

/// Project events for a specific task
public func projectTaskEvents(taskId: String, events: [Event], initialTask: Task? = nil) -> Task?

/// Project events for a specific project
public func projectProjectEvents(projectId: String, events: [Event], initialProject: Project? = nil) -> Project?
```

## Command Handling

### Command

```swift
public protocol Command {
    associatedtype ResultType
    
    func validate(state: StateContainer, validator: Validator) -> ValidationResult
    func execute(state: StateContainer, eventStore: EventStore) async throws -> CommandResult<ResultType>
}
```

### CommandHandler

```swift
public class CommandHandler {
    private let eventStore: EventStore
    private let state: StateContainer
    private let validator: Validator
    
    public init(eventStore: EventStore, state: StateContainer)
    
    public func execute<T>(_ command: Command) async -> CommandResult<T> where T == Command.ResultType
}
```

### CommandResult

```swift
public enum CommandResult<T> {
    case success(T)
    case failure(DomainError)
    
    public func map<U>(_ transform: (T) -> U) -> CommandResult<U>
    public func flatMap<U>(_ transform: (T) -> CommandResult<U>) -> CommandResult<U>
}
```

## Error Handling

### DomainError

```swift
public enum DomainError: Error, Equatable {
    // General errors
    case invalidInput(String)
    case notFound(type: String, id: String)
    case permissionDenied(String)
    case concurrencyConflict(String)
    
    // Task-specific errors
    case taskNotFound(id: String)
    case taskValidationFailed(reason: String)
    case taskAlreadyCompleted(id: String)
    case taskAlreadyUncompleted(id: String)
    
    // Project-specific errors
    case projectNotFound(id: String)
    case projectValidationFailed(reason: String)
    case projectContainsTask(projectId: String, taskId: String)
    case projectDoesNotContainTask(projectId: String, taskId: String)
    
    // Event-specific errors
    case eventStoreError(String)
    case eventNotFound(id: String)
    case eventValidationFailed(reason: String)
    case eventVersionMismatch(expected: String, actual: String)
    
    // Sync-specific errors
    case syncFailed(reason: String)
    case networkError(String)
    case conflictResolutionFailed(String)
    
    public var localizedDescription: String
}
```

### ValidationResult

```swift
public enum ValidationResult {
    case valid
    case invalid(reason: String)
    
    public var isValid: Bool
}
```

## Validation

### Validator

```swift
public struct Validator {
    // Task validation
    public static func validateTaskTitle(_ title: String) -> Result<String, DomainError>
    public static func validateTaskDescription(_ description: String) -> Result<String, DomainError>
    public static func validateTaskDueDate(_ dueDate: Date?) -> Result<Date?, DomainError>
    public static func validateTaskTags(_ tags: [String]) -> Result<[String], DomainError>
    
    // Project validation
    public static func validateProjectName(_ name: String) -> Result<String, DomainError>
    public static func validateProjectDescription(_ description: String) -> Result<String, DomainError>
    public static func validateProjectTags(_ tags: [String]) -> Result<[String], DomainError>
    
    // Helper functions
    public static func validateUUID(_ id: String) -> Result<String, DomainError>
}
```

## Event Versioning

### EventVersion

```swift
public struct EventVersion: Comparable, Equatable, Codable {
    public let major: Int
    public let minor: Int
    public let patch: Int
    
    public init(major: Int = 1, minor: Int = 0, patch: Int = 0)
    public init?(string: String)
    
    public var stringValue: String
    public func isCompatibleWith(_ other: EventVersion) -> Bool
    
    public static func < (lhs: EventVersion, rhs: EventVersion) -> Bool
}
```

### EventMigrator

```swift
public protocol EventMigrator {
    var eventType: String { get }
    var sourceVersion: EventVersion { get }
    var targetVersion: EventVersion { get }
    
    func migrate(payload: [String: AnyCodable]) -> [String: AnyCodable]
}
```

### EventMigrationRegistry

```swift
public class EventMigrationRegistry {
    public static let shared = EventMigrationRegistry()
    
    public init()
    
    public func register(migrator: EventMigrator)
    public func getMigrationPath(eventType: String, sourceVersion: EventVersion) -> [EventMigrator]
    public func getLatestVersion(eventType: String) -> EventVersion
    public func migrateEvent(_ event: Event) -> Event
}
```

## Storage

### InMemoryEventStore

```swift
public class InMemoryEventStore: EventStore {
    private var events: [Event] = []
    public let migrationRegistry: EventMigrationRegistry
    
    public init(migrationRegistry: EventMigrationRegistry = .shared)
    
    public func append(_ event: Event) async throws
    public func getEvents(startId: String? = nil, limit: Int? = nil) async throws -> [Event]
    public func getEventsByType(type: String, startId: String? = nil, limit: Int? = nil) async throws -> [Event]
    public func getEventsByCorrelationId(correlationId: String) async throws -> [Event]
}
```

### CoreDataEventStore

```swift
public class CoreDataEventStore: EventStore {
    private let dataModel: EventDataModel
    public let migrationRegistry: EventMigrationRegistry
    
    public init(
        dataModel: EventDataModel = .shared,
        migrationRegistry: EventMigrationRegistry = .shared
    )
    
    public func append(_ event: Event) async throws
    public func getEvents(startId: String? = nil, limit: Int? = nil) async throws -> [Event]
    public func getEventsByType(type: String, startId: String? = nil, limit: Int? = nil) async throws -> [Event]
    public func getEventsByCorrelationId(correlationId: String) async throws -> [Event]
    public func clearAll() async throws
}
```

### EventDataModel

```swift
public class EventDataModel {
    public static let shared = EventDataModel()
    
    public var viewContext: NSManagedObjectContext
    public func newBackgroundContext() -> NSManagedObjectContext
    public func saveContext(_ context: NSManagedObjectContext) throws
}
```

## Storage Factory

```swift
public enum EventStoreType {
    case inMemory
    case coreData
    case custom(EventStore)
}

public func createEventStore(
    type: EventStoreType,
    migrationRegistry: EventMigrationRegistry = .shared
) -> EventStore
```

## Event Type Constants

### TaskEventType

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
```

### ProjectEventType

```swift
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

## Task Commands

### CreateTaskCommand

```swift
public struct CreateTaskCommand: Command {
    public typealias ResultType = Task
    
    public let title: String
    public let description: String
    public let priority: TaskPriority
    public let dueDate: Date?
    public let tags: [String]
    
    public init(
        title: String,
        description: String = "",
        priority: TaskPriority = .medium,
        dueDate: Date? = nil,
        tags: [String] = []
    )
    
    public func validate(state: StateContainer, validator: Validator) -> ValidationResult
    public func execute(state: StateContainer, eventStore: EventStore) async throws -> CommandResult<Task>
}
```

### CompleteTaskCommand

```swift
public struct CompleteTaskCommand: Command {
    public typealias ResultType = Task
    
    public let taskId: String
    
    public init(taskId: String)
    
    public func validate(state: StateContainer, validator: Validator) -> ValidationResult
    public func execute(state: StateContainer, eventStore: EventStore) async throws -> CommandResult<Task>
}
```

## Project Commands

### CreateProjectCommand

```swift
public struct CreateProjectCommand: Command {
    public typealias ResultType = Project
    
    public let name: String
    public let description: String
    public let tags: [String]
    
    public init(
        name: String,
        description: String = "",
        tags: [String] = []
    )
    
    public func validate(state: StateContainer, validator: Validator) -> ValidationResult
    public func execute(state: StateContainer, eventStore: EventStore) async throws -> CommandResult<Project>
}
```

### AddTaskToProjectCommand

```swift
public struct AddTaskToProjectCommand: Command {
    public typealias ResultType = (Project, Task)
    
    public let projectId: String
    public let taskId: String
    
    public init(projectId: String, taskId: String)
    
    public func validate(state: StateContainer, validator: Validator) -> ValidationResult
    public func execute(state: StateContainer, eventStore: EventStore) async throws -> CommandResult<(Project, Task)>
}
```

## Utility Types

### AnyCodable

```swift
public struct AnyCodable: Codable, Equatable {
    public var value: Any
    
    public init<T: Codable & Equatable>(_ value: T)
    
    public func encode(to encoder: Encoder) throws
    public init(from decoder: Decoder) throws
    
    public static func == (lhs: AnyCodable, rhs: AnyCodable) -> Bool
}
```

## Usage Examples

### Creating and Using CommandHandler

```swift
// Create event store
let eventStore = createEventStore(type: .coreData)

// Load events and project state
let events = try await eventStore.getEvents()
let state = projectEvents(events)

// Create command handler
let commandHandler = CommandHandler(eventStore: eventStore, state: state)

// Execute a command
let createTaskCommand = CreateTaskCommand(
    title: "Implement API documentation",
    description: "Create comprehensive API docs for the Swift interfaces",
    priority: .high,
    tags: ["documentation", "development"]
)

let result = await commandHandler.execute(createTaskCommand)

// Handle result
switch result {
case .success(let task):
    print("Created task: \(task.id)")
case .failure(let error):
    print("Failed: \(error.localizedDescription)")
}
```

### Event Versioning and Migration

```swift
// Register event migrators
let registry = EventMigrationRegistry.shared
registry.register(migrator: TaskCreatedV1_0_0ToV1_1_0Migrator())

// Create event store with migration support
let eventStore = createEventStore(type: .coreData, migrationRegistry: registry)

// Events will be automatically migrated when retrieved
let events = try await eventStore.getEvents()
```

### State Projection

```swift
// Project all events to build application state
let events = try await eventStore.getEvents()
let state = projectEvents(events)

// Get specific entities
let task = state.tasks["task-123"]
let project = state.projects["project-456"]

// Project events for a specific entity
let taskEvents = try await eventStore.getEventsByType(type: TaskEventType.created)
let task = projectTaskEvents(taskId: "task-123", events: taskEvents)
```