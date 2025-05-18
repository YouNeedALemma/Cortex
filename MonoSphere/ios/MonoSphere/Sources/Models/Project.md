# Project Domain Model

The `Project` domain model represents a collection of related tasks within MonoSphere. Projects provide organizational structure and context for tasks, helping users group related work items together.

## Model Definition

```swift
public struct Project: Equatable, Identifiable, Codable {
    /// Unique identifier for the project
    public let id: String
    
    /// Name of the project
    public let name: String
    
    /// Detailed description of the project
    public let description: String
    
    /// Current status of the project
    public let status: ProjectStatus
    
    /// List of tags associated with the project for categorization
    public let tags: [String]
    
    /// Timestamp when the project was created
    public let createdAt: Date
    
    /// Timestamp when the project was last updated
    public let updatedAt: Date
    
    /// Timestamp when the project was completed (if applicable)
    public let completedAt: Date?
    
    /// List of task IDs associated with this project
    public let taskIds: [String]
}
```

## Project Status

The `ProjectStatus` enum defines the possible states for a project:

```swift
public enum ProjectStatus: String, Equatable, Codable {
    /// Project is currently being worked on
    case active
    
    /// Project is temporarily paused
    case onHold
    
    /// Project has been successfully finished
    case completed
    
    /// Project has been terminated before completion
    case cancelled
}
```

## Lifecycle

Projects follow this general lifecycle:

1. **Creation**: Projects are created with a unique ID, name, description, and other metadata
2. **Updates**: Project details can be modified (creating new immutable instances)
3. **Task Management**: Tasks can be added to or removed from the project
4. **Status Changes**: Project status can change (active, on hold, completed, cancelled)
5. **Completion**: Projects can be marked as completed, which sets a completion timestamp
6. **Deletion**: Projects can be deleted from the system

## Business Rules

Projects must adhere to these business rules:

1. Each project must have a unique identifier
2. Project name must not be empty and must be less than 255 characters
3. A project can have multiple tasks associated with it
4. A project can be in one of four status states (active, onHold, completed, cancelled)
5. A completed project must have a completedAt timestamp
6. A project can have multiple tags for categorization
7. When a project is deleted, relationships to tasks are maintained (tasks aren't deleted)

## Event Types

Projects generate the following event types:

- `ProjectCreated`: When a new project is added to the system
- `ProjectUpdated`: When project details are modified
- `ProjectStatusChanged`: When a project's status changes
- `ProjectCompleted`: When a project is marked as complete
- `ProjectTaskAdded`: When a task is added to a project
- `ProjectTaskRemoved`: When a task is removed from a project
- `ProjectDeleted`: When a project is removed from the system

## Commands

Users can interact with projects through these commands:

- `CreateProjectCommand`: Create a new project
- `UpdateProjectCommand`: Modify an existing project's details
- `ChangeProjectStatusCommand`: Change a project's status
- `CompleteProjectCommand`: Mark a project as completed
- `AddTaskToProjectCommand`: Associate a task with the project
- `RemoveTaskFromProjectCommand`: Disassociate a task from the project
- `DeleteProjectCommand`: Remove a project from the system

## Projections

Project state is derived from events through projections:

```swift
func projectProjectEvents(_ events: [Event], initialProject: Project? = nil) -> Project? {
    // Apply each event to build the current project state
}
```

## Example Usage

```swift
// Creating a project
let createProjectCommand = CreateProjectCommand(
    name: "Documentation Project",
    description: "Create comprehensive documentation for the application",
    tags: ["documentation", "knowledge-base"]
)

let result = await commandHandler.execute(createProjectCommand)

// Adding a task to a project
let addTaskCommand = AddTaskToProjectCommand(projectId: projectId, taskId: taskId)
let addResult = await commandHandler.execute(addTaskCommand)

// Completing a project
let completeProjectCommand = CompleteProjectCommand(projectId: projectId)
let completionResult = await commandHandler.execute(completeProjectCommand)
```

## Validation

Projects are validated using the following checks:

1. Name must not be empty and must be less than 255 characters
2. Description must be less than 10,000 characters
3. Tags must each be less than 50 characters
4. Project status must be a valid value from the ProjectStatus enum

## Relationships

Projects can have these relationships:

- A project can contain many `Task` instances
- A task can belong to at most one project at a time
- When a project is completed, its tasks are not automatically completed
- When a project is deleted, its tasks remain in the system (relationship is removed)