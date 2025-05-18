# TCA Integration

This document illustrates how The Composable Architecture (TCA) integrates with the event sourcing system in the MonoSphere application.

## TCA Component Structure

```mermaid
classDiagram
    class Store~State, Action~ {
        +state State
        +send(Action)
    }
    
    class Reducer~State, Action~ {
        <<protocol>>
        +reduce(state: inout State, action: Action) -> Effect~Action, Never~
    }
    
    class State {
        +projects [Project]
        +tasks [Task]
        +selectedProject Project?
        +selectedTask Task?
        +isLoading bool
        +error Error?
    }
    
    class Action {
        <<enumeration>>
        loadProjects
        projectsLoaded([Project])
        loadTasks
        tasksLoaded([Task])
        createTask(Task)
        taskCreated(Task)
        updateTask(Task)
        taskUpdated(Task)
        deleteTask(String)
        taskDeleted(String)
        selectProject(String)
        errorOccurred(Error)
        ...
    }
    
    class Effect~Output, Failure~ {
        +map(transform) Effect~NewOutput, Failure~
        +flatMap(transform) Effect~NewOutput, Failure~
        +receive(on: scheduler) Effect~Output, Failure~
        +eraseToEffect() Effect~Output, Failure~
    }
    
    Store o-- State : contains
    Store --> Reducer : uses
    Reducer --> Effect : produces
    Reducer --> Action : processes
    Reducer --> State : updates
```

## TCA and Event Sourcing Integration

```mermaid
graph TD
    subgraph TCAComponents["TCA Components"]
        A["TCA Action"]
        R["TCA Reducer"]
        E["TCA Effect"]
        S["TCA State"]
    end
    
    subgraph EventSourcing["Event Sourcing System"]
        CH["Command Handler"]
        C["Command"]
        EV["Event"]
        ES["Event Store"]
        P["Projections"]
        SC["State Container"]
    end
    
    A --> R
    R --> E
    E --> CH
    CH --> C
    C --> EV
    EV --> ES
    ES --> P
    P --> SC
    SC --> S
    S ---> A
    
    classDef tcaNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef commandNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef eventNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    
    class A,R,E,S tcaNode;
    class CH,C commandNode;
    class EV,ES,P,SC eventNode;
```

## TCA Action to Command Flow

```mermaid
sequenceDiagram
    participant View as SwiftUI View
    participant Store as TCA Store
    participant Reducer as Reducer
    participant Effect as Effect
    participant Client as Command Client
    participant Handler as Command Handler
    participant EventStore
    
    View->>Store: send(.createTask)
    activate Store
    Store->>Reducer: reduce(.createTask)
    activate Reducer
    Reducer->>Reducer: Map action to effect
    Reducer-->>Store: createTaskEffect
    deactivate Reducer
    deactivate Store
    
    Store->>Effect: run effect
    activate Effect
    Effect->>Client: createTask(...)
    activate Client
    
    Client->>Handler: execute(CreateTaskCommand)
    activate Handler
    Handler->>Handler: validate command
    Handler->>Handler: generate event
    Handler->>EventStore: store event
    activate EventStore
    EventStore-->>Handler: success
    deactivate EventStore
    Handler-->>Client: CommandResult.success(task)
    deactivate Handler
    
    Client-->>Effect: Return task result
    deactivate Client
    Effect-->>Store: send(.taskCreated(task))
    deactivate Effect
    
    Store->>Reducer: reduce(.taskCreated)
    activate Reducer
    Reducer->>Reducer: Update state with new task
    Reducer-->>Store: state updated
    deactivate Reducer
    
    Store->>View: state changes
```

## Projection to TCA State Flow

```mermaid
sequenceDiagram
    participant ES as Event Store
    participant P as Projections
    participant SC as State Container
    participant Client as State Client
    participant Store as TCA Store
    participant View as SwiftUI View
    
    Note over ES,View: Projection refresh flow (on app start or data change)
    
    Client->>ES: getEvents()
    activate ES
    ES-->>Client: events
    deactivate ES
    
    Client->>P: projectEvents(events)
    activate P
    P->>P: Apply events
    P-->>Client: stateContainer
    deactivate P
    
    Client->>Store: send(.stateLoaded(stateContainer))
    activate Store
    Store->>Store: Reduce state
    Store->>View: Updated state
    deactivate Store
```

