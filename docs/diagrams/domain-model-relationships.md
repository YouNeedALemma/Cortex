# Domain Model Relationships

This document illustrates the relationships between the core domain models in the MonoSphere application, showing their structure, relationships, and key behaviors.

## Core Domain Models

```mermaid
classDiagram
    class Project {
        +String id
        +String name
        +String description
        +ProjectStatus status
        +String[] tags
        +Date createdAt
        +Date updatedAt
        +Date? completedAt
        +String[] taskIds
        +static create(...) Project
        +update(...) Project
        +complete() Project
        +changeStatus(status) Project
        +addTask(taskId) Project
        +removeTask(taskId) Project
    }
    
    class Task {
        +String id
        +String title
        +String description
        +TaskPriority priority
        +Date? dueDate
        +bool isCompleted
        +Date? completedAt
        +Date createdAt
        +Date updatedAt
        +String? projectId
        +String[] tags
        +static create(...) Task
        +update(...) Task
        +toggle() Task
        +complete() Task
        +uncomplete() Task
        +changePriority(priority) Task
        +assignToProject(projectId) Task
        +removeFromProject() Task
    }
    
    class ProjectStatus {
        <<enumeration>>
        active
        onHold
        completed
        cancelled
    }
    
    class TaskPriority {
        <<enumeration>>
        low
        medium
        high
        urgent
    }
    
    class StateContainer {
        +Map~String, Project~ projects
        +Map~String, Task~ tasks
    }
    
    Project "1" o-- "*" Task : contains >
    Project -- ProjectStatus : has status
    Task -- TaskPriority : has priority
    Task "0..1" -- "0..1" Project : belongs to >
    StateContainer "1" *-- "*" Project : contains
    StateContainer "1" *-- "*" Task : contains
```

## Domain Model Entity Relationships

```mermaid
erDiagram
    PROJECT ||--o{ TASK : "contains"
    PROJECT {
        string id PK
        string name
        string description
        enum status
        string[] tags
        datetime createdAt
        datetime updatedAt
        datetime completedAt
        string[] taskIds
    }
    
    TASK {
        string id PK
        string title
        string description
        enum priority
        datetime dueDate
        boolean isCompleted
        datetime completedAt
        datetime createdAt
        datetime updatedAt
        string projectId FK
        string[] tags
    }
```

## Model Creation and Mutation Flows

```mermaid
graph TD
    subgraph ProjectLifecycle["Project Lifecycle"]
        PC["Project.create(...)"] --> P["Project"]
        P --> PU["project.update(...)"]
        PU --> P1["Updated Project"]
        P1 --> PS["project.changeStatus(...)"]
        PS --> P2["Project with new status"]
        P2 --> PCT["project.complete()"]
        PCT --> P3["Completed Project"]
    end
    
    subgraph TaskLifecycle["Task Lifecycle"]
        TC["Task.create(...)"] --> T["Task"]
        T --> TU["task.update(...)"]
        TU --> T1["Updated Task"]
        T1 --> TP["task.changePriority(...)"]
        TP --> T2["Task with new priority"]
        T2 --> TCT["task.complete()"]
        TCT --> T3["Completed Task"]
    end
    
    subgraph TaskProjectAssociation["Task-Project Association"]
        T --> TA["task.assignToProject(projectId)"]
        TA --> TA1["Task assigned to project"]
        P --> PA["project.addTask(taskId)"]
        PA --> PA1["Project with task added"]
        
        TA1 --> TR["task.removeFromProject()"]
        TR --> TR1["Task removed from project"]
        PA1 --> PR["project.removeTask(taskId)"]
        PR --> PR1["Project with task removed"]
    end
    
    classDef projectNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef taskNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef assocNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    
    class PC,P,PU,P1,PS,P2,PCT,P3,PA,PA1,PR,PR1 projectNode;
    class TC,T,TU,T1,TP,T2,TCT,T3,TA,TA1,TR,TR1 taskNode;
```

## Immutable Model Updates

```mermaid
sequenceDiagram
    participant Command as Command Handler
    participant Model as Task/Project
    participant OldState as Old Instance
    participant NewState as New Instance
    
    Command->>OldState: task.update(title: "New Title")
    activate OldState
    OldState->>NewState: Create copy with new title
    OldState-->>Command: Return new instance
    deactivate OldState
    
    Command->>NewState: task.complete()
    activate NewState
    NewState->>Model: Create new instance with isCompleted=true, completedAt=now
    NewState-->>Command: Return completed task
    deactivate NewState
    
    Note over OldState,NewState: Original instance remains unchanged
```

## State Container Composition

```mermaid
graph TD
    SC["StateContainer"] --> SCT["tasks: Map[id, Task]"]
    SC --> SCP["projects: Map[id, Project]"]
    
    SCP --> P1["Project #1"]
    SCP --> P2["Project #2"]
    SCP --> P3["Project #3"]
    
    SCT --> T1["Task #1"]
    SCT --> T2["Task #2"]
    SCT --> T3["Task #3"]
    SCT --> T4["Task #4"]
    SCT --> T5["Task #5"]
    
    P1 -.-> T1
    P1 -.-> T2
    P2 -.-> T3
    P3 -.-> T4
    P3 -.-> T5
    
    classDef stateNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    classDef projectNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef taskNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    
    class SC,SCT,SCP stateNode;
    class P1,P2,P3 projectNode;
    class T1,T2,T3,T4,T5 taskNode;
```

## Domain Event Types

```mermaid
graph TD
    E[Event] --> TE[Task Events]
    E --> PE[Project Events]
    
    TE --> TC[TaskCreated]
    TE --> TU[TaskUpdated]
    TE --> TD[TaskDeleted]
    TE --> TCP[TaskCompleted]
    TE --> TUC[TaskUncompleted]
    TE --> TPR[TaskPriorityChanged]
    TE --> TAP[TaskAssignedToProject]
    TE --> TRP[TaskRemovedFromProject]
    
    PE --> PC[ProjectCreated]
    PE --> PU[ProjectUpdated]
    PE --> PD[ProjectDeleted]
    PE --> PCP[ProjectCompleted]
    PE --> PSC[ProjectStatusChanged]
    PE --> PTA[ProjectTaskAdded]
    PE --> PTR[ProjectTaskRemoved]
    
    classDef eventNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef taskEventNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef projectEventNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    
    class E,TE,PE eventNode;
    class TC,TU,TD,TCP,TUC,TPR,TAP,TRP taskEventNode;
    class PC,PU,PD,PCP,PSC,PTA,PTR projectEventNode;
```

## Domain Model Design Principles

MonoSphere's domain models follow these key design principles:

1. **Immutability** - All models are immutable value types
2. **Functional Updates** - Models are updated through pure functions that return new instances
3. **Identity** - Each model has a unique identifier
4. **Rich Domain Model** - Models encapsulate behavior, not just data
5. **Consistent Time Tracking** - All models track creation, update, and completion times
6. **Explicit Relationships** - Relationships between models are explicit and managed
7. **Type Safety** - Enums are used for constrained values like status and priority