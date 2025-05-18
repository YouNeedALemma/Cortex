# Module Dependencies

This document illustrates the module dependencies in the MonoSphere application, showing how the different modules are organized and how they depend on each other.

## Swift Package Module Structure

```mermaid
graph TD
    A[MonoSphere Package] --> B[App]
    A --> C[Models]
    A --> D[Features]
    A --> E[Common]
    
    C --> C1[Events]
    C --> C2[Validation]
    C --> C3[Storage]
    C --> C4[Errors]
    
    B --> F[SwiftUI]
    B --> G[Combine]
    B --> H[ComposableArchitecture]
    
    D --> F
    D --> H
    D --> C
    
    E --> F
    
    C3 --> I[CoreData]
    
    classDef packageNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef moduleNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef submoduleNode fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#f57f17;
    classDef externalNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    
    class A packageNode;
    class B,C,D,E moduleNode;
    class C1,C2,C3,C4 submoduleNode;
    class F,G,H,I externalNode;
```

## Module Dependencies Diagram

```mermaid
flowchart TD
    subgraph External["External Dependencies"]
        TCA["The Composable Architecture"]
        SwiftUI["SwiftUI"]
        Combine["Combine"]
        CoreData["CoreData"]
    end
    
    subgraph MonoSphere["MonoSphere Package"]
        App["App Module"]
        
        subgraph Models["Models Module"]
            Events["Events Submodule"]
            Storage["Storage Submodule"]
            Validation["Validation Submodule"]
            Errors["Errors Submodule"]
            DomainModels["Domain Models"]
        end
        
        Features["Features Module"]
        Common["Common Module"]
    end
    
    App --> TCA
    App --> SwiftUI
    App --> Models
    App --> Features
    App --> Common
    
    Features --> TCA
    Features --> SwiftUI
    Features --> Models
    Features --> Common
    
    Models --> Combine
    Storage --> CoreData
    Storage --> Events
    Events --> DomainModels
    Validation --> DomainModels
    Validation --> Errors
    
    Common --> SwiftUI
    
    classDef externalNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    classDef packageNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef moduleNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef submoduleNode fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#f57f17;
    
    class TCA,SwiftUI,Combine,CoreData externalNode;
    class MonoSphere packageNode;
    class App,Features,Models,Common moduleNode;
    class Events,Storage,Validation,Errors,DomainModels submoduleNode;
```

## Models Module Decomposition

```mermaid
classDiagram
    class ModelsModule {
        Domain Models
        Event Infrastructure
        Command Handling
        Validation
        Error Types
        Storage
    }
    
    class DomainModels {
        Project
        Task
        ProjectStatus
        TaskPriority
    }
    
    class EventInfrastructure {
        Event
        EventMetadata
        EventStore
        Projections
        EventVersioning
        EventMigration
    }
    
    class CommandHandling {
        CommandHandler
        Commands
        Command Results
    }
    
    class ValidationSystem {
        Validators
        Validation Rules
        Validation Errors
    }
    
    class ErrorTypes {
        DomainError
        ValidationError
        StorageError
    }
    
    class StorageSystem {
        EventStore Implementations
        CoreData Integration
        Serialization
    }
    
    ModelsModule *-- DomainModels
    ModelsModule *-- EventInfrastructure
    ModelsModule *-- CommandHandling
    ModelsModule *-- ValidationSystem
    ModelsModule *-- ErrorTypes
    ModelsModule *-- StorageSystem
    
    CommandHandling --> DomainModels : uses
    CommandHandling --> EventInfrastructure : creates events
    CommandHandling --> ValidationSystem : validates commands
    CommandHandling --> ErrorTypes : returns errors
    
    EventInfrastructure --> StorageSystem : persists events
    EventInfrastructure --> DomainModels : projects to models
    
    ValidationSystem --> ErrorTypes : produces errors
    ValidationSystem --> DomainModels : validates models
```

## Features Module Composition

```mermaid
graph TD
    F[Features Module] --> P[Projects Feature]
    F --> T[Tasks Feature]
    F --> S[Settings Feature]
    F --> J[Journal Feature]
    
    P --> PL[Project List]
    P --> PD[Project Detail]
    P --> PC[Project Creation]
    P --> PE[Project Editing]
    
    T --> TL[Task List]
    T --> TD[Task Detail]
    T --> TC[Task Creation]
    T --> TE[Task Editing]
    
    subgraph TCA["TCA Components"]
        PR[Project Reducer]
        PS[Project State]
        PA[Project Actions]
        
        TR[Task Reducer]
        TS[Task State]
        TA[Task Actions]
    end
    
    P -.-> PR
    P -.-> PS
    P -.-> PA
    
    T -.-> TR
    T -.-> TS
    T -.-> TA
    
    classDef moduleNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef featureNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef viewNode fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#f57f17;
    classDef tcaNode fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#1b5e20;
    
    class F moduleNode;
    class P,T,S,J featureNode;
    class PL,PD,PC,PE,TL,TD,TC,TE viewNode;
    class PR,PS,PA,TR,TS,TA tcaNode;
```

## Common Module Components

```mermaid
graph TD
    C[Common Module] --> UI[UI Components]
    C --> U[Utilities]
    C --> S[Styles]
    
    UI --> B[Button Styles]
    UI --> CB[Card Components]
    UI --> L[List Components]
    UI --> T[Text Field Styles]
    UI --> N[Navigation Components]
    UI --> E[Empty States]
    UI --> LD[Loading Indicators]
    
    U --> F[Formatters]
    U --> V[Validators]
    U --> A[Animations]
    
    S --> C1[Color Schemes]
    S --> T1[Typography]
    S --> L1[Layout Constants]
    S --> Sh[Shadows and Elevations]
    
    classDef moduleNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef categoryNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    classDef componentNode fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#f57f17;
    
    class C moduleNode;
    class UI,U,S categoryNode;
    class B,CB,L,T,N,E,LD,F,V,A,C1,T1,L1,Sh componentNode;
```

## Dependency Flow Between Modules

```mermaid
flowchart LR
    subgraph Layers["Architectural Layers"]
        direction TB
        UI["UI Layer (App, Features)"]
        Domain["Domain Layer (Models)"]
        Storage["Storage Layer (CoreData)"]
    end
    
    UI --> Domain
    Domain --> Storage
    
    subgraph Dependencies["Dependency Flow"]
        direction TB
        AppFeatures["App + Features Modules"]
        ModelsModule["Models Module"]
        ExternalDeps["External Dependencies"]
    end
    
    AppFeatures --> ModelsModule
    AppFeatures --> ExternalDeps
    ModelsModule --> ExternalDeps
    
    classDef layerNode fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#01579b;
    classDef depNode fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#4a148c;
    
    class UI,Domain,Storage layerNode;
    class AppFeatures,ModelsModule,ExternalDeps depNode;
```

## Key Module Responsibilities

| Module | Responsibility | Dependencies |
|--------|----------------|--------------|
| App | Application bootstrap, view composition | SwiftUI, TCA, Models, Features, Common |
| Models | Domain models, event sourcing, command handling | Combine, CoreData |
| Features | Feature-specific views and logic | SwiftUI, TCA, Models, Common |
| Common | Reusable UI components and utilities | SwiftUI |

The module organization follows clean architecture principles with:

1. **Separation of Concerns** - Each module has a clear, focused responsibility
2. **Dependency Direction** - Dependencies flow inward, with domain models at the core
3. **Testability** - Modules can be tested in isolation with mocked dependencies
4. **Reusability** - Common components are extracted to the Common module
5. **Encapsulation** - Implementation details are hidden behind clear interfaces