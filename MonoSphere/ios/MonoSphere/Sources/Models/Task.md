# Task Domain Model

The `Task` domain model represents an individual work item within MonoSphere. Tasks are the foundational unit of work and can exist independently or be associated with projects.

## Model Definition

```swift
public struct Task: Equatable, Identifiable, Codable {
    /// Unique identifier for the task
    public let id: String
    
    /// Title or name of the task
    public let title: String
    
    /// Detailed description of the task
    public let description: String
    
    /// Priority level of the task
    public let priority: TaskPriority
    
    /// Optional deadline for the task
    public let dueDate: Date?
    
    /// Flag indicating if the task is completed
    public let isCompleted: Bool
    
    /// Timestamp when the task was completed (if applicable)
    public let completedAt: Date?
    
    /// Timestamp when the task was created
    public let createdAt: Date
    
    /// Timestamp when the task was last updated
    public let updatedAt: Date
    
    /// ID of the project this task belongs to (if any)
    public let projectId: String?
    
    /// List of tags associated with the task for categorization
    public let tags: [String]
}
```

## Task Priority

The `TaskPriority` enum defines the importance levels for tasks:

```swift
public enum TaskPriority: String, Equatable, Codable, CaseIterable {
    /// Low priority tasks can be deferred
    case low
    
    /// Medium priority tasks should be done in a reasonable timeframe
    case medium
    
    /// High priority tasks should be prioritized
    case high
    
    /// Urgent tasks require immediate attention
    case urgent
}
```

## Lifecycle

Tasks follow this general lifecycle:

1. **Creation**: Tasks are created with a unique ID, title, description, and other metadata
2. **Updates**: Task details can be modified (creating new immutable instances)
3. **Status Changes**: Tasks can be marked as completed or uncompleted
4. **Priority Changes**: Task priority can be adjusted based on changing requirements
5. **Project Assignment**: Tasks can be assigned to or removed from projects
6. **Deletion**: Tasks can be deleted from the system

## Business Rules

Tasks must adhere to these business rules:

1. Each task must have a unique identifier
2. Task title must not be empty and must be less than 255 characters
3. Tasks can optionally have a due date in the future
4. Tasks can be in various priority states (low, medium, high, urgent)
5. Tasks can be associated with at most one project at a time
6. Tasks can have multiple tags for categorization
7. A completed task must have a completedAt timestamp

## Event Types

Tasks generate the following event types:

- `TaskCreated`: When a new task is added to the system
- `TaskUpdated`: When task details are modified
- `TaskCompleted`: When a task is marked as complete
- `TaskUncompleted`: When a task is marked as incomplete
- `TaskPriorityChanged`: When a task's priority level is changed
- `TaskAssignedToProject`: When a task is assigned to a project
- `TaskRemovedFromProject`: When a task is removed from a project
- `TaskDeleted`: When a task is removed from the system

## Commands

Users can interact with tasks through these commands:

- `CreateTaskCommand`: Create a new task
- `UpdateTaskCommand`: Modify an existing task's details
- `CompleteTaskCommand`: Mark a task as completed
- `UncompleteTaskCommand`: Mark a task as not completed
- `ChangeTaskPriorityCommand`: Change a task's priority
- `AssignTaskToProjectCommand`: Associate a task with a project
- `RemoveTaskFromProjectCommand`: Disassociate a task from a project
- `DeleteTaskCommand`: Remove a task from the system

## Projections

Task state is derived from events through projections:

```swift
func projectTaskEvents(_ events: [Event], initialTask: Task? = nil) -> Task? {
    // Apply each event to build the current task state
}
```

## Example Usage

```swift
// Creating a task
let createTaskCommand = CreateTaskCommand(
    title: "Write documentation",
    description: "Create comprehensive documentation for the Task model",
    priority: .high,
    dueDate: Calendar.current.date(byAdding: .day, value: 1, to: Date()),
    tags: ["documentation", "development"]
)

let result = await commandHandler.execute(createTaskCommand)

// Completing a task
let completeTaskCommand = CompleteTaskCommand(taskId: taskId)
let completionResult = await commandHandler.execute(completeTaskCommand)
```

## Validation

Tasks are validated using the following checks:

1. Title must not be empty and must be less than 255 characters
2. Description must be less than 10,000 characters
3. Due date, if provided, must be in the future
4. Tags must each be less than 50 characters

## Relationships

Tasks can have these relationships:

- A task can belong to at most one `Project` at a time
- A task can have many tags for categorization