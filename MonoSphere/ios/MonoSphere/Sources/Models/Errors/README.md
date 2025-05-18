# Error Handling System

MonoSphere implements a comprehensive error handling system to manage failures and exceptions throughout the application. This document explains the error types, handling patterns, and best practices.

## Overview

The error handling system in MonoSphere serves these key purposes:

1. **Type Safety**: Errors are represented as strongly-typed values
2. **Specificity**: Error types provide detailed information about the failure
3. **Recoverability**: Error handling allows for appropriate recovery strategies
4. **Usability**: Error messages are clear and actionable for users

## Core Components

### Domain Errors

The `DomainError` enum is the cornerstone of the error handling system:

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
}
```

The `DomainError` enum categorizes errors into:

1. **General Errors**: Common errors that apply across the application
2. **Entity-Specific Errors**: Errors related to specific domain entities
3. **System Component Errors**: Errors from specific system components
4. **External System Errors**: Errors from external systems or services

### Error Localization

All domain errors provide localized descriptions for user-facing messages:

```swift
public var localizedDescription: String {
    switch self {
    case .taskNotFound(let id):
        return "Task with ID \(id) not found"
    case .taskValidationFailed(let reason):
        return "Task validation failed: \(reason)"
    case .taskAlreadyCompleted(let id):
        return "Task \(id) is already completed"
    // Other cases...
    }
}
```

### Result Type

MonoSphere uses Swift's `Result` type for functions that can fail:

```swift
// Validation function returning a Result
public static func validateTaskTitle(_ title: String) -> Result<String, DomainError>

// Command execution returning a CommandResult
public func execute<T>(_ command: Command) async -> CommandResult<T>
```

## Error Handling Patterns

### Early Validation

Inputs are validated early to prevent cascading errors:

```swift
func createTask(title: String, description: String) async -> CommandResult<Task> {
    // Validate inputs first
    let titleResult = Validator.validateTaskTitle(title)
    if case .failure(let error) = titleResult {
        return .failure(error)
    }
    
    let descriptionResult = Validator.validateTaskDescription(description)
    if case .failure(let error) = descriptionResult {
        return .failure(error)
    }
    
    // Proceed with operation...
}
```

### Error Propagation

Errors are propagated up the call stack without loss of information:

```swift
func updateTask(id: String, title: String) async -> CommandResult<Task> {
    // Get task, propagating not found error
    guard let task = state.tasks[id] else {
        return .failure(.taskNotFound(id: id))
    }
    
    // Validate title, propagating validation error
    let titleResult = Validator.validateTaskTitle(title)
    if case .failure(let error) = titleResult {
        return .failure(error)
    }
    
    // Proceed with update...
}
```

### Command Results

Command execution returns a `CommandResult` type:

```swift
public enum CommandResult<T> {
    case success(T)
    case failure(DomainError)
}
```

This type provides a clear indication of success or failure, with associated values in both cases.

### Result Mapping

The `CommandResult` type supports mapping for result transformation:

```swift
extension CommandResult {
    public func map<U>(_ transform: (T) -> U) -> CommandResult<U> {
        switch self {
        case .success(let value):
            return .success(transform(value))
        case .failure(let error):
            return .failure(error)
        }
    }
    
