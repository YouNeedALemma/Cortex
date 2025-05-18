# Command Processing Flow

This document illustrates the command processing flow in the MonoSphere application, showing how user actions are transformed into commands, validated, and then converted into events that update the application state.

## Basic Command Flow

```mermaid
graph LR
    A[User Action] --> B[Command Creation]
    B --> C[Validation]
    C --> D[Processing]
    D --> E[Event Storage]
    E --> F[Projection]
    F --> G[State Update]
    G --> H[UI Update]
    
    classDef userNode fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#e65100;
    classDef commandNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef eventNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef stateNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    
    class A,H userNode;
    class B,C,D commandNode;
    class E,F eventNode;
    class G stateNode;
```

## Detailed Command Processing Sequence

```mermaid
sequenceDiagram
    participant User
    participant UI as User Interface
    participant CH as Command Handler
    participant V as Validator
    participant EF as Event Factory
    participant ES as Event Store
    participant P as Projections
    participant S as State Container
    
    User->>UI: Initiates Action
    UI->>CH: Creates Command
    
    Note over CH: Command Validation Phase
    CH->>V: Validate Command
    activate V
    V->>V: Check Business Rules
    V->>V: Verify Required Fields
    V->>V: Validate References
    V-->>CH: Validation Result
    deactivate V
    
    alt Validation Failed
        CH-->>UI: Validation Error
        UI-->>User: Show Error Message
    else Validation Passed
        Note over CH: Command Execution Phase
        CH->>EF: Create Event(s)
        activate EF
        EF->>EF: Set Event Metadata
        EF->>EF: Build Event Payload
        EF-->>CH: New Event(s)
        deactivate EF
        
        CH->>ES: Store Event(s)
        activate ES
        ES->>ES: Apply Migration if Needed
        ES->>ES: Persist Event
        ES-->>CH: Storage Confirmation
        deactivate ES
        
        Note over CH: State Update Phase
        CH->>P: Rebuild State
        activate P
        P->>ES: Get Events
        ES-->>P: Event Stream
        P->>P: Apply Events to State
        P-->>S: Updated State
        deactivate P
        
        CH-->>UI: Command Success
        UI->>S: Get Updated State
        S-->>UI: Current State
        UI-->>User: Show Updated View
    end
```

## Command Structure

```mermaid
classDiagram
    class Command {
        <<interface>>
        +validate() -> [ValidationError]?
    }
    
    class CreateTaskCommand {
        +String title
        +String description
        +TaskPriority priority
        +Date? dueDate
        +String[] tags
        +String? projectId
        +validate() -> [ValidationError]?
    }
    
    class UpdateTaskCommand {
        +String id
        +String? title
        +String? description
        +TaskPriority? priority
        +Date? dueDate
        +String[]? tags
        +validate() -> [ValidationError]?
    }
    
    class CreateProjectCommand {
        +String name
        +String description
        +ProjectStatus status
        +String[] tags
        +validate() -> [ValidationError]?
    }
    
    class UpdateProjectCommand {
        +String id
        +String? name
        +String? description
        +ProjectStatus? status
        +String[]? tags
        +validate() -> [ValidationError]?
    }
    
    Command <|-- CreateTaskCommand
    Command <|-- UpdateTaskCommand
    Command <|-- CreateProjectCommand
    Command <|-- UpdateProjectCommand
```

## Command Handler Implementation

```mermaid
classDiagram
    class CommandHandler {
        -EventStore eventStore
        -StateContainer state
        +CommandHandler(eventStore, state)
        +execute~T~(Command) async -> CommandResult~T~
        -validateCommand(Command) -> [ValidationError]?
        -processCommand~T~(Command) async -> CommandResult~T~
        -storeEvents([Event]) async throws
    }
    
    class CommandResult~T~ {
        <<enumeration>>
        success(T)
        failure(DomainError)
    }
    
    class DomainError {
        <<enumeration>>
        taskNotFound(id: String)
        projectNotFound(id: String)
        validationFailed(reason: String)
        eventStoreError(reason: String)
        ...
    }
    
    CommandHandler -- CommandResult~T~ : returns
    CommandResult~T~ -- DomainError : contains on failure
```

## Validation Process

```mermaid
flowchart TD
    subgraph CommandHandlerProcess["Command Handler Process"]
        A[Receive Command] --> B{Validate Command}
        B -->|Valid| C[Process Command]
        B -->|Invalid| D[Return Validation Error]
        C --> E[Generate Events]
        E --> F[Store Events]
        F --> G[Update State]
        G --> H[Return Success Result]
    end
    
    subgraph ValidationProcess["Validation Process"]
        V1[Check Required Fields] --> V2[Validate Field Formats]
        V2 --> V3[Check Entity References]
        V3 --> V4[Verify Business Rules]
        V4 --> V5[Check Entity State]
    end
    
    B --- ValidationProcess
    
    classDef processNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef validationNode fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#f57f17;
    
    class A,B,C,D,E,F,G,H processNode;
    class ValidationProcess,V1,V2,V3,V4,V5 validationNode;
```

## Example Command Flow: Create Task

```mermaid
sequenceDiagram
    participant App
    participant CH as CommandHandler
    participant V as Validator
    participant EF as TaskEvents
    participant ES as EventStore
    participant P as Projections
    
    App->>CH: execute(CreateTaskCommand)
    activate CH
    
    CH->>V: validate(command)
    activate V
    V-->>CH: validation ok
    deactivate V
    
    CH->>EF: createTaskCreatedEvent(...)
    activate EF
    EF-->>CH: TaskCreatedEvent
    deactivate EF
    
    CH->>ES: append(event)
    activate ES
    ES-->>CH: success
    deactivate ES
    
    CH->>P: projectEvents([event], state)
    activate P
    P-->>CH: updatedState
    deactivate P
    
    CH-->>App: CommandResult.success(task)
    deactivate CH
```

## Command Benefits in MonoSphere

The command processing flow in MonoSphere provides several benefits:

1. **Validation** - Commands are validated before execution to ensure data integrity
2. **Single Responsibility** - Each command handles one specific action
3. **Auditing** - Commands generate events that form an audit trail
4. **Testability** - Command handling logic can be tested in isolation
5. **Error Handling** - Structured error types for different failure scenarios
6. **Immutability** - Commands and events are immutable, preventing side effects