# Event Sourcing Architecture

This diagram illustrates the Event Sourcing architecture used in the MonoSphere application. Event Sourcing is a pattern that captures all changes to an application state as a sequence of events.

## Core Event Sourcing Flow

```mermaid
flowchart TD
    subgraph UserAction["User Action"]
        Command["Command"]
    end

    subgraph CommandProcessing["Command Processing"]
        CommandHandler["CommandHandler"]
        Validation["Validation"]
    end

    subgraph EventCreation["Event Creation"]
        EventFactory["Event Factory"]
        EventMetadata["Event Metadata"]
    end

    subgraph EventStorage["Event Storage"]
        EventStore["Event Store"]
        EventMigration["Event Migration Registry"]
        EventVersioning["Event Versioning"]
    end

    subgraph StateReconstruction["State Reconstruction"]
        Projections["Projections"]
        StateContainer["State Container"]
    end

    subgraph UserInterface["User Interface"]
        UIUpdate["UI Update"]
    end

    Command --> CommandHandler
    CommandHandler --> Validation
    Validation --> CommandHandler
    CommandHandler --> EventFactory
    EventFactory --> EventMetadata
    EventMetadata --> EventFactory
    EventFactory --> EventStore
    EventStore --> EventMigration
    EventStore --> EventVersioning
    EventStore --> Projections
    Projections --> StateContainer
    StateContainer --> UIUpdate

    classDef commandNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef eventNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef stateNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    classDef uiNode fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#e65100;

    class Command,CommandHandler,Validation commandNode;
    class EventFactory,EventMetadata,EventStore,EventMigration,EventVersioning eventNode;
    class Projections,StateContainer stateNode;
    class UIUpdate uiNode;
```

## Event Structure

```mermaid
classDiagram
    class Event {
        +String id
        +String type
        +Date timestamp
        +EventMetadata metadata
        +Map~String, AnyCodable~ payload
    }

    class EventMetadata {
        +String? userId
        +String? deviceId
        +String version
        +String? correlationId
        +String? causationId
    }

    Event --> EventMetadata : contains
```

## Event Types

```mermaid
classDiagram
    class Event {
        +String id
        +String type
        +Date timestamp
        +EventMetadata metadata
        +Map~String, AnyCodable~ payload
    }

    class TaskCreated {
        +String id
        +String title
        +String description
        +TaskPriority priority
        +Date? dueDate
        +String[] tags
    }

    class TaskUpdated {
        +String id
        +String? title
        +String? description
        +TaskPriority? priority
        +Date? dueDate
        +String[]? tags
    }

    class ProjectCreated {
        +String id
        +String name
        +String description
        +ProjectStatus status
        +String[] tags
    }

    class ProjectUpdated {
        +String id
        +String? name
        +String? description
        +ProjectStatus? status
        +String[]? tags
    }

    Event <|-- TaskCreated : type=task.created
    Event <|-- TaskUpdated : type=task.updated
    Event <|-- ProjectCreated : type=project.created
    Event <|-- ProjectUpdated : type=project.updated
```

## Event Flow Through System

```mermaid
sequenceDiagram
    participant User
    participant Command as Command Handler
    participant EventFactory as Event Factory
    participant EventStore as Event Store
    participant Projections
    participant State as State Container
    participant UI as User Interface

    User->>Command: Create Command
    activate Command
    Command->>Command: Validate Command
    Command->>EventFactory: Generate Event
    activate EventFactory
    EventFactory-->>Command: Return Event
    deactivate EventFactory
    Command->>EventStore: Store Event
    activate EventStore
    EventStore-->>Command: Confirm Storage
    deactivate EventStore
    deactivate Command
    
    EventStore->>Projections: Retrieve Events
    activate Projections
    Projections->>Projections: Apply Events
    Projections->>State: Update State
    activate State
    State-->>Projections: Return New State
    deactivate State
    deactivate Projections
    
    State->>UI: Update View
    activate UI
    UI-->>User: Display Changes
    deactivate UI
```

## Key Benefits of Event Sourcing in MonoSphere

1. **Complete Audit Trail** - Every change in the system is recorded as an event
2. **Temporal Queries** - State can be reconstructed for any point in time
3. **Immutability** - Events are immutable, simplifying concurrency management
4. **Business Focus** - Events represent meaningful business changes
5. **Extensibility** - New projections can be added without changing the event store
6. **Debuggability** - System behavior can be reproduced by replaying events