## TCA Component Structure

```mermaid
graph TD
    subgraph Feature["Feature"]
        State["State"]
        Action["Action"]
        Environment["Environment"]
        Reducer["Reducer"]
    end
    
    subgraph Domain["Domain Integration"]
        CommandMap["Action -> Command"]
        CH["Command Handler"]
        CM["Command -> Effect"]
        Proj["Projection -> State"]
    end
    
    Action --> CommandMap
    CommandMap --> CH
    CH --> CM
    CM --> Reducer
    
    Proj --> State
    Environment --> CH
    Reducer -- "updates" --> State
    
    classDef tcaNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef integrationNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    
    class State,Action,Environment,Reducer tcaNode;
    class CommandMap,CH,CM,Proj integrationNode;
```

## TCA Environment Integration

```mermaid
classDiagram
    class Environment {
        +commandHandler CommandHandler
        +mainQueue AnySchedulerOf~DispatchQueue~
        +uuid () -> String
        +date () -> Date
    }
    
    class AppEnvironment {
        +eventStore EventStore
        +commandHandler CommandHandler
        +stateContainer StateContainer
        +mainQueue AnySchedulerOf~DispatchQueue~
        +uuid () -> String
        +date () -> Date
    }
    
    class ProjectsEnvironment {
        +commandHandler CommandHandler
        +mainQueue AnySchedulerOf~DispatchQueue~
        +uuid () -> String
    }
    
    class TasksEnvironment {
        +commandHandler CommandHandler
        +mainQueue AnySchedulerOf~DispatchQueue~
        +uuid () -> String
    }
    
    AppEnvironment --|> Environment : provides base
    AppEnvironment --o ProjectsEnvironment : creates
    AppEnvironment --o TasksEnvironment : creates
```

## Example TCA Reducer

```mermaid
flowchart TD
    subgraph Reducer["TaskReducer"]
        direction TB
        
        TaskAction["TaskAction Enumeration"] --> ReduceFunction["reduce(state:action:)"]
        
        subgraph ActionHandlers["Action Handlers"]
            CreateTask["Handle: createTask"]
            UpdateTask["Handle: updateTask"]
            DeleteTask["Handle: deleteTask"]
            CompleteTask["Handle: completeTask"]
        end
        
        ReduceFunction --> ActionHandlers
        
        CreateTask --> CommandEffect1["Effect: Execute CreateTaskCommand"]
        UpdateTask --> CommandEffect2["Effect: Execute UpdateTaskCommand"]
        DeleteTask --> CommandEffect3["Effect: Execute DeleteTaskCommand"]
        CompleteTask --> CommandEffect4["Effect: Execute CompleteTaskCommand"]
        
        CommandEffect1 --> ResultAction1["Action: taskCreated/taskCreateFailed"]
        CommandEffect2 --> ResultAction2["Action: taskUpdated/taskUpdateFailed"]
        CommandEffect3 --> ResultAction3["Action: taskDeleted/taskDeleteFailed"]
        CommandEffect4 --> ResultAction4["Action: taskCompleted/taskCompleteFailed"]
    end
    
    classDef actionNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef reducerNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef effectNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    classDef resultNode fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#f57f17;
    
    class TaskAction,ResultAction1,ResultAction2,ResultAction3,ResultAction4 actionNode;
    class Reducer,ReduceFunction,ActionHandlers,CreateTask,UpdateTask,DeleteTask,CompleteTask reducerNode;
    class CommandEffect1,CommandEffect2,CommandEffect3,CommandEffect4 effectNode;
```

## TCA-Event Sourcing Benefits

The integration of TCA with event sourcing in MonoSphere offers several benefits:

1. **Predictable State Updates** - TCA reducers ensure consistent state transitions
2. **Isolated Side Effects** - Side effects are isolated in TCA effects
3. **Composable Features** - TCA enables feature composition and modularity
4. **Testable Actions** - Each action and effect can be tested in isolation
5. **Unidirectional Data Flow** - Clear data flow from user actions to state updates
6. **Single Source of Truth** - Event store is the single source of truth for application state

The integration pattern ensures that:

- TCA Actions map to Commands in the event sourcing system
- Command results trigger TCA Actions that update the UI state
- Event store projections feed into TCA State
- TCA Environment provides access to the CommandHandler and EventStore