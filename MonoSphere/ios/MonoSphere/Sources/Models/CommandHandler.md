# Command Handler

The `CommandHandler` is a central component in MonoSphere's domain logic that processes user commands, applies business rules, and produces events. It follows the Command pattern and bridges the gap between user actions and the event sourcing system.

## Overview

The `CommandHandler` is responsible for:

1. Validating commands against business rules
2. Executing domain logic to process commands
3. Generating appropriate events
4. Persisting events to the event store
5. Reporting success or failure results

## Structure

```swift
public class CommandHandler {
    /// Event store for persisting events
    private let eventStore: EventStore
    
    /// Current application state
    private let state: StateContainer
    
    /// Validation service for checking command inputs
    private let validator: Validator
    
    /// Initialize a command handler
    public init(eventStore: EventStore, state: StateContainer)
    
    /// Execute a command and return the result
    public func execute<T>(_ command: Command) async -> CommandResult<T>
}
```

## Command Protocol

Commands are modeled as types conforming to the `Command` protocol:

```swift
public protocol Command {
    /// Type of the result expected when the command succeeds
    associatedtype ResultType
    
    /// Execute the command logic
    func execute(state: StateContainer, eventStore: EventStore) async throws -> CommandResult<ResultType>
    
    /// Validate the command inputs
    func validate(state: StateContainer, validator: Validator) -> ValidationResult
}
```

## Command Result

Command execution produces a `CommandResult` that encapsulates success or failure:

```swift
public enum CommandResult<T> {
    /// Command succeeded with a result value
    case success(T)
    
    /// Command failed with an error
    case failure(DomainError)
}
```

## Execution Flow

The command handler follows this execution flow:

1. **Validation**: Verify the command inputs against business rules
2. **State Check**: Ensure the current state allows the command
3. **Command Execution**: Process the command logic
4. **Event Generation**: Create events representing the state change
5. **Event Persistence**: Store events in the event store
6. **Result Return**: Return success or failure result

## Command Types

MonoSphere implements various command types:

### Task Commands

- `CreateTaskCommand`: Create a new task
- `UpdateTaskCommand`: Update an existing task
- `CompleteTaskCommand`: Mark a task as completed
- `UncompleteTaskCommand`: Mark a task as not completed
- `DeleteTaskCommand`: Remove a task from the system
- `ChangeTaskPriorityCommand`: Change a task's priority level

### Project Commands

- `CreateProjectCommand`: Create a new project
- `UpdateProjectCommand`: Update an existing project
- `CompleteProjectCommand`: Mark a project as completed
- `ChangeProjectStatusCommand`: Change a project's status
- `DeleteProjectCommand`: Remove a project from the system

### Relationship Commands

- `AssignTaskToProjectCommand`: Add a task to a project
- `RemoveTaskFromProjectCommand`: Remove a task from a project

## Error Handling

The command handler uses `DomainError` for comprehensive error reporting:

```swift
public enum DomainError: Error, Equatable {
    // Task errors
    case taskNotFound(id: String)
    case invalidTaskTitle(reason: String)
    case taskAlreadyCompleted(id: String)
    
    // Project errors
    case projectNotFound(id: String)
    case invalidProjectName(reason: String)
    case projectAlreadyCompleted(id: String)
    
    // Validation errors
    case validationFailed(reason: String)
    
    // Event errors
    case eventStoreError(reason: String)
}
```

## Validation

Commands are validated using the `Validator` service:

```swift
public class Validator {
    /// Validate a string field
    public func validateString(_ value: String, fieldName: String, minLength: Int, maxLength: Int) -> ValidationResult
    
    /// Validate a date field
    public func validateDate(_ value: Date?, fieldName: String, mustBeInFuture: Bool) -> ValidationResult
    
    /// Validate an array of strings
    public func validateStringArray(_ values: [String], fieldName: String, maxLength: Int) -> ValidationResult
}
```

Validation results are represented as:

```swift
public enum ValidationResult {
    case valid
    case invalid(reason: String)
}
```

## Example Usage

```swift
// Create a command handler
let eventStore = createEventStore(type: .coreData)
let state = await projectEvents(try await eventStore.getEvents())
let commandHandler = CommandHandler(eventStore: eventStore, state: state)

// Create a task command
let createTaskCommand = CreateTaskCommand(
    title: "Implement feature X",
    description: "Add support for feature X in the application",
    priority: .high,
    dueDate: Date().addingTimeInterval(86400), // Tomorrow
    tags: ["development", "feature"]
)

// Execute the command
let result = await commandHandler.execute(createTaskCommand)

// Handle the result
switch result {
case .success(let task):
    print("Task created with ID: \(task.id)")
case .failure(let error):
    print("Failed to create task: \(error)")
}
```

## Command Implementation Example

Here's an example of how commands are implemented:

```swift
public struct CreateTaskCommand: Command {
    public typealias ResultType = Task
    
    public let title: String
    public let description: String
    public let priority: TaskPriority
    public let dueDate: Date?
    public let tags: [String]
    
    public func validate(state: StateContainer, validator: Validator) -> ValidationResult {
        // Validate title
        let titleValidation = validator.validateString(
            title, 
            fieldName: "title", 
            minLength: 1, 
            maxLength: 255
        )
        
        if case .invalid(let reason) = titleValidation {
            return .invalid(reason: reason)
        }
        
        // Validate description
        let descriptionValidation = validator.validateString(
            description,
            fieldName: "description",
            minLength: 0,
            maxLength: 10000
        )
        
        if case .invalid(let reason) = descriptionValidation {
            return .invalid(reason: reason)
        }
        
        // Validate tags
        let tagsValidation = validator.validateStringArray(
            tags,
            fieldName: "tags",
            maxLength: 50
        )
        
        if case .invalid(let reason) = tagsValidation {
            return .invalid(reason: reason)
        }
        
        // Validate due date
        let dueDateValidation = validator.validateDate(
            dueDate,
            fieldName: "dueDate",
            mustBeInFuture: true
        )
        
        if case .invalid(let reason) = dueDateValidation {
            return .invalid(reason: reason)
        }
        
        return .valid
    }
    
    public func execute(state: StateContainer, eventStore: EventStore) async throws -> CommandResult<Task> {
        // Generate a unique ID
        let taskId = UUID().uuidString
        
        // Create the event
        let event = TaskEvents.createTaskCreatedEvent(
            taskId: taskId,
            title: title,
            description: description,
            priority: priority,
            dueDate: dueDate,
            tags: tags
        )
        
        // Store the event
        try await eventStore.append(event)
        
        // Project the new state
        let events = [event]
        let newState = projectEvents(events, initialState: state)
        
        // Return the new task
        guard let task = newState.tasks[taskId] else {
            return .failure(.eventStoreError(reason: "Failed to retrieve created task"))
        }
        
        return .success(task)
    }
}
```

## Design Principles

The command handler follows these principles:

1. **Separation of Concerns**: Commands contain execution logic, CommandHandler orchestrates
2. **Single Responsibility**: Each command does one thing well
3. **Explicit Validation**: All commands validate inputs before execution
4. **Type Safety**: Generic command results with specific return types
5. **Error Explicitness**: Comprehensive error reporting with DomainError

## Advanced Features

The command handler supports:

1. **Transactional Command Processing**: Commands either fully succeed or fail
2. **Event Correlation**: Events from a command share correlation IDs
3. **Command Composition**: Complex commands can be composed from simpler ones
4. **Context Propagation**: Command context (user, device) flows to events