    public func flatMap<U>(_ transform: (T) -> CommandResult<U>) -> CommandResult<U> {
        switch self {
        case .success(let value):
            return transform(value)
        case .failure(let error):
            return .failure(error)
        }
    }
}
```

### Async Error Handling

For async operations, errors are handled using Swift's async/await pattern:

```swift
do {
    let events = try await eventStore.getEvents()
    let state = projectEvents(events)
    return state
} catch let error as DomainError {
    // Handle domain error
    return createEmptyState()
} catch {
    // Handle other errors
    return createEmptyState()
}
```

## Error Categories

### Validation Errors

Failures that occur when input data doesn't meet requirements:

```swift
case taskValidationFailed(reason: String)
case projectValidationFailed(reason: String)
case eventValidationFailed(reason: String)
```

### Not Found Errors

Failures that occur when an entity doesn't exist:

```swift
case taskNotFound(id: String)
case projectNotFound(id: String)
case eventNotFound(id: String)
```

### State Conflict Errors

Failures due to entity state conflicts:

```swift
case taskAlreadyCompleted(id: String)
case projectContainsTask(projectId: String, taskId: String)
case eventVersionMismatch(expected: String, actual: String)
```

### System Errors

Failures in system components:

```swift
case eventStoreError(String)
case syncFailed(reason: String)
case networkError(String)
```

## Error Recovery Strategies

MonoSphere implements these error recovery strategies:

1. **User Correction**: Validation errors allow users to correct input
2. **Retry Logic**: Network errors can trigger retries with backoff
3. **Graceful Degradation**: System errors can fall back to alternative functionality
4. **State Reconstruction**: Event errors can trigger state reconstruction from available events

## UI Error Presentation

Errors are presented to users in a clear, actionable way:

1. **Toast Messages**: Brief error messages for non-critical errors
2. **Alert Dialogs**: Detailed error information for critical failures
3. **Field Validation**: Inline validation errors for form inputs
4. **Error Pages**: Dedicated screens for serious system errors

## Logging and Monitoring

Errors are logged for debugging and analysis:

```swift
func logError(_ error: DomainError) {
    Logger.error("Domain error: \(error.localizedDescription)")
    
    // For specific error types, add additional context
    switch error {
    case .taskNotFound(let id):
        Logger.error("Task lookup failed with ID: \(id)")
    case .eventStoreError(let reason):
        Logger.error("Event store operation failed: \(reason)", metadata: ["component": "eventStore"])
    default:
        break
    }
}
```

## Best Practices

When working with the error handling system, follow these practices:

1. **Be Specific**: Use the most specific error type available
2. **Include Context**: Add contextual information to errors (IDs, reasons)
3. **Handle Gracefully**: Provide appropriate recovery options when possible
4. **Log Effectively**: Log errors with sufficient context for debugging
5. **User-Friendly Messages**: Translate technical errors to user-friendly messages

## Examples

### Command Execution with Error Handling

```swift
let createTaskCommand = CreateTaskCommand(
    title: "",  // Invalid - empty title
    description: "Task description",
    priority: .medium,
    tags: []
)

let result = await commandHandler.execute(createTaskCommand)

switch result {
case .success(let task):
    // Handle success
    print("Created task: \(task.title)")
    
case .failure(let error):
    // Handle specific error types
    switch error {
    case .taskValidationFailed(let reason):
        print("Invalid task data: \(reason)")
        // Show validation error to user
        
    case .eventStoreError(let reason):
        print("System error: \(reason)")
        // Show generic error to user
        
    default:
        print("Unexpected error: \(error.localizedDescription)")
        // Show generic error to user
    }
}
```

### Validation Result Composition

```swift
func validateTaskInput(title: String, description: String, dueDate: Date?) -> Result<Void, DomainError> {
    // Validate title
    let titleResult = Validator.validateTaskTitle(title)
    if case .failure(let error) = titleResult {
        return .failure(error)
    }
    
    // Validate description
    let descriptionResult = Validator.validateTaskDescription(description)
    if case .failure(let error) = descriptionResult {
        return .failure(error)
    }
    
    // Validate due date
    let dueDateResult = Validator.validateTaskDueDate(dueDate)
    if case .failure(let error) = dueDateResult {
        return .failure(error)
    }
    
    return .success(())
}
```

## Design Principles

The error handling system follows these principles:

1. **Type Safety**: Errors are strongly typed for compile-time checks
2. **Explicitness**: Error handling is explicit, not hidden
3. **Granularity**: Errors provide specific, detailed information
4. **Context Preservation**: Errors maintain context for diagnosis
5. **User Focus**: Error messages are designed for end-users