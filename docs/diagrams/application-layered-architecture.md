# Application Layered Architecture

This document illustrates the layered architecture of the MonoSphere application, showing how the different layers interact and their responsibilities.

## Clean Architecture Layers

```mermaid
graph TD
    subgraph UILayer["UI Layer"]
        UI["User Interface"]
        UIComponents["UI Components"]
        ViewModels["View Models / TCA State"]
        ViewActions["User Actions / TCA Actions"]
    end
    
    subgraph DomainLayer["Domain Layer"]
        DomainLogic["Domain Logic"]
        CommandHandler["Command Handler"]
        Domain["Domain Models"]
        Validation["Validation"]
    end
    
    subgraph EventSourcingLayer["Event Sourcing Layer"]
        Events["Events"]
        Projections["Projections"]
        Migration["Event Migration"]
        StateContainer["State Container"]
    end
    
    subgraph StorageLayer["Storage Layer"]
        EventStore["Event Store"]
        CoreData["CoreData"]
        Serialization["Serialization"]
    end
    
    UI --> UIComponents
    UI --> ViewModels
    UI --> ViewActions
    
    ViewActions --> CommandHandler
    CommandHandler --> DomainLogic
    DomainLogic --> Domain
    DomainLogic --> Validation
    
    CommandHandler --> Events
    Events --> EventStore
    EventStore --> Events
    Events --> Projections
    Projections --> StateContainer
    StateContainer --> ViewModels
    EventStore --> Migration
    
    EventStore --> CoreData
    EventStore --> Serialization
    
    classDef uiNode fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#e65100;
    classDef domainNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef eventNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef storageNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    
    class UI,UIComponents,ViewModels,ViewActions uiNode;
    class DomainLogic,CommandHandler,Domain,Validation domainNode;
    class Events,Projections,Migration,StateContainer eventNode;
    class EventStore,CoreData,Serialization storageNode;
```

## Layered Architecture with Concerns

```mermaid
graph TB
    subgraph UILayer["User Interface Layer"]
        direction LR
        UI["SwiftUI Views"] 
        FC["Feature Components"]
        UIComponents["Reusable UI Components"]
        
        subgraph " "
            A["Presentation Concerns"]
            A1["Layout"]
            A2["Styling"]
            A3["User Interaction"]
            A4["Reactive Updates"]
        end
    end
    
    subgraph DomainLayer["Domain Layer"]
        direction LR
        C["Command Handler"]
        D["Domain Models"]
        V["Validation"]
        E["Error Handling"]
        
        subgraph " "
            B["Domain Concerns"]
            B1["Business Rules"]
            B2["Command Processing"]
            B3["Domain Validation"]
            B4["Invariant Protection"]
        end
    end
    
    subgraph EventLayer["Event Sourcing Layer"]
        direction LR
        ES["Events"]
        P["Projections"]
        EV["Event Versioning"]
        SC["State Container"]
        
        subgraph " "
            C1["Event Sourcing Concerns"]
            C2["Event Creation"]
            C3["State Reconstruction"]
            C4["Schema Evolution"]
            C5["Temporal Queries"]
        end
    end
    
    subgraph StorageLayer["Storage Layer"]
        direction LR
        ESt["Event Store"]
        CD["CoreData Integration"]
        S["Serialization"]
        Q["Query Capabilities"]
        
        subgraph " "
            D1["Storage Concerns"]
            D2["Persistence"]
            D3["Transactions"]
            D4["Data Integrity"]
            D5["Search & Retrieval"]
        end
    end
    
    UILayer --> DomainLayer
    DomainLayer --> EventLayer
    EventLayer --> StorageLayer
    
    classDef uiNode fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#e65100;
    classDef domainNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef eventNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef storageNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    classDef concernNode fill:#f8bbd0,stroke:#880e4f,stroke-width:1px,color:#880e4f;
    
    class UILayer,UI,FC,UIComponents uiNode;
    class DomainLayer,C,D,V,E domainNode;
    class EventLayer,ES,P,EV,SC eventNode;
    class StorageLayer,ESt,CD,S,Q storageNode;
    class A,A1,A2,A3,A4,B,B1,B2,B3,B4,C1,C2,C3,C4,C5,D1,D2,D3,D4,D5 concernNode;
```

## Data Flow Through Layers

