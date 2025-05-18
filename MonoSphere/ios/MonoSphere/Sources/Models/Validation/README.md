# Command Validation System

MonoSphere implements a comprehensive validation system to ensure data integrity and enforce business rules before commands are executed. This document explains the validation approach, components, and implementation details.

## Overview

The validation system in MonoSphere serves these key purposes:

1. **Data Integrity**: Ensure input data meets basic structural requirements
2. **Business Rules**: Enforce domain-specific rules and constraints
3. **Error Prevention**: Catch invalid operations before they affect the system
4. **User Feedback**: Provide clear, specific error messages for invalid inputs

## Core Components

### Validator

The `Validator` struct provides static validation methods for all domain entities:

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
    
    // Helper validation
    public static func validateUUID(_ id: String) -> Result<String, DomainError>
}
```

### Validation Result

Validation methods return a `Result` type with either a success value or a domain error:

```swift
// Successful validation
Result<String, DomainError>.success(validatedValue)

// Failed validation
Result<String, DomainError>.failure(.taskValidationFailed(reason: "Title cannot be empty"))
```

### Domain Errors

The `DomainError` enum provides comprehensive error types for validation failures:

```swift
public enum DomainError: Error, Equatable {
    // Task-specific validation errors
    case taskValidationFailed(reason: String)
    
    // Project-specific validation errors
    case projectValidationFailed(reason: String)
    
    // General validation errors
    case invalidInput(String)
    
    // Other domain errors...
}
```

## Validation Rules

### Task Validation

Tasks are validated with these rules:

1. **Title**:
   - Cannot be empty
   - Maximum length of 100 characters

2. **Description**:
   - Can be empty
   - Maximum length of 2000 characters

3. **Due Date**:
   - Optional
   - If provided, must not be in the past

4. **Tags**:
   - No empty tags
   - Maximum of 10 tags
   - Each tag maximum length of 30 characters

### Project Validation

Projects are validated with these rules:

1. **Name**:
   - Cannot be empty
   - Maximum length of 100 characters

2. **Description**:
   - Can be empty
   - Maximum length of 2000 characters

3. **Tags**:
   - No empty tags
   - Maximum of 10 tags
   - Each tag maximum length of 30 characters

## Integration with Command Handling

Validation is integrated with the command handling system:

```swift
public struct CreateTaskCommand: Command {
    public typealias ResultType = Task
    
    public let title: String
    public let description: String
    public let priority: TaskPriority
    public let dueDate: Date?
    public let tags: [String]
    
    public func validate() -> Result<Void, DomainError> {
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
        
        // Validate tags
        let tagsResult = Validator.validateTaskTags(tags)
        if case .failure(let error) = tagsResult {
            return .failure(error)
        }
        
        return .success(())
    }
    
    public func execute(state: StateContainer, eventStore: EventStore) async throws -> CommandResult<Task> {
        // Validate inputs first
        let validationResult = validate()
        if case .failure(let error) = validationResult {
            return .failure(error)
        }
        
        // Proceed with command execution...
    }
}
```

## Command Execution Flow

The validation system fits into the command execution flow:

1. **Command Creation**: User initiates an action with input data
2. **Validation**: Input data is validated against rules
3. **State Check**: Current state is checked for additional validations
4. **Execution**: Command logic executes if validation passes
5. **Event Generation**: Events are created from valid commands
6. **Result**: Success with result or failure with error message

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│             │     │             │     │             │     │             │
│  Command    │────►│  Validation │────►│  Execution  │────►│  Events     │
│  Creation   │     │             │     │             │     │             │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                           │                                       │
                           ▼                                       ▼
                    ┌─────────────┐                        ┌─────────────┐
                    │             │                        │             │
                    │  Validation │                        │  Event      │
                    │  Errors     │                        │  Store      │
                    └─────────────┘                        └─────────────┘
```

## Error Handling

Validation errors are handled in these ways:

1. **Early Return**: Commands fail fast when validation fails
2. **Specific Errors**: Errors include specific reasons for failure
3. **Result Type**: `Result` type clearly indicates success or failure
4. **User Feedback**: Validation errors are reported to the user interface

## Validation Examples

### Task Title Validation

```swift
public static func validateTaskTitle(_ title: String) -> Result<String, DomainError> {
    // Title cannot be empty
    if title.isEmpty {
        return .failure(.taskValidationFailed(reason: "Title cannot be empty"))
    }
    
    // Title cannot be too long (arbitrary limit)
    if title.count > 100 {
        return .failure(.taskValidationFailed(reason: "Title cannot exceed 100 characters"))
    }
    
    return .success(title)
}
```

### Due Date Validation

```swift
public static func validateTaskDueDate(_ dueDate: Date?) -> Result<Date?, DomainError> {
    guard let dueDate = dueDate else {
        return .success(nil)
    }
    
    // Due date cannot be in the past
    if dueDate < Date().startOfDay() {
        return .failure(.taskValidationFailed(reason: "Due date cannot be in the past"))
    }
    
    return .success(dueDate)
}
```

## Command Error Handling

The `CommandResult` type handles the result of command execution:

```swift
public enum CommandResult<T> {
    case success(T)
    case failure(DomainError)
    
    // Map result to a new type
    public func map<U>(_ transform: (T) -> U) -> CommandResult<U>
    
    // Flat map result to a new result
    public func flatMap<U>(_ transform: (T) -> CommandResult<U>) -> CommandResult<U>
}
```

## Best Practices

When working with the validation system, follow these practices:

1. **Validate Early**: Perform validation before any business logic
2. **Specific Messages**: Provide clear, actionable error messages
3. **Single Responsibility**: Each validation method checks one aspect
4. **Result Composition**: Combine validation results for complex validations
5. **Pure Functions**: Keep validation methods pure without side effects

## Advanced Validation

MonoSphere supports advanced validation scenarios:

1. **Cross-Field Validation**: Rules that depend on multiple fields
2. **State-Dependent Validation**: Rules that depend on current system state
3. **Conditional Validation**: Rules that apply only in certain contexts
4. **Complex Business Rules**: Multi-step validations for complex constraints

## Design Principles

The validation system follows these principles:

1. **Explicitness**: Validation rules are explicitly defined
2. **Composability**: Validation results can be combined
3. **Immutability**: Validation doesn't modify input data
4. **Type Safety**: Leverage Swift's type system for validation
5. **Clear Feedback**: Validation errors provide clear guidance