```mermaid
sequenceDiagram
    participant User
    
    box "UI Layer" #fff3e0
    participant UI as User Interface
    participant Action as TCA Action
    end
    
    box "Domain Layer" #e1f5fe
    participant Command as Command Handler
    participant Validator as Validation
    end
    
    box "Event Sourcing Layer" #f3e5f5
    participant Event as Event Creation
    participant EventStore as Event Store
    participant Projections
    participant State as State Container
    end
    
    box "Storage Layer" #e8f5e9
    participant CoreData
    end
    
    User->>UI: User Action
    UI->>Action: Dispatch Action
    Action->>Command: Execute Command
    
    Command->>Validator: Validate Command
    Validator-->>Command: Validation Result
    
    alt Validation Failed
        Command-->>Action: Return Error
        Action-->>UI: Update with Error
        UI-->>User: Show Error
    else Validation Passed
        Command->>Event: Create Event
        Event->>EventStore: Store Event
        EventStore->>CoreData: Persist Event
        CoreData-->>EventStore: Persist Success
        
        EventStore->>Projections: Get Events
        Projections->>EventStore: Retrieve Events
        EventStore->>CoreData: Fetch Events
        CoreData-->>EventStore: Return Events
        EventStore-->>Projections: Event Stream
        
        Projections->>Projections: Project Events
        Projections->>State: Update State
        State-->>UI: State Update
        UI-->>User: View Update
    end
```

## Clean Architecture Onion Model

```mermaid
graph TD
    subgraph Onion["Clean Architecture"]
        subgraph CoreLayer["Core Domain"]
            DM["Domain Models"]
            BR["Business Rules"]
        end
        
        subgraph DomainLayer["Domain Layer"]
            ES["Event Sourcing"]
            CH["Command Handling"]
            V["Validation"]
        end
        
        subgraph ApplicationLayer["Application Layer"]
            S["Services"]
            P["Projections"]
            I["Interfaces"]
        end
        
        subgraph InfrastructureLayer["Infrastructure Layer"]
            ESt["Event Store"]
            CD["CoreData"]
            C["Cache"]
        end
        
        subgraph PresentationLayer["Presentation Layer"]
            UI["User Interface"]
            TCA["TCA Components"]
            VM["View Models"]
        end
    end
    
    PresentationLayer --> ApplicationLayer
    ApplicationLayer --> DomainLayer
    DomainLayer --> CoreLayer
    InfrastructureLayer --> ApplicationLayer
    
    classDef coreNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef domainNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef applicationNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    classDef infraNode fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#f57f17;
    classDef presentationNode fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#e65100;
    
    class CoreLayer,DM,BR coreNode;
    class DomainLayer,ES,CH,V domainNode;
    class ApplicationLayer,S,P,I applicationNode;
    class InfrastructureLayer,ESt,CD,C infraNode;
    class PresentationLayer,UI,TCA,VM presentationNode;
```

## Feature Module Architecture

```mermaid
graph TD
    subgraph FM["Feature Module"]
        subgraph UI["UI Components"]
            V["Views"]
            L["Lists"]
            F["Forms"]
        end
        
        subgraph S["State Management"]
            R["Reducer"]
            A["Actions"]
            St["State"]
        end
        
        subgraph DI["Domain Integration"]
            CM["Command Mapper"]
            SM["State Mapper"]
        end
    end
    
    UI <--> S
    S <--> DI
    
    subgraph DL["Domain Layer"]
        CH["Command Handler"]
        SC["State Container"]
    end
    
    DI <--> DL
    
    classDef featureNode fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#e65100;
    classDef stateNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef integrationNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef domainNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    
    class FM,UI,V,L,F featureNode;
    class S,R,A,St stateNode;
    class DI,CM,SM integrationNode;
    class DL,CH,SC domainNode;
```

## Key Architecture Principles

The layered architecture of MonoSphere is guided by these key principles:

1. **Separation of Concerns** - Each layer has a specific, focused responsibility
2. **Dependency Rule** - Dependencies flow inward, with the domain layer at the center
3. **Domain-Driven Design** - Business rules and domain logic are in the domain layer
4. **Event Sourcing** - All state changes are captured as events
5. **Testability** - Each layer can be tested in isolation with mocked dependencies
6. **Functional Core, Imperative Shell** - Core domain logic is functional and pure, with side effects at the edges
7. **Unidirectional Data Flow** - Data flows in one direction, with state updates triggering UI changes